---
# 📄 Manual de Documentación Técnica
**Desarrollado y automatizado bajo la arquitectura de: YisusByte** 🛠️🚀
---

## 1. ¿Qué hace este proyecto?

El proyecto es un **sistema local y automatizado de transcripción de video a texto**, diseñado para procesar archivos audiovisuales (incluso de larga duración, superiores a 1 hora) sin depender de servicios en la nube, claves de API (API Keys) ni conexiones activas a Internet tras la descarga inicial del modelo.

### Propósito Principal
Convertir contenido de voz dentro de un archivo de video/audio a formato de texto plano (`.txt`) y documento de Microsoft Word (`.docx`), incluyendo timestamps (marcas de tiempo) para facilitar la auditoría, indexación y consumo de la transcripción.

### Stack Tecnológico y Dependencias
* **Lenguaje de Programación:** Python 3.10+
* **Motor de Inferencia:** `faster-whisper` (Reimplementación optimizada del modelo Whisper de OpenAI basada en CTranslate2).
* **Procesamiento Multimedia:** `ffmpeg` (herramienta CLI externa invocado a través de `subprocess`).
* **Generación de Documentos:** `python-docx` (creación y estilización de archivos `.docx`).
* **Procesamiento de Argumentos:** `argparse` (módulo nativo de Python para la interfaz de línea de comandos).
* **Manejo de Rutas:** `pathlib` (módulo nativo para manipulación agnóstica del sistema operativo).

---

## 2. Estructura del Proyecto y Flujo de Datos

Dado que el proyecto está estructurado como una herramienta CLI monolítica en un único script autocontenido (`transcribir_video.py`), la organización de carpetas e interacción de componentes sigue una arquitectura pipeline secuencial.

### Estructura de Archivos
```text
.
├── transcribir_video.py         # Script principal (CLI, extracción, transcripción y reporte)
├── <video_input>.mp4            # Archivo de entrada provisto por el usuario
├── <video_input>.wav            # Archivo de audio temporal (mono 16kHz PCM)
├── <video_input>_transcripcion.docx # Salida: Documento Word enriquecido con timestamps
└── <video_input>_transcripcion.txt  # Salida: Transcripción continua en texto plano
```

### Arquitectura del Pipeline de Datos (Workflow)

```
[ Archivo de Video ] 
        │
        ▼ (FFmpeg / subprocess)
[ Audio Temporal WAV 16kHz Mono ] 
        │
        ▼ (faster-whisper + VAD Filter)
[ Generador de Segmentos de Texto + Timestamps ]
        │
        ├──► Genera archivo .txt (Texto corrido)
        └──► Genera archivo .docx (Encabezados + Texto continuo + Marcas de tiempo)
        │
        ▼ (Unlink / Cleanup)
[ Eliminación de WAV Temporal ]
```

---

## 3. Explicación de Módulos (Paso a Paso)

El archivo `transcribir_video.py` se divide en funciones con responsabilidades únicas (Principio de Responsabilidad Única - SRP).

### 3.1 `extraer_audio(video_path: Path, audio_path: Path)`
* **Propósito:** Extrae el canal de audio del archivo de video y lo convierte al formato nativo ideal que espera Whisper.
* **Lógica Interna:** Invoca una subclave de `ffmpeg` con los parámetros:
  * `-vn`: Deshabilita la transmisión de video.
  * `-ac 1`: Convierte a canal mono (1 único canal).
  * `-ar 16000`: Muestrea el audio a 16,000 Hz (16 kHz).
  * `-acodec pcm_s16le`: Codifica en PCM lineal de 16 bits poco endian.
* **Manejo de Errores:** Captura la salida mediante `subprocess.run`. Si el código de retorno es diferente de `0`, detiene la ejecución e imprime los últimos 2000 caracteres de `stderr`.

### 3.2 `transcribir(audio_path: Path, modelo: str, device: str, idioma: str | None)`
* **Propósito:** Ejecuta la inferencia del modelo de lenguaje en el archivo de audio.
* **Lógica Interna:**
  1. Selecciona el tipo de cómputo (`compute_type`): `float16` si se ejecuta en una GPU NVIDIA (`cuda`), o `int8` (cuantización de 8 bits) si corre en CPU para optimizar rendimiento y consumo de RAM.
  2. Instancia `WhisperModel`.
  3. Aplica un filtro **VAD (Voice Activity Detection)** mediante `vad_filter=True` con un umbral de silencio mínimo (`min_silence_duration_ms=500`). Esto previene alucinaciones del modelo y aceleración en segmentos vacíos.
  4. Recorre el generador de segmentos (`segments_gen`), imprimiendo la progresión en tiempo real en la consola con formato `[HH:MM:SS]`.

### 3.3 `formatear_timestamp(segundos: float) -> str`
* **Propósito:** Convierte valores numéricos de tiempo flotante (en segundos) a una cadena de texto estructurada en tiempo legible (`HH:MM:SS`).
* **Lógica Interna:** Utiliza la función nativa `divmod` para descomponer recursivamente los segundos totales en horas, minutos y segundos.

