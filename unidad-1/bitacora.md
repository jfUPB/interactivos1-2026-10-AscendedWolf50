# Unidad 1

## Bitácora de proceso de aprendizaje
### Actividad 01
#### ¿Qué es un sistema físico interactivo?
Es cuando se usa herramientas tecnologicas para generar un estimulo o una experiencia en el mundo real.

#### ¿Cómo podrías aplicar lo que has visto en tu perfil profesional?
Lo podria usar para diseñar mecanicas de juego en espacios fisicos, como sensores de movimiento para simular una mecanica de sigilo.


### Actividad 02
#### ¿Qué es el diseño/arte generativo?
Es cuando el artista no hace el arte directamente, sino que desarrolla un sistema (algoritmo) para que el arte se genere de forma autonoma.

#### ¿Cómo podrías aplicar lo que has visto en tu perfil profesional?
Pues podria combinar las matematicas, el codigo y el diseño para crear nuevos productos como experiencias que puedan entretener o educar a la comunidad.

### Actividad 03
(Actividad guiada)

### Actividad 04

#### ¿Por qué no funcionaba el programa con was_pressed() y por qué funciona con is_pressed()? Explica detalladamente.
Porque was_pressed() solo detecta el momento en el que el boton fue presionado, osea detecta el boton solo ese momento y despues lo "olvida". Is_pressed() siempre detecta el boton mientras este se mantenga presionado, osea el boton va a ser detectado siempre y cuando sea y se mantenga presionado.


## Bitácora de aplicación 
### Actividad 05

#### Crea un programa en p5.js que muestre un círculo en la pantalla. Utiliza los botones A y B del micro:bit para controlar la posición en x del círculo en el canvas de p5.js.

**Micro:Bit**
``` py
from microbit import *

uart.init(baudrate=115200)
display.show(Image.BUTTERFLY)

while True:
    if button_a.is_pressed():
        uart.write('A')
        sleep(500)
    if button_b.is_pressed():
        uart.write('B')
        sleep(500)
    if accelerometer.was_gesture('shake'):
        uart.write('C')
        sleep(500)
    if uart.any():
        data = uart.read(1)
        if data:
            if data[0] == ord('h'):
                display.show(Image.HEART)
                sleep(500)
                display.show(Image.HAPPY)
                
  ```
**p5.js**
``` js
let port;
let connectBtn;
let circleX;

function setup() {
    createCanvas(400, 400);
    port = createSerial();

    connectBtn = createButton('Connect to micro:bit');
    connectBtn.position(135, 300);
    connectBtn.mousePressed(connectBtnClick);

    circleX = width / 2;
}

function draw() {
    background(220); 

    // leer serial 
    while (port.availableBytes() > 0) {
        let dataRx = port.read(1);

        if (dataRx === 'A') {
            circleX -= 20;
        }
        else if (dataRx === 'B') {
            circleX += 20;
        }
    }

    
    
    fill('white');
    ellipse(circleX, height / 2, 100, 100);

    // botón
    connectBtn.html(port.opened() ? 'Disconnect' : 'Connect to micro:bit');
}

function connectBtnClick() {
    if (!port.opened()) {
        port.open('MicroPython', 115200);
    } else {
        port.close();
    }
}
```
##### Explicacion de codigo
El codigo esta compuesto de dos partes, una en Micro:Bit el cual ejecuta un bucle infinito en el que lee constantemente los sensores. Cuando el boton A es presionado, el Micro:Bit envia el caracter "A" por el puerto serial, y cuando el boton B es presionado, se envia el caracter "B".

El codigo en p5.js ejecuta un bucle de dibujo continuo. En cada frame, el codigo limpia el canvas y dibuja un circulo cuya posicion en x esta almacenada por la variable "circlex".
Cuando el codigo detecta informacion en el puerto serial, lee los caracteres enviados y segun el caracter modifica la variable "circlex". El caracter "A" mueve el circulo hacia la izquierda y el "B" hacia la derecha.


## Bitácora de reflexión

### Actividad 06
#####  ¿Cómo funciona el sistema físico interactivo de la Actividad 04?
Primero que todo el codigo en el editor de Micro:Bit esta en un bucle infinito en el que lee constantemente los sensores, cuando se presiona el boton A del Micro:Bit se envia el caracter "A" a traves del puerto serial. 
Ya en el codigo de p5.js, esta este bloque de codigo:
``` js
function setup() {
        createCanvas(400, 400);
        background(220);
        port = createSerial();
        connectBtn = createButton("Connect to micro:bit");
        connectBtn.position(80, 300);
        connectBtn.mousePressed(connectBtnClick);
```
Crea el canvas, abre el puerto serial y crea un boton para poder conectarse con el Micro:Bit.

``` js
    function draw() {
        background(220);

        if (port.availableBytes() > 0) {
            let dataRx = port.read(1);
            if (dataRx == "A") {
            fill("red");
            }
        } else {
            fill("green");
        }
```
Este bloque dibuja constantemente en la pantalla (cada frame) un cuadrado verde, pero si se presiona el boton A en el Micro:Bit el cuadrado va a cambiar a color rojo por un frame. Cuando el boton deja de ser presionado, el cuadrado vuelve a color verde.

``` js
 if (!port.opened()) {
            connectBtn.html("Connect to micro:bit");
        } else {
            connectBtn.html("Disconnect");
        }
    }

    function connectBtnClick() {
        if (!port.opened()) {
            port.open("MicroPython", 115200);
        } else {
            port.close();
        }
    }
```
Este bloque se encarga de conectar con el Micro:Bit en caso de que el puerto haya sido abierto. Si el boton para conectarse al Micro::Bit es presionado (y si el puerto esta abierto) se abre el puerto y establece la velocidad de 115200 para la comunicacion serial.






