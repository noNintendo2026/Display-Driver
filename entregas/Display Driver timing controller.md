Este repositorio es una copia del repositorio [Display Driver](https://github.com/Fdiaz718/Digital-1_2026-2_Felipe-Diaz/blob/main/entregas/02_protocolo_HUB75.md)
Se agrega a este repositorio para que quede constancia del trabajo y que el resto de equipos lo puedan ver, pero los cambios y commits originales se hicieron en el repositorio listado arriba.
# Protocolos de Comunicación y Control de Pantalla (Display Driver)

En este documento se detalla la interfaz de comunicación del modulo Display Driver. Al ser el driver de video del sistema, el bloque debe manejar dos protocolos distintos: un **Protocolo de Entrada** (para recibir la información gráfica desde la lógica del juego o bram) y un **Protocolo de Salida** (para controlar físicamente la matriz LED).

## 1. Protocolo de Entrada: Framebuffer y Actualización Dinámica

La capa física opera bajo el estándar de comunicación SPI; sin embargo, el módulo de pantalla implementa una capa de aplicación basada en un mapa de memoria (Framebuffer). La lógica del sistema no transmite comandos de alto nivel para el comportamiento de los elementos en el juego, sino que efectúa escrituras directas sobre coordenadas específicas de la memoria de video.

### 1.1. Recepción de Datos y Diagrama de Tiempos
El procesador principal actúa como dispositivo Maestro (Controller) y la FPGA como Esclavo (Peripheral). La comunicación sigue el diagrama de tiempos estándar:
* La señal de selección de chip (`CS` o `NSS`) se pone en estado bajo (0V) para indicar el inicio de una transmisión activa.
* El Controlador genera la señal de reloj (`SCK`)].
* Los datos gráficos se envían a través de la línea `MOSI` (también llamada `COPI`) y son muestreados por nuestro módulo en los flancos correspondientes del reloj.
* La línea `MISO` (`CIPO`) se mantiene inactiva (o en un estado constante) ya que nuestro driver de pantalla solo recibe datos y no necesita responder al controlador.
* Una vez finalizada la transmisión del paquete de datos, la señal `CS` (o `NSS`) vuelve a su estado inactivo en alto.

![Diagrama de tiempos SPI](./02_spi_timing.png)

### 1.2. Protocolo de Actualización de Sprites
El desplazamiento de objetos gráficos en pantalla (sprites) requiere la sobrescritura de los datos previos en memoria. El flujo de actualización ejecutado por la lógica del sistema consta de los siguientes pasos secuenciales:
1. **Borrado (Background):** La lógica envía las coordenadas anteriores del sprite y las pinta del color del fondo (ej. Negro).
2. **Dibujado (Foreground):** La lógica calcula la nueva posición (X, Y) y envía una ráfaga de datos por SPI con los nuevos colores RGB565 para pintar el personaje en su nueva ubicación.
3. **Actualización en Hardware:** El controlador de pantalla recibe la trama de 32 bits (Comando + Coordenada + Color), decodifica la dirección y actualiza de manera síncrona la memoria BRAM interna del framebuffer.

## 2. Protocolo de Salida: Control HUB75 (Matriz LED)

Una vez que la imagen está en la memoria de la FPGA, nuestro driver debe enviarla a la matriz LED. Estos paneles operan mediante un protocolo paralelo de multiplexación y barrido constante denominado HUB75.
Este protocolo exige una sincronización rigurosa generada desde el hardware de la FPGA, controlando las siguientes señales lógicas:

* **Datos RGB (`R1, G1, B1, R2, G2, B2`):** Se envían 2 píxeles al mismo tiempo (uno para la mitad superior de la pantalla y otro para la mitad inferior).
* **Reloj (`CLK`):** Por cada flanco de subida, la matriz desplaza y guarda los colores en sus registros internos (Shift Registers). Se necesitan 64 pulsos de reloj para llenar una fila completa.
* **Output Enable (`OE`):** Señal activa en bajo. Debe ponerse en ALTO (apagar LEDs) mientras se cambia de fila para evitar un efecto de "fantasmeo" (ghosting) o parpadeo. También se modula por ancho de pulso (PWM) para controlar el brillo general o la profundidad de color.
* **Latch / Strobe (`LAT`):** Un pulso rápido en alto al final de los 64 ciclos de reloj para aplicar los datos guardados en los registros a los LEDs visibles.
* **Direccionamiento de Fila (`A, B, C, D, E`):** Pines binarios que seleccionan cuál de las 32 filas físicas de la matriz se va a encender en ese instante.
![Puerto HUB75](./02_d1.svg)

### Máquina de Estados del Ciclo de Barrido
1. `OE` = 1 (Pantalla apagada).
2. Se cambian los pines `A, B, C, D, E` a la siguiente fila.
3. `OE` = 0 (Pantalla encendida mostrando la fila anterior).
4. Mientras la fila anterior brilla, se envían 64 pulsos de `CLK` empujando los datos RGB de la *nueva* fila.
5. Se envía un pulso de `LAT` para fijar los datos.
6. El ciclo se repite a altísima velocidad (más de 1000 veces por segundo) para engañar al ojo humano y crear una imagen estática estable.

## Reglas de Temporización y Operación
Para garantizar una correcta visualización y evitar artefactos gráficos, el control de timing debe seguir estas condiciones:

* **Sincronismo de Datos:** Las únicas señales que dependen estrictamente de los flancos del reloj (`clk`) son las líneas de datos `rgb`. Las señales de control (`oe`, `latch` y `addr`) pueden variar de forma asíncrona con respecto al reloj.
* **Prevención de "Ghosting":** Si el direccionamiento de fila (`addr`) cambia mientras la pantalla está encendida (`oe` en estado lógico bajo), o si se fija una nueva fila en ese instante, se producirá un efecto de "fantasmeo" o solapamiento de líneas[cite: 10]. Por lo tanto, `latch` y `addr` deben cambiar única y exclusivamente cuando `oe` está deshabilitado (estado lógico alto).
* **Compatibilidad de Hardware:** Algunos paneles específicos exigen por diseño que la señal `oe` se encuentre en estado alto mientras la señal `latch` esté en estado alto.
* **Control de Brillo:** El nivel de brillo de los LEDs depende directamente de la cantidad de tiempo que `oe` permanece en estado bajo. Para evitar variaciones de luminosidad entre distintas filas, el tiempo de activación de `oe` debe ser estrictamente igual para cada una de ellas.
* **Precisión del Reloj:** Se debe generar exactamente la misma cantidad de ciclos de reloj como píxeles tenga el ancho de la pantalla antes de activar el `latch` (por ejemplo, 64 ciclos)[cite: 10]. Un número menor desplazará la imagen; los ciclos adicionales emitidos después del pulso de `latch` serán ignorados por el hardware del panel.
* ![HUB75 timing](./02_wd1.png)

## 3. Implementación Síncrona Sugerida
El segundo diagrama propone una alternativa de diseño donde las señales de control (`oe`, `latch` y `addr`) se tratan de manera síncrona con el reloj. 

La principal ventaja de este enfoque es que elimina la necesidad de implementar un reloj condicionado (gated clock) en el hardware. Bajo este esquema, la señal `oe` mantiene una longitud constante (calculada como el ancho del panel menos 3 ciclos de reloj), simplificando el diseño de la máquina de estados en el código HDL.

![HUB75 timing suggestion](./02_wd2.png)

Informacion tomada de [Moonbaseotago](http://www.moonbaseotago.com/hub75/) y [vanhunteradams](https://vanhunteradams.com/Protocols/SPI/SPI.html)

