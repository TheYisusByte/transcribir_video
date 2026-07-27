---
# 📄 Manual de Documentación Técnica
**Desarrollado y automatizado bajo la arquitectura de: YisusByte** 🛠️🚀
---

---

# Manual de Documentación Técnica: Sistema de Transcripción de Video Local

Este documento técnico describe la arquitectura, funcionamiento e instalación del script de transcripción local `transcribir_video.py`. Como solución de ingeniería, este sistema ha sido diseñado para operar de manera autónoma, con alta eficiencia computacional y coste operativo cero tras su despliegue inicial.

---

## 1. ¿Qué hace este proyecto?

El proyecto es una herramienta de consola (CLI) escrita en **Python** que permite transcribir archivos de video de cualquier duración (probado en archivos de más de 1 hora) a formato de texto plano y documentos estructurados de Microsoft Word (`.docx`). 

La característica fundamental del software es que opera **100% de manera local y gratuita**, sin depender de APIs de terceros (como OpenAI API o Google Cloud) y garantizando la privacidad absoluta de los datos.

### Tecnologías, Frameworks y Dependencias Clave

*   **Motor de Inteligencia Artificial (`faster-whisper`):** Es una reimplementación optimizada de la arquitectura de reconocimiento de voz *Whisper* de OpenAI utilizando el motor *CTranslate2*. Logra hasta un **4x de velocidad de procesamiento y menor consumo de memoria** en comparación con la biblioteca oficial de OpenAI.
*   **Manipulación de Formatos Office (`python-docx`):** Utilizada para la creación, formateo dinámico y exportación del informe final de transcripción con estilos aplicados (negritas, saltos de página y títulos).
*   **Procesamiento de Audio (`FFmpeg`):** Herramienta externa invocada a bajo nivel para demultiplexar, decodificar y remuestrear la pista de audio del contenedor de video al formato exacto requerido por el motor de inferencia.
*   **Gestión de Rutas (`pathlib`):** Facilita la portabilidad del código entre sistemas operativos (Windows, macOS, Linux).

---

## 2. Estructura del Proyecto

Al tratarse de una herramienta modular contenida en un archivo único (`transcribir_video.py`), la arquitectura se organiza mediante flujos secuenciales y almacenamiento temporal en el sistema de archivos local.

### Diagrama de Flujo y Relación de Módulos

```text
[ Video Input (.mp4/.mkv) ]
            │
            ▼
┌──────────────────────────┐
│     extraer_audio()      │ ──► Invoca FFmpeg en subprocess
└──────────────────────────┘
            │
            ▼
[ Temp Audio (.wav, 16kHz) ]
            │
            ▼
┌──────────────────────────┐
│      transcribir()       │ ──► Carga faster-whisper (CPU/GPU)
└──────────────────────────┘
            │
            ▼
┌──────────────────────────┐
│    generar_documento()   │ ──► Construye outputs (.docx y .txt)
└──────────────────────────┘
            │
            ├──────────────────────────┐
            ▼                          ▼
 [ Output .docx con marcas ]   [ Output .txt plano ]
            │
            ▼
 (Limpieza: Eliminación del .wav temporal)
```

### Gestión del Espacio de Trabajo (Directorio de Ejecución)
El script no requiere directorios dedicados para su funcionamiento. Utiliza la técnica de **co-localización**, lo que significa que el archivo de audio temporal y los documentos resultantes se generan en la misma ruta donde reside el archivo de video original.

---

## 3. Explicación de Módulos (Paso a Paso)

A continuación se detalla la anatomía de las funciones que componen el script `transcribir_video.py`:

### A. Función `extraer_audio`
```python
def extraer_audio(video_path: Path, audio_path: Path):
```
*   **Rol:** Separar el audio de la pista de video sin comprometer la velocidad del pipeline.
*   **Detalles Técnicos:** Llama a un subproceso de `ffmpeg`. La configuración de argumentos es crítica para el rendimiento de Whisper:
    *   `-vn`: Elimina la transmisión de video para ahorrar tiempo de cómputo.
    *   `-ac 1`: Convierte el audio a canal mono (Whisper no requiere estéreo).
    *   `-ar 16000`: Remuestrea la frecuencia de muestreo a 16 kHz (estándar acústico de Whisper).
    *   `-acodec pcm_s16le`: Codifica en PCM sin comprimir de 16 bits para una lectura ultrarrápida.

### B. Función `transcribir`
```python
def transcribir(audio_path: Path, modelo: str, device: str, idioma: str | None):
```
*   **Rol:** Carga el modelo de Deep Learning seleccionado y procesa el espectrograma del audio para inferir el texto.
*   **Detalles Técnicos:**
    *   **Cuantización:** Selecciona dinámicamente el tipo de cómputo: `float16` para tarjetas gráficas NVIDIA (CUDA) o `int8` (cuantización de enteros de 8 bits) para CPU, lo cual reduce drásticamente el consumo de RAM/VRAM a cambio de una pérdida de precisión imperceptible.
    *   **VAD (Voice Activity Detection):** Utiliza el filtro VAD (`vad_filter=True`) provisto por Silero VAD para ignorar silencios continuos superiores a 500 ms, evitando que el modelo se "atasque" repitiendo bucles de texto.
    *   **Generador:** El modelo retorna un iterador generador (`segments_gen`) que permite procesar el texto bloque a bloque en tiempo real sin tener que esperar a que termine todo el audio.

### C. Función `formatear_timestamp`
```python
def formatear_timestamp(segundos: float) -> str:
```
*   **Rol:** Formateador utilitario de tiempos.
*   **Detalles Técnicos:** Toma un valor de coma flotante que representa segundos y lo convierte en el estándar de marcas temporales legible por humanos `HH:MM:SS` usando aritmética de división entera (`divmod`).

