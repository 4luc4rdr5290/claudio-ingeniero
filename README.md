# Claudio Ingeniero

> Un robot de escritorio que le da cuerpo a tu agente de IA para programar.

Robot pequeño que se mueve, cambia de cara, se ilumina y habla mientras tu agente de IA (Claude Code, Cursor, etc.) trabaja.

Basado en el diseño open hardware [Tiny Engineer](https://github.com/jamro/tiny-engineer) de [@jamro](https://github.com/jamro). Esta versión agrega **audios propios**, **documentación en español** y una corrección para compilar con PlatformIO. No es un producto oficial de Tiny Engineer.

| Tu agente… | El robot… |
| --- | --- |
| Lee archivos | Mira alrededor |
| Escribe código | Teclea |
| Piensa o espera | Mira hacia arriba |
| Termina la tarea | Toca la campana |
| Necesita tu respuesta | Te llama la atención |
| Falla o se cancela | Se queja |

## Cambios respecto al proyecto original

- **Audios personalizados:** se reemplazaron `abort.wav`, `attention.wav`, `dead.wav`, `error.wav` y `welcome.wav` en [`assets/`](assets/).
- **Compilación corregida:** en [`platformio.ini`](platformio.ini) la plataforma ESP32 está fijada a `pioarduino 55.03.311`. Las versiones `55.03.312` y posteriores exigen PlatformIO Core `>=6.2.0` y rechazan la versión `6.2.0.post1`, así que la compilación falla con `IncompatiblePlatform`.
- **Documentación en español:** este README, con el conexionado completo.

## Hardware

| Componente | Para qué |
| --- | --- |
| **Waveshare ESP32-C3-Zero** | Microcontrolador: WiFi, I2C, I2S, USB y LED RGB integrado |
| **PCA9685** | Controlador de los 5 servos |
| **5 servos SG90** | Movimiento de cabeza, brazos y cuerpo |
| **OLED 0,91" (128×32, I2C)** | La cara del robot |
| **MAX98357A** | Amplificador de audio I2S |
| **Parlante de 8 Ω y 1 W** | Voz y sonidos |
| **Fuente USB de 5 V y ≥2 A** | Alimentación |

Lista completa, placa de control y piezas impresas en 3D: [lista de compras](https://github.com/jamro/tiny-engineer/blob/main/docs/shopping.md) · [placa de control principal](https://github.com/jamro/tiny-engineer/blob/main/docs/hardware/main-control-board.md) · [modelos 3D](3d_models/README.md) · [ensamblado](https://github.com/jamro/tiny-engineer/blob/main/docs/3d/assembly.md).

### Pines de la ESP32-C3-Zero

La lógica es de **3,3 V**: no conectes 5 V a ningún GPIO. La definición de los pines está en [`include/pins.h`](include/pins.h).

| GPIO | Función | Va a |
| --- | --- | --- |
| **0** | I2C SDA | PCA9685 y OLED |
| **1** | I2C SCL | PCA9685 y OLED |
| **2** | I2S BCLK | MAX98357A |
| **3** | I2S LRC (WS) | MAX98357A |
| **4** | I2S DIN | MAX98357A |
| **5** | OE del PCA9685 (activo en LOW) | PCA9685 |
| **10** | LED RGB WS2812 | Integrado en la placa |
| 6, 7 | **Libres** | Para módulos extra |
| 8 | Libre, pero no lo pongas en LOW al encender (pin de arranque) | — |
| 20, 21 | UART RX/TX; se pueden usar si la consola va por USB | — |
| 9 | Reservado: botón BOOT | — |
| 18, 19 | Reservados: USB nativo | — |
| 12–17 | No disponibles: los usa la memoria flash interna | — |

### Conexión del MAX98357A

| MAX98357A | ESP32-C3-Zero |
| --- | --- |
| BCLK | GPIO2 |
| LRC | GPIO3 |
| DIN | GPIO4 |
| VIN | 5V |
| GND | GND |
| GAIN | Sin conectar (9 dB). A GND: 12 dB · a VIN: 6 dB · 100 kΩ a GND: 15 dB |
| SD | Sin conectar |

El parlante va a los bornes **+** y **−** del módulo. No conectes ninguna de las dos salidas a GND.

> **¿Querés usar un amplificador analógico (PAM8403, LM386…)?** El ESP32-C3 **no tiene DAC**. Tenés dos opciones:
> - Poner un DAC I2S **PCM5102A** en los mismos pines (GPIO2, 3 y 4) y conectar su salida al amplificador. No hace falta tocar el código.
> - Sacar el audio por **PWM** en GPIO6 o GPIO7, con un filtro RC (1 kΩ + 10 nF) y un condensador de 1 µF en serie. Esto obliga a reescribir la parte de audio.

### LED RGB

Es un **WS2812** integrado en la placa, conectado al **GPIO10**. Usa orden de color **GRB**. En algunas placas los colores salen cambiados; se corrige con el ajuste `rgb_order` desde el asistente de configuración.

| Color | Estado |
| --- | --- |
| Verde tenue | Arrancando |
| Blanco | Trabajando (escribiendo, leyendo, pensando, hablando) |
| Rojo pulsante | Necesita atención, hubo un error o se quedó "sin energía" |
| Rojo fijo | Tarea cancelada |
| Apagado | En reposo o durmiendo |
| Parpadeo rojo y se detiene | Falló el arranque: 1 parpadeo = PCA9685, 2 parpadeos = audio I2S |

## Compilar y subir el firmware

Necesitás [PlatformIO](https://platformio.org/) (CLI o la extensión de VS Code) y un cable USB-C **de datos**.

```bash
git clone <URL-de-este-repositorio>
cd tiny-engineer

# Compila y sube el firmware; después sube los audios automáticamente
pio run -t upload

# Monitor serie
pio device monitor
```

Si tenés varias placas conectadas, indicá el puerto con `--upload-port /dev/cu.usbmodemXXXX` (macOS) o `COMx` (Windows).

Si la subida de los audios falla después de grabar el firmware, armá la imagen primero y subí de nuevo:

```bash
pio run -t buildfs
pio run -t upload
```

Si la placa no entra en modo de carga: mantené apretado **BOOT**, apretá y soltá **RESET**, soltá **BOOT** y reintentá.

## Primer arranque

1. Al encender por primera vez, el robot crea una red WiFi llamada `TinyEngineer-XXXX`.
2. Conectate a esa red y abrí `http://192.168.4.1/config`.
3. Seguí el asistente: calibrá los servos, configurá el LED y cargá tu red WiFi de **2,4 GHz**.
4. Cuando se conecta, la pantalla muestra su IP y suena `welcome.wav`. También responde en `http://tiny-engineer.local`.

## Audios personalizados

Los audios están en [`assets/`](assets/). Para cambiarlos, reemplazá los archivos manteniendo el mismo nombre y este formato:

- **WAV, 44100 Hz, mono, PCM de 16 bits**
- Durante la compilación se convierten a 22050 Hz y se copian a `data/`. No edites `data/`, porque se sobrescribe.

| Archivo | Cuándo suena |
| --- | --- |
| `welcome.wav` | Al conectarse / inicio de sesión |
| `attention.wav` | El agente necesita tu respuesta |
| `error.wav` | Algo falló |
| `abort.wav` | Tarea cancelada |
| `dead.wav` | "Sin energía" |
| `bell.wav` | Tarea terminada (campana) |

En macOS podés convertir un audio así:

```bash
afconvert -f WAVE -d LEI16@44100 -c 1 mi_audio.m4a assets/welcome.wav
```

Después subí todo con `pio run -t upload`.

> Las animaciones están sincronizadas con la duración de los audios originales. Si tu audio es mucho más corto o más largo, el movimiento puede no coincidir; los tiempos se ajustan en [`src/animation/`](src/animation/).

## Conectarlo a tu agente de IA

**Opción fácil** (Claude Code, Cursor, Antigravity). Requiere Node.js:

```bash
npx -y tiny-engineer setup     # asistente interactivo: elegí tu IDE
npx -y tiny-engineer doctor    # verifica la conexión con el robot
```

**Opción manual:** cualquier script que pueda hacer un `POST` HTTP.

```bash
curl -X POST "http://tiny-engineer.local/anim?name=typing"
```

| `name` | Uso típico |
| --- | --- |
| `typing` | El agente escribe código o usa herramientas |
| `reading` | El agente lee archivos |
| `thinking` | El agente piensa o espera |
| `ring` | Terminó el turno |
| `welcome` | Inicio de sesión / saludo |
| `attention` | Necesita tu respuesta |
| `error` / `abort` | Error / cancelación |
| `dead` | "Sin energía" |
| `wakeup` / `sleep` | Despertar / dormir |
| `none` | Reposo |

Pruebas rápidas del hardware:

```bash
curl -X POST http://tiny-engineer.local/test/audio      # audio
curl -X POST http://tiny-engineer.local/test/led        # LED RGB
curl -X POST http://tiny-engineer.local/test/movement   # servos
curl -X POST http://tiny-engineer.local/test/screen     # pantalla
```

Si `tiny-engineer.local` no responde, usá la IP que aparece en la pantalla. Documentación completa (en inglés): [API HTTP](docs/api.md) · [integraciones](docs/integration.md) · [USB serie](docs/serial.md).

## Problemas comunes

| Problema | Solución |
| --- | --- |
| `IncompatiblePlatform` al compilar | Dejá la plataforma fijada en `55.03.311` (ver [Cambios](#cambios-respecto-al-proyecto-original)) |
| No se ve nada en el monitor serie | Verificá que `ARDUINO_USB_CDC_ON_BOOT=1` esté en `platformio.ini` (ya viene así) |
| El LED parpadea en rojo y no arranca | 1 parpadeo: revisá el PCA9685 (I2C en GPIO0/1). 2 parpadeos: revisá el MAX98357A (GPIO2/3/4) |
| No suena nada | Revisá VIN y GND del MAX98357A y el parlante; probá con `POST /test/audio` |
| Colores del LED cambiados | Cambiá `rgb_order` en el asistente de configuración |
| No se conecta al WiFi | La red tiene que ser de 2,4 GHz |

## Licencia

Este proyecto es una versión derivada de [Tiny Engineer](https://github.com/jamro/tiny-engineer) y mantiene sus licencias:

- **Software** (firmware, integraciones, scripts y documentación): [MIT](LICENSE)
- **Diseños de hardware** (CAD, piezas imprimibles y placas KiCad): [CERN-OHL-S-2.0](3d_models/LICENSE)

El nombre y el logo de **Tiny Engineer** no están licenciados; ver [TRADEMARK.md](TRADEMARK.md) y [LICENSING.md](LICENSING.md).
