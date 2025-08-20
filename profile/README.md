# Dream In Code — Voice Assistant for Elderly Care

Proyecto de **asistente de voz** para acompañamiento de adultos mayores. Permite:
- Interacción por voz (“Hey Dream” como wake‑word)
- Conversación natural (STT → LLM → TTS)
- **Recordatorios de medicamentos** con confirmación de toma
- (Opcional) Módulo de **citas médicas** similar a medicamentos
- Integración con **Raspberry Pi** (micrófono + parlante)

> Backend: Flask + OpenAI.  
> Cliente: Python (Porcupine, PyAudio, Pydub) sobre Raspberry Pi.


## Equipo
- **Samuel Solano**
- **Emilio Conejo**
- **Sebastián Arrieta**


## Visión general

El flujo principal está diseñado pensando en la interacción natural del usuario con sus audífonos:  

1. **Activación por wake word**: el adulto mayor pronuncia la palabra clave (ej. “Hey Dream”) y el micrófono Bluetooth despierta al asistente.  
2. **Conversación natural**: el usuario puede hablar libremente, hacer preguntas o pedir ayuda. El flujo es:  
   - audio del usuario → transcripción (**STT**) → respuesta del LLM → síntesis de voz (**TTS**) → respuesta en los audífonos.  
3. **Recordatorios de medicamentos**: la Raspberry Pi consulta periódicamente el endpoint `/reminder_tts` (con `auto=true`) para verificar si corresponde un medicamento en ese momento.  
   - Si es la hora, el sistema genera un **mensaje en audio (WAV/MP3)** que se reproduce en los audífonos, por ejemplo:  
     > “Es hora de tomar la pastilla para la presión.”  
4. **Confirmación verbal**: el asistente escucha la respuesta del usuario:  
   - Si responde “sí”, se marca la toma de medicamento (vía `/confirm_intake`).  
   - Si responde “no” o no hay respuesta, queda registrado como pendiente.  

De esta manera, el dispositivo se convierte en un **compañero conversacional**, que no solo responde preguntas, sino que también **acompaña de forma activa** recordando tareas críticas de salud y solicitando una **confirmación verbal simple** que refuerza la adherencia al tratamiento.  



## Impacto social

El asistente de voz busca **mejorar la calidad de vida de los adultos mayores** al ofrecerles un acompañamiento constante y discreto mediante recordatorios y conversaciones naturales. El dispositivo no pretende sustituir a las personas cuidadoras, sino **potenciar su autonomía** y permitir que los adultos mayores mantengan la mayor libertad posible dentro de su vida cotidiana.  

Entre los beneficios sociales más importantes se destacan:  

- **Autonomía y dignidad**: al recordar medicamentos, citas médicas o actividades, se reduce la dependencia directa de terceros y se refuerza la sensación de independencia.  
- **Confianza para las familias**: los familiares pueden monitorear desde cualquier lugar si la persona está tomando sus medicamentos o atendiendo recordatorios, lo que brinda tranquilidad y reduce la necesidad de supervisión constante.  
- **Reducción de riesgos de salud**: un recordatorio a tiempo puede prevenir olvidos que deriven en complicaciones médicas, mejorando así la adherencia a tratamientos.  
- **Conexión emocional**: aunque es un dispositivo tecnológico, la interacción por voz genera un ambiente más cercano y humano que interfaces tradicionales, evitando la sensación de aislamiento digital.  
- **Inclusión digital**: al estar basado en voz, no requiere conocimientos avanzados de tecnología, lo que permite que personas de mayor edad, incluso con poca experiencia en dispositivos modernos, puedan usarlo sin dificultad.  

En conjunto, el proyecto contribuye a un **ecosistema de cuidado remoto** que busca equilibrar la **independencia del adulto mayor** con la **tranquilidad de sus familiares y cuidadores**, promoviendo un envejecimiento más activo, seguro y conectado.  


## Arquitectura (resumen)

