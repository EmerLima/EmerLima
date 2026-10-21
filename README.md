# Piano ESP32-C3 — OLED + Botones + Buzzer

Mini-teclado musical embebido: 4 botones disparan notas en un buzzer pasivo
(vía PWM/LEDC) y una pantalla OLED SH1106 de 1.3" muestra qué nota está
sonando. Al arrancar reproduce una melodía de bienvenida a modo de demo.

## Hardware

| Componente              | Notas                                             |
|--------------------------|----------------------------------------------------|
| ESP32-C3 **SuperMini**   | El pinout por defecto está probado con esta placa. Otras placas C3 también sirven, ajustando pines. |
| OLED SH1106 1.3" (I2C)   | 128x64 px, 4 pines (VCC, GND, SDA, SCL)            |
| Buzzer **pasivo**        | Necesario para generar tonos variables (PWM)       |
| 4x pulsadores táctiles 6x6mm | Uno por nota                                   |

## Conexionado (por defecto, ver `include/config.h`)

| Señal         | Pin ESP32-C3 |
|---------------|--------------|
| OLED SDA      | GPIO5        |
| OLED SCL      | GPIO6        |
| Buzzer        | GPIO4        |
| Botón 1 (DO)  | GPIO3        |
| Botón 2 (RE)  | GPIO7        |
| Botón 3 (MI)  | GPIO10       |
| Botón 4 (FA)  | GPIO20       |

Los botones usan `INPUT_PULLUP`: van del pin a GND, sin resistencia externa.
Los pines evitan los *strapping pins* del C3 (GPIO2, GPIO8, GPIO9) y los
pines de USB nativo (GPIO18/19), que en la SuperMini no están expuestos en
el header (GPIO8 tiene el LED azul integrado, GPIO9 el botón BOOT). Si tu
placa usa otro mapeo de pines, sólo hace falta tocar `include/config.h`.

## Arquitectura (sistema embebido: superloop no bloqueante)

```
include/          Headers públicos de cada módulo (drivers + app)
  config.h        Pines y constantes de hardware
  notes.h         Frecuencias de notas musicales
  button.h        Debounce no bloqueante
  buzzer.h        Driver PWM/LEDC de bajo nivel (noteOn/noteOff)
  melody_player.h Reproductor de melodías no bloqueante
  display.h       Driver de la OLED (U8g2)
  melodies.h       Datos de melodías (ej. demo de bienvenida)
src/               Implementación de cada módulo
  main.cpp         setup()/loop(): arma el superloop
```

**Regla de oro:** nada bloquea. Ningún módulo usa `delay()`; todos
guardan su propio estado y deciden qué hacer en cada vuelta de `loop()`
comparando contra `millis()`. Esto es lo que permite, por ejemplo, que los
botones se seguan leyendo con baja latencia mientras suena una nota o una
melodía.

- **`button.h/.cpp`**: máquina de estados de debounce. Expone eventos de
  flanco (`pressedEvent`/`releasedEvent`) que se consumen una sola vez por
  transición.
- **`buzzer.h/.cpp`**: driver mínimo (HAL) sobre el periférico LEDC del
  ESP32. Sólo sabe encender/apagar un tono a una frecuencia; no conoce
  duraciones ni melodías.
- **`melody_player.h/.cpp`**: capa de aplicación construida sobre el driver
  del buzzer. Reproduce un array de `MelodyNote` avanzando nota por nota de
  forma no bloqueante (máquina de estados NOTE/GAP).
- **`display.h/.cpp`**: wrapper fino sobre U8g2 con funciones de alto nivel
  (`displayShowNote`, `displayShowIdle`, etc.) para que `main.cpp` no
  dependa directamente de la librería gráfica.

## Compilar y flashear (PlatformIO)

```bash
pio run                # compilar
pio run -t upload      # compilar y flashear
pio device monitor      # monitor serie
```

## Próximos pasos posibles

- Agregar más notas/octavas si se suman más botones.
- Guardar melodías grabadas por el usuario (secuencia de notas + tiempos)
  en flash (NVS/Preferences) para reproducirlas después.
- Modo "grabación": un botón extra para grabar/reproducir lo tocado.
