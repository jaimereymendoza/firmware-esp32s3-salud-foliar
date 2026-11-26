# Firmware Edge Impulse para ESP32-S3-N16R8 con cámara OV3660

Firmware para conectar una placa **ESP32-S3-N16R8** con cámara **OV3660** a **Edge Impulse Studio** como dispositivo de adquisición de datos. Con él se ve la vista previa de la cámara en vivo y se capturan imágenes directamente al proyecto de Edge Impulse.

Forma parte del proyecto *Diagnóstico de salud foliar mediante TinyML en el Edge* (clasificación de hojas sanas y enfermas).

- **Autor:** Jaime Rey Mendoza, Ingeniería en Sistemas Computacionales en Hardware, Facultad de Ingeniería, UACH
- **Basado en:** el firmware oficial [edgeimpulse/firmware-espressif-esp32](https://github.com/edgeimpulse/firmware-espressif-esp32), que está pensado para ESP-EYE y FireBeetle (ESP32 de primera generación). Este repositorio lo adapta al ESP32-S3.

---

## 1. Material necesario

### Hardware

| Elemento | Detalle |
| --- | --- |
| Placa | ESP32-S3-N16R8 (16 MB de flash, 8 MB de PSRAM octal) |
| Cámara | OV3660 (conector tipo ESP32-S3-EYE) |
| Cable | USB-C **de datos** conectado al puerto USB-UART de la placa (chip CH343) |
| Computadora | Probado en Windows |

### Software

| Programa | Versión probada | Para qué sirve |
| --- | --- | --- |
| [ESP-IDF](https://docs.espressif.com/projects/esp-idf/en/v5.3.6/esp32s3/get-started/windows-setup.html) | 5.3.6 | Compilar y grabar el firmware |
| [Node.js](https://nodejs.org/) | LTS | Necesario para la CLI de Edge Impulse |
| [Edge Impulse CLI](https://docs.edgeimpulse.com/docs/tools/edge-impulse-cli/cli-installation) | Última | Conectar la placa con Edge Impulse Studio (`edge-impulse-daemon`) |
| Driver CH343 | — | Solo si Windows no reconoce la placa como puerto COM |
| Cuenta de [Edge Impulse](https://studio.edgeimpulse.com/) | — | Para recibir las imágenes |

Instalación de la CLI de Edge Impulse (después de instalar Node.js), en una terminal:

```bash
npm install -g edge-impulse-cli --force
```

---

## 2. Conexión de la cámara (mapa de pines)

El mapa de pines corresponde al esquema de la ESP32-S3-EYE y ya está definido en el código (`ei_camera.h`, modelo `CAMERA_MODEL_ESP32S3_CUSTOM`). No hay que cambiar nada si la cámara usa estos pines:

| Señal | GPIO | Señal | GPIO |
| --- | --- | --- | --- |
| XCLK | 15 | VSYNC | 6 |
| SIOD (SDA) | 4 | HREF | 7 |
| SIOC (SCL) | 5 | PCLK | 13 |
| D0 (Y2) | 11 | D4 (Y6) | 12 |
| D1 (Y3) | 9 | D5 (Y7) | 18 |
| D2 (Y4) | 8 | D6 (Y8) | 17 |
| D3 (Y5) | 10 | D7 (Y9) | 16 |
| PWDN | Sin usar (-1) | RESET | Sin usar (-1) |

La cámara responde en la dirección I2C (SCCB) **0x3C**.

---

## 3. Compilar y grabar el firmware

La configuración de la placa (particiones, flash de 16 MB, PSRAM octal, etc.) ya viene incluida en los archivos `sdkconfig` y `partitions.csv`, así que **no hace falta abrir `menuconfig`**.

1. Descarga este repositorio: botón verde **Code → Download ZIP** y descomprímelo, o usa `git clone`.
   > Recomendación: ponlo en una ruta corta y sin espacios ni acentos, por ejemplo `C:\esp\firmware-esp32s3`. ESP-IDF falla con rutas largas o con caracteres especiales.
2. Abre la terminal **ESP-IDF 5.3 CMD** (o PowerShell de ESP-IDF), que se instala junto con ESP-IDF.
3. Entra a la carpeta del proyecto:
   ```bash
   cd C:\esp\firmware-esp32s3
   ```
4. Selecciona el chip (solo la primera vez):
   ```bash
   idf.py set-target esp32s3
   ```
   > Si este comando regenera el `sdkconfig` y se pierden los ajustes, revisa la sección 6.
5. Compila:
   ```bash
   idf.py build
   ```
6. Conecta la placa y averigua su puerto en el **Administrador de dispositivos → Puertos (COM y LPT)**. En mi caso fue `COM9`.
7. Graba la placa (cambia `COM9` por tu puerto):
   ```bash
   idf.py -p COM9 flash
   ```

---

## 4. Conectar la placa con Edge Impulse Studio

1. **Cierra cualquier monitor serie** que tengas abierto (si usaste `idf.py monitor`, sal con `Ctrl + ]`). Mientras esté abierto ocupa el puerto y el daemon no puede conectarse.
2. En una terminal normal ejecuta:
   ```bash
   edge-impulse-daemon --clean
   ```
3. Inicia sesión con tu cuenta de Edge Impulse y elige el proyecto donde se guardarán las imágenes.
4. Ponle un nombre al dispositivo cuando lo pida.
5. En Edge Impulse Studio, en **Devices**, la placa debe aparecer con un punto verde.

## 5. Capturar imágenes

1. En Studio entra a **Data acquisition**.
2. En *Device* elige la placa y en *Sensor* elige **Camera (320x240)**.
3. Debe verse la vista previa en vivo de la cámara.
4. Escribe la etiqueta (por ejemplo `sana` o `enferma`) y pulsa **Start sampling**.
5. La imagen aparece en la lista de datos del proyecto.

> La vista previa va algo lenta porque la velocidad del puerto serie está fijada en 115200 baudios (ver sección 7). La captura de imágenes funciona bien.

---

## 6. Ajustes de configuración que se cambiaron

Si por algún motivo el `sdkconfig` se regenera, hay que volver a poner estos valores con `idf.py menuconfig`:

| Parámetro | Valor necesario |
| --- | --- |
| Tabla de particiones | CSV personalizada: `partitions.csv` |
| Tamaño de flash | 16 MB |
| PSRAM | Modo Octal, 80 MHz |
| Nivel de log por defecto | Error |
| Watchdog de tareas al arranque (`ESP_TASK_WDT_INIT`) | Desactivado |
| Pila de la tarea principal (`ESP_MAIN_TASK_STACK_SIZE`) | 12288 |
| Consola secundaria USB-Serial-JTAG | Sin consola secundaria |

Por qué importan: los mensajes de log y del watchdog salen por el mismo puerto serie que usa Edge Impulse, y si aparecen, el daemon recibe datos corruptos (`Timeout when waiting for OK`, `Received snapshot is not a RAW or JPEG image`).

### Cambios en el código respecto al firmware original

| Archivo | Cambio |
| --- | --- |
| `CMakeLists.txt` (raíz) | Activa ESP-NN para ESP32-S3 (`EI_CLASSIFIER_TFLITE_ENABLE_ESP_NN_S3`) y ESP-DSP |
| `edge-impulse-sdk/porting/espressif/ESP-NN/CMakeLists.txt` | Incluye los archivos en ensamblador del S3 |
| `main/CMakeLists.txt` | Rutas relativas y componentes requeridos (cámara, GPIO, UART, ADC, NVS, temporizador) |
| `main/main.cpp` | LED del GPIO 22 al GPIO 2 (el 22 no existe en el S3) |
| `ei_device_espressif_esp32.cpp` | Mismo cambio de LED; `UART_NUM_0` y `portTICK_PERIOD_MS` |
| `ei_device_espressif_esp32.h` | `MAX_BAUD` de 1000000 a 115200 |
| `ei_analogsensor.cpp` | ADC1 configurado de forma explícita (12 bits, canal 0) |
| `ei_camera.h` | Nuevo modelo `CAMERA_MODEL_ESP32S3_CUSTOM` con el mapa de pines |
| `firmware-sdk/ei_image_lib.cpp` | Corrección de un error de memoria: el búfer de imagen se liberaba antes de usarse y la captura fallaba con `got 23` |

---

## 7. Problemas comunes

| Síntoma | Solución |
| --- | --- |
| La placa se reinicia una y otra vez | La configuración de particiones/flash/PSRAM no se aplicó. Revisa la sección 6 |
| El daemon no encuentra la placa | Cierra el monitor serie y revisa que el cable sea de datos y el puerto COM sea el correcto |
| `Timeout when waiting for OK` | Nivel de log o watchdog activados. Revisa la sección 6 |
| La vista previa se congela | Deja `MAX_BAUD` en 115200 en `ei_device_espressif_esp32.h` |
| Aparece `i2c_set_pin(988): scl gpio number error` o "Inertial sensor initialization failed" | **No es un error de la cámara**: el firmware intenta iniciar un acelerómetro que esta placa no tiene. Se puede ignorar |

---

## 8. Licencia

Este proyecto se basa en el firmware de Edge Impulse y conserva su licencia original (ver archivo `LICENSE`).