```
[Raspberry Pi]
  ├─ Wake-word Porcupine ("Hey Dream")
  ├─ Captura micrófono (PyAudio) + VAD
  ├─ /voice_mcp → envía audio → recibe TTS (respuesta)
  └─ Poller /reminder_tts?auto=true → reproduce recordatorios

[API Flask (Azure)]
  ├─ /stt, /tts, /voice_mcp
  ├─ /reminder_tts (auto/manual)
  ├─ /meds/all, /meds/due
  └─ /confirm_intake (clasifica sí/no/inesc)
        ↳ (opcional) log_med_intake() a BD
```


## Endpoints (API)

- `GET  /health` → estado simple de la API.
- `POST /voice_mcp` (form-data: `audio`)  
  STT → LLM → TTS. Devuelve audio (`audio/wav` preferido).
- `POST /stt` (form-data: `audio`)  
  Devuelve `{"transcripcion":"..."}`.
- `POST /tts` (JSON: `{"texto": "..."}`)  
  Devuelve audio con el TTS del texto enviado.
- `GET  /meds/due?usuario_id=3&window_min=5&tz_offset_min=-360`  
  Devuelve en JSON los medicamentos que tocan **ahora** (± ventana).
- `GET  /meds/all?usuario_id=3`  
  Devuelve todas las filas de medicamentos para el usuario.
- `POST /reminder_tts`  
  - **Auto**: `{"usuario_id":3, "auto":true, "tz_offset_min":-360}` → verifica la BD y si corresponde devuelve audio del recordatorio.  
  - **Manual**: `{"usuario_id":3, "auto":false, "medicamento":"...", "dosis":"...", "hora":"HH:MM"}`
- `POST /confirm_intake` (voz o texto)  
  Clasifica la respuesta del usuario como **yes/no/unsure** y puede registrar el resultado si existe `log_med_intake()`.

> **Formato de audio**: por defecto se intenta WAV usando `audio.speech` con `extra_body={"format":"wav"}` y se cae a MP3 sólo si no es posible.


## Variables de entorno (API)

- `OPENAI_API_KEY` — clave de OpenAI
- `OPENAI_MODEL` — p.ej. `gpt-4o-mini`
- `OPENAI_STT_MODEL` — `gpt-4o-mini-transcribe`
- `OPENAI_TTS_MODEL` — `gpt-4o-mini-tts`
- `OPENAI_TTS_FORMAT` — `wav` (preferido)
- `OPENAI_VOICE` — voz (p.ej. `alloy`)


## Base de datos (esquema mínimo)

**dbo.Medicamentos** (para recordatorios)
- `MedicamentoID` (PK), `UsuarioID` (FK)
- `NombreMedicamento` (nvarchar)
- `Dosis` (nvarchar), `Instrucciones` (nvarchar)
- `FechaInicio` (date), `FechaHasta` (date)
- `Lunes..Domingo` (bit) — días activos
- `HoraToma` (time)
- `Activo` (bit), `CreatedAt` (datetime2)

> El endpoint `/meds/all` devuelve todos los campos normalizados (bools/ISO/HH:mm).  
> `/reminder_tts?auto=true` usa la hora local (con `tz_offset_min`) y un **window_min** interno para decidir “toca ahora”.


### (Opcional) Citas médicas
Un módulo análogo puede usar la tabla `dbo.CitasMedicas` con campos: `CitaID (PK)`, `UsuarioID`, `Titulo`, `Lugar`, `Fecha`, `Hora`, `Notas`, `Recordar`(bit). Endpoints sugeridos: `/appointments/all`, `/appointment_tts` (auto/manual).


## Raspberry Pi (cliente)

Requisitos del sistema:
- Raspberry Pi con **Raspberry Pi OS**.
- Micrófono USB y parlante (o jack 3.5mm).

Dependencias:
```bash
sudo apt-get update
sudo apt-get install -y python3-venv python3-pip portaudio19-dev ffmpeg mpg123
python3 -m venv venv
source venv/bin/activate
pip install pvporcupine pyaudio pydub requests
```