### D. Función `generar_documento`
```python
def generar_documento(segments, salida_docx: Path, salida_txt: Path, nombre_video: str):
```
*   **Rol:** Consolidación de datos procesados.
*   **Detalles Técnicos:**
    *   **Texto Plano (`.txt`):** Se genera mediante una comprensión de lista de los segmentos procesados, minimizando el impacto en memoria al no concatenar strings de forma repetitiva en bucle.
    *   **Estructurado Word (`.docx`):** Crea un objeto `Document` de MS Word. Inserta un bloque del texto íntegro y, posteriormente, un desglose estructurado donde el timestamp se le añade el formato de estilo **negrita** (`run_ts.bold = True`) para una lectura profesional.

### E. Función `main`
```python
def main():
```
*   **Rol:** Orquestador principal e interfaz CLI.
*   **Detalles Técnicos:** Inicializa el módulo `argparse` para la captura de parámetros de terminal. Valida la existencia del archivo de entrada mediante `Path.exists()`, ejecuta de manera coordinada la extracción, transcripción, renderizado de documentos y realiza la tarea higiénica de eliminar el archivo `.wav` temporal (`audio_path.unlink()`) para evitar sobrecargar el disco del usuario.

---

## 4. Conceptos y Glosario Técnico

*   **Whisper (OpenAI):** Red neuronal encoder-decoder de tipo Transformer para el procesamiento de lenguaje natural enfocado a reconocimiento de voz. Fue entrenado con 680,000 horas de audio multilingüe.
*   **faster-whisper (CTranslate2):** Motor de inferencia rápida escrito en C++ que optimiza la carga de capas de atención del Transformer usando técnicas de optimización de gráficos de computación y cuantización de pesos.
*   **Cuantización (INT8 / FP16):** Proceso de conversión de los pesos numéricos del modelo de IA desde floats de precisión simple (32 bits) a enteros (8 bits) o floats de precisión media (16 bits). Reduce el tamaño del modelo a la mitad o a una cuarta parte, acelerando enormemente el cómputo.
*   **VAD (Voice Activity Detection):** Algoritmo que detecta la presencia o ausencia de voz humana en una señal de audio. Evita falsas transcripciones causadas por ruidos de fondo o respiraciones en silencios largos.
*   **Beam Search (Beam Size):** Algoritmo de búsqueda que explora múltiples hipótesis para la siguiente palabra a predecir basándose en probabilidades acumuladas. Un `beam_size=5` (usado aquí) significa que evalúa las 5 mejores secuencias candidatas simultáneamente, garantizando alta fidelidad gramatical.
*   **PCM (Pulse Code Modulation):** Representación digital sin comprimir de una señal analógica de audio. Al ser cruda, es perfecta para que Whisper acceda a la forma de onda de manera directa.

---

## 5. Guía de Instalación y Ejecución

### Prerrequisitos del Sistema

#### 1. Instalar FFmpeg
El script requiere que el binario de FFmpeg esté disponible en las variables de entorno del sistema.

*   **Windows:**
    Abra una terminal de PowerShell como administrador y ejecute:
    ```powershell
    winget install FFmpeg
    ```
    *Nota: Reinicie la consola de comandos tras la instalación.*

*   **macOS (usando Homebrew):**
    ```bash
    brew install ffmpeg
    ```

*   **Linux (Ubuntu/Debian):**
    ```bash
    sudo apt update && sudo apt install ffmpeg -y
    ```

*   **Verificación:** Ejecute `ffmpeg -version` en su consola. Debe retornar información de la versión del software.

### Instalación de Dependencias de Python

Se recomienda crear un entorno virtual para no colisionar con librerías del sistema:

```bash
# Crear entorno virtual
python -m venv venv

# Activar entorno virtual
# En Windows (PowerShell):
.\venv\Scripts\Activate.ps1
# En Linux/macOS:
source venv/bin/activate

# Instalar dependencias requeridas
pip install faster-whisper python-docx
```

---

### Guía de Uso del Script

El script provee una interfaz CLI versátil con opciones adaptadas a diferentes escenarios de hardware.

#### Caso 1: Ejecución Básica (Por defecto en CPU con modelo Medium en Español)
Ideal para computadoras portátiles o de escritorio comunes sin gráfica dedicada.
```bash
python transcribir_video.py "ruta/al/video.mp4"
```

#### Caso 2: Alta Precisión y Aceleración por GPU (NVIDIA CUDA)
Si dispone de una tarjeta de video NVIDIA con soporte CUDA configurado. Se utiliza el modelo más potente disponible (`large-v3`).
```bash
python transcribir_video.py "ruta/al/video.mp4" --modelo large-v3 --device cuda
```

#### Caso 3: Ejecución Ultra Rápida con Recursos Limitados
Para transcripciones instantáneas donde la precisión perfecta no es crítica (ej. notas rápidas).
```bash
python transcribir_video.py "ruta/al/video.mp4" --modelo tiny --idioma es
```

#### Caso 4: Detección Automática de Idioma
Si el video cuenta con un idioma desconocido o alternancia de idiomas, configure el valor `auto` en el parámetro `--idioma`.
```bash
python transcribir_video.py "ruta/al/video.mp4" --idioma auto
```

---

### Resultados del Procesamiento
Tras la finalización exitosa, se podrán ubicar en la carpeta de origen del video dos archivos listos para usar:
1.  `[nombre_video]_transcripcion.docx`: Documento formal de MS Word con títulos y marcas de tiempo detalladas.
2.  `[nombre_video]_transcripcion.txt`: Archivo en texto plano ideal para alimentar LLMs locales, realizar resúmenes o búsquedas textuales de manera rápida.