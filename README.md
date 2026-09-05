# hoverboard-firmware-hack-FOC

## Guía de este fork: Optimus-NoPrime

Este fork conserva el firmware FOC de [EFeru](https://github.com/EFeru/hoverboard-firmware-hack-FOC).
Integra aquí una guía de montaje, programación y diagnóstico inspirada en el
[README de lucysrausch/hoverboard-firmware-hack](https://github.com/lucysrausch/hoverboard-firmware-hack/blob/master/README.md),
con atribución a sus autores. **La integración es documental: no mezcla los dos firmwares.**
Las secciones originales de FOC se mantienen más abajo.

### Papel dentro del robot

Cada placa de hoverboard controla dos motores de tracción. El robot utiliza dos
placas para cuatro ruedas. Un micro independiente traduce CAN a UART; este fork
no implementa ese puente. Los cuatro motores de dirección usan sus propios Nano,
L298N y AS5048B y no se controlan desde este firmware.

### Índice de la guía integrada

- [Build / compilación](#build--compilación)
- [Hardware](#hardware-del-hoverboard)
- [Configuración UART para Optimus-NoPrime](#configuración-uart-para-optimus-noprime)
- [Flashing / programación](#flashing--programación)
- [Desbloqueo del microcontrolador](#desbloqueo-del-microcontrolador)
- [Troubleshooting / diagnóstico](#troubleshooting--diagnóstico)
- [Ejemplos y proyectos relacionados](#ejemplos-y-proyectos-relacionados)

### Build / compilación

Este fork ofrece dos formas de compilar. Para Optimus-NoPrime se recomienda
PlatformIO porque fija la placa, el framework, las opciones de enlace y la variante
en un único comando reproducible.

#### Opción recomendada: PlatformIO

Instala [PlatformIO Core](https://docs.platformio.org/en/latest/core/installation/index.html)
o la extensión de PlatformIO para Visual Studio Code. Desde la raíz del firmware:

```sh
pio run -e VARIANT_USART
```

El resultado se genera dentro de `.pio/build/VARIANT_USART/`. Para compilar otra
variante, sustituye `VARIANT_USART` por uno de los entornos declarados en
[platformio.ini](platformio.ini), por ejemplo `VARIANT_ADC`, `VARIANT_PWM` o
`VARIANT_NUNCHUK`.

Antes de compilar, revisa en [Inc/config.h](Inc/config.h):

- `CTRL_TYP_SEL`: conmutación, sinusoidal o FOC.
- `CTRL_MOD_REQ`: tensión, velocidad o par, según el tipo de control.
- Límites de corriente, velocidad, tensión y temperatura.
- Entrada seleccionada y UART utilizada. Dos funciones incompatibles no pueden
  compartir el mismo conector.

#### Opción heredada: Makefile y GNU Arm Embedded

El flujo procedente del proyecto original también está disponible:

```sh
make
```

Necesita GNU Make y el toolchain `arm-none-eabi-gcc`. La variable `PREFIX` del
[Makefile](Makefile) debe apuntar al prefijo correcto si las herramientas no están
en `PATH`. El binario se genera como `build/hover.bin`. Este método toma la variante
definida en `Inc/config.h`; no se debe definir una variante distinta a la vez desde
varios sitios. Para limpiar sus artefactos:

```sh
make clean
```

Las instrucciones históricas citaban GCC Arm Embedded 7. Hoy conviene usar primero
la versión que valida el CI de este fork o la indicada por la documentación FOC,
porque distintas versiones del compilador pueden cambiar el resultado.

### Hardware del hoverboard

![Pinout de la placa principal](docs/pictures/mainboard_pinout.png)

La placa principal habitual incorpora un STM32F103RCT6; algunas revisiones usan un
GD32F103RCT6. Antes de programar, confirma el micro y que la placa aparece en la
[tabla de compatibilidad FOC](https://github.com/EFeru/hoverboard-firmware-hack-FOC/wiki/Firmware-Compatibility).
Las placas partidas o basadas en AT32 requieren proyectos o procedimientos distintos.

Las dos tomas de cuatro hilos que originalmente iban a las placas laterales exponen
masa, alimentación de aproximadamente 12/15 V y las señales de USART2 o USART3.
Según la configuración del firmware, esas señales pueden utilizarse para UART,
PWM, PPM, iBUS, ADC o I2C. No todas las funciones pueden coexistir en los mismos pines.

Puntos eléctricos importantes:

- USART3, en el cable derecho/corto, admite señales lógicas de 5 V y es la opción
  recomendada cuando el emisor es un Arduino de 5 V.
- USART2, en el cable izquierdo/largo, no admite 5 V. Utiliza lógica de 3,3 V o
  adaptación de nivel.
- El hilo de alimentación de 12/15 V no es una salida lógica ni debe conectarse
  directamente a un microcontrolador.
- Todas las señales necesitan una masa común. Alimenta cada circuito con un
  regulador adecuado y comprueba su esquema antes de unirlo a la placa.
- El rango de las entradas ADC es 0–3,3 V. La adaptación descrita para GameTrak en
  el proyecto original solo corresponde a esa variante y no debe aplicarse al puente UART.

El esquema reconstruido está incluido en
[docs/20150722_hoverboard_sch.pdf](docs/20150722_hoverboard_sch.pdf). Cerca del
microcontrolador hay pads de depuración con GND, 3V3, SWDIO y SWCLK. El pin 3V3 sirve
como referencia de nivel; no debe usarse para alimentar la placa desde el ST-Link.

### Configuración UART para Optimus-NoPrime

La variante `VARIANT_USART` incluida actualmente habilita control y feedback por
USART2 a 115200 bit/s. Como USART2 no tolera 5 V, el futuro puente CAN/UART debe
trabajar a 3,3 V o incorporar adaptación de nivel. Si se decide emplear USART3,
hay que cambiar de forma coherente las macros `CONTROL_SERIAL_USARTx` y
`FEEDBACK_SERIAL_USARTx` en [Inc/config.h](Inc/config.h).

El protocolo binario de referencia está en
[Arduino/hoverserial/hoverserial.ino](Arduino/hoverserial/hoverserial.ino). Utiliza
una palabra inicial `0xABCD`, dos consignas de 16 bits y checksum; el feedback tiene
su propia estructura y checksum. El puente CAN debe serializar exactamente esas
estructuras y aplicar timeout. Enviar números como texto por UART no funciona.

No habilites `DEBUG_SERIAL_USART2` sobre la misma UART que `CONTROL_SERIAL_USART2`
y `FEEDBACK_SERIAL_USART2`: `config.h` trata esas combinaciones como incompatibles.

### Flashing / programación

#### Conexión del ST-Link

Con la potencia de los motores desconectada y después de identificar los pads,
conecta únicamente:

| ST-Link | Placa hoverboard |
|---|---|
| GND | GND |
| SWDIO | SWDIO |
| SWCLK | SWCLK |
| RESET | RESET, opcional según placa y herramienta |

**No conectes el 3V3 del programador para alimentar la placa.** La placa debe usar
su propia alimentación. El proyecto original indica que muchas placas necesitan
mantener pulsado el botón de encendido o puentear temporalmente sus contactos para
que el circuito de enclavamiento no corte la alimentación durante el flasheo.
El voltaje necesario depende de la placa y de la batería: verifica el esquema y la
documentación FOC antes de aplicar la afirmación histórica de “más de 36 V”.

#### Flashear con PlatformIO

Con un ST-Link reconocido por el sistema:

```sh
pio run -e VARIANT_USART -t upload
```

PlatformIO compila y escribe usando `upload_protocol = stlink`. Revisa primero el
entorno seleccionado: este comando modifica la memoria flash del controlador conectado.

#### Flashear el binario generado por Make

Con la utilidad de [stlink](https://github.com/stlink-org/stlink):

```sh
st-flash --reset write build/hover.bin 0x08000000
```

O con OpenOCD y un ST-Link V2:

```sh
openocd -f interface/stlink-v2.cfg -f target/stm32f1x.cfg \
  -c init -c "reset halt" \
  -c "flash write_image erase build/hover.bin 0x08000000" \
  -c "reset run" -c shutdown
```

El nombre del fichero y la interfaz pueden variar según sistema, versión del ST-Link
y método de compilación. Comprueba siempre que el binario corresponde a la variante
y placa conectadas.

### Desbloqueo del microcontrolador

Una placa que nunca se ha reprogramado puede tener protección de lectura activa.
Compruébalo primero con la herramienta del ST-Link. El procedimiento estándar
documentado por el proyecto original para STM32F1 es:

```sh
openocd -f interface/stlink-v2.cfg -f target/stm32f1x.cfg \
  -c init -c "reset halt" -c "stm32f1x unlock 0" -c shutdown
```

Desbloquear normalmente provoca un borrado masivo: se pierde el firmware de fábrica
y cualquier calibración almacenada. Los procedimientos alternativos que escriben
directamente registros flash son específicos del STM32F1 y no deben ejecutarse en
un GD32, AT32 o una revisión desconocida. Si el desbloqueo estándar falla, consulta
primero [How to Unlock MCU Flash](https://github.com/EFeru/hoverboard-firmware-hack-FOC/wiki/How-to-Unlock-MCU-Flash)
y la [secuencia histórica completa](https://github.com/lucysrausch/hoverboard-firmware-hack/blob/master/README.md#flashing).

En Windows también se puede usar STM32 ST-LINK Utility o STM32CubeProgrammer. La
opción equivalente suele aparecer como eliminación de Read Out Protection; revisa
el dispositivo detectado antes de aceptarla.

### Troubleshooting / diagnóstico

#### El programador no detecta el micro

- Comprueba masa común, SWDIO/SWCLK, continuidad y que no estén intercambiados.
- Mantén alimentada y encendida la placa durante toda la operación.
- Reduce la frecuencia SWD si el cable es largo o el contacto es deficiente.
- Verifica que el target y el fichero de OpenOCD coinciden con el micro real.
- Si hay protección de lectura, sigue el apartado anterior y asume que se borrará.

#### Compila, flashea, pero la placa no arranca

- Confirma que se compiló la variante deseada y revisa `Inc/config.h`.
- Comprueba el botón/circuito de power latch; un apagado inmediato puede parecer
  un fallo de firmware.
- Interpreta los pitidos y errores con la
  [página de diagnóstico FOC](https://github.com/EFeru/hoverboard-firmware-hack-FOC/wiki/Diagnostics).
- Usa la salida debug solo en una UART que no esté dedicada al control o feedback.

#### El motor vibra, hace ruido o no gira suavemente

Revisa las tres fases y los sensores Hall. Unir colores iguales suele funcionar,
pero no garantiza el orden eléctrico en todas las marcas. Una combinación incorrecta
puede causar tirones, corriente elevada o giro deficiente. No pruebes combinaciones
con la rueda cargada: limita corriente y velocidad y valida la correspondencia según
la guía FOC antes de aumentar consignas.

#### UART sin comandos o feedback

- Confirma 115200, formato binario, byte order, palabra inicial y checksum.
- Cruza TX con RX y comparte GND.
- Comprueba que el puerto activado en `config.h` coincide con el cable físico.
- Respeta los niveles: USART2 a 3,3 V; USART3 es la alternativa tolerante a 5 V.
- No mezcles debug con control/feedback sobre la misma UART.

#### Fallos intermitentes al acelerar

El cableado del motor genera interferencias que afectan especialmente a señales
largas, I2C y PPM. Mantén cables de señal cortos y separados de las fases, usa pares
con masa o cable apantallado cuando proceda, añade ferritas y desacoplo, y verifica
pull-ups I2C. Errores de entrada pueden provocar cambios bruscos de consigna y hacer
actuar la protección de la batería.

Para variantes analógicas, el proyecto original recomienda pull-down próximos a
las entradas para que un cable desconectado no quede flotante. El valor y la conexión
deben calcularse para el circuito concreto; la referencia histórica era de unos
100 kΩ hacia masa con potenciómetros alimentados a 3,3 V.

### Ejemplos y proyectos relacionados

- [Charla “Howto: Moving Objects”](https://media.ccc.de/v/gpn18-95-howto-moving-objects):
  introducción al reaprovechamiento de hardware de hoverboard.
- [Guía de construcción TranspOtter](https://github.com/lucysrausch/hoverboard-firmware-hack/wiki/Build-Instruction:-TranspOtter):
  chasis, electrónica, montaje, toolchain y programación del proyecto original.
- [Ejemplo UART incluido](Arduino/hoverserial/hoverserial.ino): estructuras de
  comandos y feedback compatibles con este firmware FOC.
- [Control UART bidireccional](https://github.com/RoboDurden/hoverboard-firmware-hack):
  otra referencia histórica con ejemplo Arduino.
- [Firmware para placas AT32F403RCT6](https://github.com/cloidnerux/hoverboard-firmware-hack).
- [Firmware para placas partidas](https://github.com/flo199213/Hoverboard-Firmware-Hack-Gen2).
- [Placas de interconexión](https://github.com/Jana-Marie/hoverboard-breakout).
- [Silla de ruedas](https://github.com/Lahorde/steer_speed_ctrl),
  [TranspOtterNG](https://github.com/Jan--Henrik/transpOtterNG) y
  [BiPropellant](https://github.com/bipropellant): proyectos derivados citados
  por las comunidades original y FOC.

### Fuentes y diferencias entre proyectos

- [README original y proyectos de referencia](https://github.com/lucysrausch/hoverboard-firmware-hack):
  contexto de reutilización, SWD y diagnóstico de hardware.
- [Documentación FOC](https://github.com/EFeru/hoverboard-firmware-hack-FOC/wiki):
  autoridad para variantes, placas compatibles y control de este fork.
- No se trasladan como reglas universales los umbrales de tensión/corriente ni
  las secuencias de escritura directa de registros del README antiguo: dependen
  del hardware y de la herramienta. Tampoco se sustituyen los modos FOC por los
  modos disponibles en el firmware original.

---

## Documentación original de FOC

[![Build status](https://github.com/EFeru/hoverboard-firmware-hack-FOC/actions/workflows/build_on_commit.yml/badge.svg)](https://github.com/EFeru/hoverboard-firmware-hack-FOC/actions/workflows/build_on_commit.yml)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![paypal](https://www.paypalobjects.com/en_US/i/btn/btn_donate_SM.gif)](https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=CU2SWN2XV9SCY&currency_code=EUR&source=url)

This repository implements Field Oriented Control (FOC) for stock hoverboards. Compared to the commutation method, this new FOC control method offers superior performance featuring:
 - reduced noise and vibrations 	
 - smooth torque output and improved motor efficiency. Thus, lower energy consumption
 - field weakening to increase maximum speed range

Table of Contents
=======================

* **Wiki:** please check the wiki pages for [Getting Started](https://github.com/EFeru/hoverboard-firmware-hack-FOC/wiki#getting-started) and for [Troubleshooting](https://github.com/EFeru/hoverboard-firmware-hack-FOC/wiki#troubleshooting)
* [Hardware](#hardware)
* [FOC Firmware](#foc-firmware)
* [Example Variants](#example-variants)
* [Projects and Links](#projects-and-links)
* [Contributions](#contributions)

#### The hoverboards with mainboards also come with 2 sideboards(not [splitboards](https://github.com/EFeru/hoverboard-firmware-hack-FOC/wiki/Firmware-Compatibility#split-boards)), check the following [wiki](https://github.com/EFeru/hoverboard-firmware-hack-FOC/wiki/Sideboards) about this firmware

#### For the FOC controller design, see the following repository:
 - [bldc-motor-control-FOC](https://github.com/EFeru/bldc-motor-control-FOC)

#### Videos:
<table>
  <tr>
    <td><a href="https://youtu.be/IgHCcj0NgWQ" title="Hovercar" rel="noopener"><img src="/docs/pictures/videos_preview/hovercar_intro.png"></a></td>
    <td><a href="https://youtu.be/gtyqtc37r10" title="Cruise Control functionality" rel="noopener"><img src="/docs/pictures/videos_preview/cruise_control.png"></a></td>
    <td><a href="https://youtu.be/jadD0M1VBoc" title="Hovercar pedal functionality" rel="noopener"><img src="/docs/pictures/videos_preview/hovercar_pedals.png"></a></td>
  </tr>
  <tr>
    <td><a href="https://youtu.be/UnlbMrCkjnE" title="Commutation vs. FOC (constant speed)" rel="noopener"><img src="/docs/pictures/videos_preview/com_foc_const.png"></a></td> 
    <td><a href="https://youtu.be/V-_L2w10wZk" title="Commutation vs. FOC (variable speed)" rel="noopener"><img src="/docs/pictures/videos_preview/com_foc_var.png"></a></td>       
    <td><a href="https://youtu.be/tVj_lpsRirA" title="Reliable Serial Communication" rel="noopener"><img src="/docs/pictures/videos_preview/serial_com.png"></a></td>
  </tr>
</table>


---
## Hardware
 
![mainboard_pinout](/docs/pictures/mainboard_pinout.png)

The original Hardware supports two 4-pin cables that originally were connected to the two sideboards. They break out GND, 12/15V and USART2&3 of the Hoverboard mainboard. Both USART2&3 support UART, PWM, PPM, and iBUS input. Additionally, the USART2 can be used as 12bit ADC, while USART3 can be used for I2C. Note that while USART3 (right sideboard cable) is 5V tolerant, USART2 (left sideboard cable) is **not** 5V tolerant.

Typically, the mainboard brain is an [STM32F103RCT6](/docs/literature/[10]_STM32F103xC_datasheet.pdf), however some mainboards feature a [GD32F103RCT6](/docs/literature/[11]_GD32F103xx-Datasheet-Rev-2.7.pdf) which is also supported by this firmware.

For the reverse-engineered schematics of the mainboard, see [20150722_hoverboard_sch.pdf](/docs/20150722_hoverboard_sch.pdf)

 
---
## FOC Firmware
 
In this firmware 3 control types are available, it can be set in config.h file via CTRL_TYP_SEL parameter:
- Commutation (COM_CTRL)
- Sinusoidal (SIN_CTRL)
- Field Oriented Control (FOC_CTRL) with the following 3 control modes that can be set in config.h file with parameter CTRL_MOD_REQ:
  - **VOLTAGE MODE(VLT_MODE)**: in this mode the controller applies a constant Voltage to the motors. Recommended for robotics applications or applications where a fast motor response is required.
  - **SPEED MODE(SPD_MODE)**: in this mode a closed-loop controller realizes the input speed RPM target by rejecting any of the disturbance (resistive load) applied to the motor. Recommended for robotics applications or constant speed applications.
  - **TORQUE MODE(TRQ_MODE)**: in this mode the input torque target is realized. This mode enables motor "freewheeling" when the torque target is `0`. Recommended for most applications with a sitting human driver.

#### Comparison between different control methods

|Control method| Complexity | Efficiency | Smoothness | Field Weakening | Freewheeling | Standstill hold |
|--|--|--|--|--|--|--|
|Commutation| - | - | ++ | n.a. | n.a. | + |
|Sinusoidal| + | ++ | ++ | +++ | n.a. | + |
|FOC VOLTAGE| ++ | +++ | ++ | ++ | n.a. | +<sup>(2)</sup> |
|FOC SPEED| +++ | +++ | + | ++ | n.a. | +++ |
|FOC TORQUE| +++ | +++ | +++ | ++ | +++<sup>(1)</sup> | n.a<sup>(2)</sup> |

<sup>(1)</sup> By enabling `ELECTRIC_BRAKE_ENABLE` in `config.h`, the freewheeling amount can be adjusted using the `ELECTRIC_BRAKE_MAX` parameter.<br/>
<sup>(2)</sup> The standstill hold functionality can be forced by enabling `STANDSTILL_HOLD_ENABLE` in `config.h`. 

In all FOC control modes, the controller features maximum motor speed and maximum motor current protection. This brings great advantages to fulfil the needs of many robotic applications while maintaining safe operation.


### Field Weakening / Phase Advance

 - By default the Field weakening is disabled. You can enable it in config.h file by setting the FIELD_WEAK_ENA = 1 
 - The Field Weakening is a linear interpolation from 0 to FIELD_WEAK_MAX or PHASE_ADV_MAX (depeding if FOC or SIN is selected, respectively)
 - The Field Weakening starts engaging at FIELD_WEAK_LO and reaches the maximum value at FIELD_WEAK_HI
 - The figure below shows different possible calibrations for Field Weakening / Phase Advance
 ![Field Weakening](/docs/pictures/FieldWeakening.png)
 
 ⚠️ If you re-calibrate the Field Weakening please take all the safety measures! The motors can spin very fast!
 Power consumption will be highly increase and you can trigger the overvoltage protection of your BMS ⚠️


### Parameters
 - All the calibratable motor parameters can be found in the 'BLDC_controller_data.c'. I provided you with an already calibrated controller, but if you feel like fine tuning it feel free to do so 
 - The parameters are represented in Fixed-point data type for a more efficient code execution
 - For calibrating the fixed-point parameters use the [Fixed-Point Viewer](https://github.com/EFeru/FixedPointViewer) tool
 - The controller parameters are given in [this table](https://github.com/EFeru/bldc-motor-control-FOC/blob/master/02_Figures/paramTable.png)


### FOC Webview

To explore the controller without a Matlab/Simulink installation click on the link below:

[https://eferu.github.io/bldc-motor-control-FOC/](https://eferu.github.io/bldc-motor-control-FOC/)

---
## Example Variants

- **VARIANT_ADC**: The motors are controlled by two potentiometers connected to the Left sensor cable (long wired)
- **VARIANT_USART**: The motors are controlled via serial protocol (e.g. on USART3 right sensor cable, the short wired cable). The commands can be sent from an Arduino. Check out the [hoverserial.ino](/Arduino/hoverserial) as an example sketch.
- **VARIANT_NUNCHUK**: Wii Nunchuk offers one hand control for throttle, braking and steering. This was one of the first input device used for electric armchairs or bottle crates.
- **VARIANT_PPM**: RC remote control with PPM Sum signal.
- **VARIANT_PWM**: RC remote control with PWM signal.
- **VARIANT_IBUS**: RC remote control with Flysky iBUS protocol connected to the Left sensor cable.
- **VARIANT_HOVERCAR**: The motors are controlled by two pedals brake and throttle. Reverse is engaged by double tapping on the brake pedal at standstill. See [HOVERCAR wiki](https://github.com/EFeru/hoverboard-firmware-hack-FOC/wiki/Variant-HOVERCAR).
- **VARIANT_HOVERBOARD**: The mainboard reads the two sideboards data. The sideboards need to be flashed with the hacked version. The balancing controller is **not** yet implemented.
- **VARIANT_TRANSPOTTER**: This is for transpotter build, which is a hoverboard based transportation system. For more details on how to build it check [here](https://github.com/NiklasFauth/hoverboard-firmware-hack/wiki/Build-Instruction:-TranspOtter) and [here](https://hackaday.io/project/161891-transpotter-ng).
- **VARIANT_SKATEBOARD**: This is for skateboard build, controlled using an RC remote with PWM signal connected to the right sensor cable.

Of course the firmware can be further customized for other needs or projects.


---
## Projects and Links

- **Original firmware:** [https://github.com/lucysrausch/hoverboard-firmware-hack](https://github.com/lucysrausch/hoverboard-firmware-hack)
- **[Candas](https://github.com/Candas1/) Hoverboard Web Serial Control:** [https://github.com/Candas1/Hoverboard-Web-Serial-Control](https://github.com/Candas1/Hoverboard-Web-Serial-Control)
- **[RoboDurden's](https://github.com/RoboDurden) online compiler:** [https://pionierland.de/hoverhack/](https://pionierland.de/hoverhack/) 
- **Hoverboard hack for AT32F403RCT6 mainboards:** [https://github.com/cloidnerux/hoverboard-firmware-hack](https://github.com/cloidnerux/hoverboard-firmware-hack)
- **Hoverboard hack for split mainboards:** [https://github.com/flo199213/Hoverboard-Firmware-Hack-Gen2](https://github.com/flo199213/Hoverboard-Firmware-Hack-Gen2)
- **Hoverboard hack from BiPropellant:** [https://github.com/bipropellant](https://github.com/bipropellant)
- **Hoverboard breakout boards:** [https://github.com/Jana-Marie/hoverboard-breakout](https://github.com/Jana-Marie/hoverboard-breakout)

<a/>

- **Bobbycar** [https://github.com/larsmm/hoverboard-firmware-hack-FOC-bbcar](https://github.com/larsmm/hoverboard-firmware-hack-FOC-bbcar)
- **Wheel chair:** [https://github.com/Lahorde/steer_speed_ctrl](https://github.com/Lahorde/steer_speed_ctrl)
- **TranspOtterNG:** [https://github.com/Jan--Henrik/transpOtterNG](https://github.com/Jan--Henrik/transpOtterNG)
- **Hoverboard driver for ROS:** [https://github.com/alex-makarov/hoverboard-driver](https://github.com/alex-makarov/hoverboard-driver)
- **Ongoing OneWheel project:** [https://forum.esk8.news/t/yet-another-hoverboard-to-onewheel-project/60979/14](https://forum.esk8.news/t/yet-another-hoverboard-to-onewheel-project/60979/14)
- **ST Community:** [Custom FOC motor control](https://community.st.com/s/question/0D50X0000B28qTDSQY/custom-foc-control-current-measurement-dma-timer-interrupt-needs-review)
- **Android app for flashed hoverborad with remote control and interface** [Usb connection required](https://github.com/elioscordo/hoverdroid)
<a/>

- **Telegram Community:** The telegram group was closed by the owners, but no problem, we already launched a copy on matrix which is better than telegram anyways :)
- **Matrix Community:** [Join the new matrix group here](https://matrix.to/#/#hooover:brunner.ninja) (we imported all old telegram messages from the past 5+ years including pictures there)

---
## Stargazers

[![Stargazers over time](https://starchart.cc/EFeru/hoverboard-firmware-hack-FOC.svg)](https://starchart.cc/EFeru/hoverboard-firmware-hack-FOC)

---
## Contributions

Every contribution to this repository is highly appreciated! Feel free to create pull requests to improve this firmware as ultimately you are going to help everyone. 

If you want to donate to keep this firmware updated, please use the link below:

[![paypal](https://www.paypalobjects.com/en_US/NL/i/btn/btn_donateCC_LG.gif)](https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=CU2SWN2XV9SCY&currency_code=EUR&source=url)

---