### 3.4 `generar_documento(segments, salida_docx: Path, salida_txt: Path, nombre_video: str)`
* **Propósito:** Transforma los objetos de segmento retornados por el modelo en archivos utilizables por el usuario.
* **Lógica Interna:**
  * **Exportación TXT:** Une todo el texto de los segmentos limpiando espacios en blanco (`strip()`) y los guarda con codificación UTF-8.
  * **Exportación DOCX:** Inicializa un documento `python-docx`, añade título principal, sección de texto corrido y una segunda página con la lista detallada de marcas de tiempo e intervenciones en negrita.

### 3.5 `main()`
* **Propósito:** Punto de entrada del programa (Entry Point).
* **Lógica Interna:** Configura el parser de argumentos de la línea de comandos (`argparse`), valida la existencia del archivo de entrada, resuelve las rutas absolutas, coordina la ejecución secuencial del pipeline y asegura la eliminación del archivo `.wav` intermedio como tarea de limpieza final.

---

## 4. Conceptos y Glosario Técnico

* **faster-whisper / CTranslate2:** `CTranslate2` es un motor de inferencia optimizado para modelos de Transformer. `faster-whisper` reimplementa la arquitectura Whisper de OpenAI usando CTranslate2, logrando ejecuciones de hasta 4x más rápidas con menor uso de memoria mediante técnicas de cuantización.
* **VAD (Voice Activity Detection):** Algoritmo que detecta la presencia o ausencia de voz humana en una señal de audio. Al descartar los silencios antes de pasarlos al Transformer, se evitan bucles infinitos de repetición (alucinaciones) y se reduce sustancialmente el tiempo total de procesamiento.
* **Cuantización (Quantization - `int8` vs `float16`):** Técnica de compresión del modelo que convierte los pesos de coma flotante de alta precisión (`float32` o `float16`) a enteros de 8 bits (`int8`). Permite ejecutar modelos complejos en CPUs convencionales sin degradar sensiblemente la precisión del reconocimiento de voz.
* **Beam Search (Tamaño 5):** Algoritmo de búsqueda heurística que explora los mejores caminos en el árbol de probabilidades al momento de generar palabras. Un `beam_size=5` evalúa las 5 secuencias de texto más probables en paralelo para reducir errores de transcripción.
* **PCM (Pulse-Code Modulation) 16kHz Mono:** El formato estándar de representación digital de señales de audio sin compresión codificado a 16,000 muestras por segundo. Es la resolución requerida por los espectrogramas de entrada del modelo Whisper.

---

## 5. Guía de Instalación y Ejecución

### Prerrequisitos
1. **Python 3.10 o superior** instalado.
2. **FFmpeg** instalado en el sistema operativo y agregado a la variable de entorno PATH.
   * **Windows:** Instalar mediante Winget ejecutando `winget install ffmpeg` en PowerShell.
   * **Linux (Ubuntu/Debian):** `sudo apt update && sudo apt install ffmpeg`
   * **macOS:** `brew install ffmpeg`
   * *Verificación:* Ejecutar `ffmpeg -version` en la terminal.

### Paso 1: Clonar o descargar el código
Coloca el archivo `transcribir_video.py` en la carpeta de tu preferencia.

### Paso 2: Crear un entorno virtual (Recomendado)
```bash
python -m venv venv
# En Windows:
.\venv\Scripts\activate
# En Linux/macOS:
source venv/bin/activate
```

### Paso 3: Instalar dependencias de Python
```bash
pip install faster-whisper python-docx tqdm
```

### Paso 4: Comandos de Ejecución

#### Uso Básico (CPU - Modelo Medium)
```bash
python transcribir_video.py "C:\Ruta\A\Tu\video.mp4"
```

#### Uso Optimizado para GPU NVIDIA (requiere drivers CUDA)
```bash
python transcribir_video.py "video.mp4" --modelo large-v3 --device cuda
```

#### Especificar idioma manualmente o usar otro modelo
```bash
python transcribir_video.py "conferencia.mp4" --modelo small --idioma en
```

#### Opciones de la Línea de Comandos (CLI Flags)
| Flag | Valores Posibles | Valor por Defecto | Descripción |
| :--- | :--- | :--- | :--- |
| `video` | Ruta de archivo (string) | **Requerido** | Ruta al archivo de video o audio. |
| `--modelo` | `tiny`, `base`, `small`, `medium`, `large-v3` | `medium` | Tamaño del modelo (precisión vs velocidad). |
| `--device` | `cpu`, `cuda` | `cpu` | Dispositivo de procesamiento hardware. |
| `--idioma` | Código ISO (ej. `es`, `en`, `fr`) o `auto` | `es` | Idioma del audio. `auto` activa autodetección. |