Archivos relevantes:
- `dream.py` — cliente principal (wake‑word, VAD, envío a API, poller de recordatorios)
- `PrefabAudios/waitResponse.wav` — audio breve que se reproduce **una vez** mientras se espera la respuesta (se normaliza a 16 kHz mono PCM)
- `Wakewords/Hey-Dream_en_raspberry-pi_v3_0_0.ppn` — keyword personalizado de Porcupine

Variables en `dream.py` a revisar:
- `API_BASE_URL`
- `USER_ID`
- Ruta del `.ppn` en `KEYWORD_PATH`
- Dispositivo ALSA (si hace falta, usar `arecord -l` y exportar `PYTHONWARNINGS`/`ALSA_CARD` o abrir `PyAudio` con indices específicos).

Ejecución:
```bash
source venv/bin/activate
python3 dream.py
```
Verás en consola: **“Listening for wake word…”**.  
El hilo de recordatorios consulta `/reminder_tts` cada 30s y reproduce si corresponde.


## Pruebas rápidas (Postman)

- **TTS**  
  POST `${API_BASE_URL}/tts`  
  Body JSON:
  ```json
  {"texto":"Hola, esto es una prueba."}
  ```

- **Recordatorio manual**  
  POST `${API_BASE_URL}/reminder_tts`  
  ```json
  {
    "usuario_id": 3,
    "auto": false,
    "medicamento": "vitamina D",
    "dosis": "1 tableta",
    "hora": "16:53"
  }
  ```

- **Recordatorio automático**  
  POST `${API_BASE_URL}/reminder_tts`  
  ```json
  {
    "usuario_id": 3,
    "auto": true,
    "tz_offset_min": -360
  }
  ```

- **Confirmación de toma (texto)**  
  POST `${API_BASE_URL}/confirm_intake`  
  ```json
  {
    "usuario_id": 3,
    "texto": "sí, ya me la tomé",
    "medicamento": "vitamina D",
    "hora": "16:53",
    "return": "json"
  }
  ```


## Notas de diseño

- El cliente detecta formato real del audio devuelto (WAV/MP3) y reproduce sin “estática” (sniff de cabecera).
- El “wait audio” se reproduce **una sola vez** por petición para evitar loops molestos.
- VAD (energy‑based) para cortar la grabación cuando el usuario deja de hablar.
- Tras una confirmación de recordatorio, el estado del cliente vuelve a `IDLE` (esperar wake‑word).


## Seguridad

- Usa HTTPS siempre (ya está en Azure Websites).  
- Mantén `OPENAI_API_KEY` en **Application Settings** (no en código).  
- Limita CORS o usa tokens si expones paneles.

## Roadmap

Este proyecto tiene un enfoque evolutivo y modular. Las próximas etapas planteadas son:

### 1. Integración con dispositivos inteligentes en el hogar (Smart Home)
- Control básico de luces, ventiladores, aire acondicionado u otros dispositivos mediante comandos de voz.  
- Integración con plataformas estándar como **Home Assistant**, **Google Home** o **Alexa**.  
- Automatización de rutinas para acompañamiento (ej. encender luz nocturna cuando el adulto mayor se levante).

### 2. Notificaciones de peligro o situaciones de riesgo
- Detección de caídas o emergencias mediante sensores IoT o wearables conectados.  
- Alertas inmediatas enviadas a familiares o cuidadores por medio de:
  - Notificaciones móviles (push o SMS).  
  - Llamadas automatizadas en caso de eventos críticos.  
- Integración con cámaras o sensores de movimiento para monitoreo pasivo.

### 3. Conexión con sistemas de salud en Costa Rica
- Implementación de interoperabilidad con los sistemas de la **Caja Costarricense de Seguro Social (CCSS)** y hospitales.  
- Consulta y sincronización de **expedientes digitales en salud (EDUS)** para acceder a:
  - Recetas médicas activas.  
  - Próximas citas programadas.  
  - Indicaciones de médicos tratantes.  
- Generación de reportes automáticos del estado de adherencia a tratamientos para profesionales de salud.  

---


## Licencia

Proyecto interno del equipo — uso académico / demostrativo.
