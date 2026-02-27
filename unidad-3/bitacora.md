# Unidad 3

## Bitácora de proceso de aprendizaje
### Actividad 01


### Actividad 03

## Bitácora de aplicación 

### Actividad 04
#### Codigo original funcionando con Micro:Bit: 
**Sketch.js**
``` js
// --- CONSTANTES Y CONFIGURACIÓN ORIGINAL ---
const TIMER_LIMITS = {
  min: 15,
  max: 25,
  defaultValue: 20,
};

const EVENTS = {
  DEC: "A",
  INC: "B",
  START: "S",
  TICK: "Timeout",
};

// --- VARIABLES GLOBALES ---
let temporizador;
let serial; 
let portButton;
const renderer = new Map();

// --- CLASE TEMPORIZADOR (FSM) ---
class Temporizador extends FSMTask {
  constructor(minValue, maxValue, defaultValue) {
    super();
    this.minValue = minValue;
    this.maxValue = maxValue;
    this.defaultValue = defaultValue;
    this.configValue = defaultValue;
    this.totalSeconds = defaultValue;
    this.remainingSeconds = defaultValue;

    this.myTimer = this.addTimer(EVENTS.TICK, 1000);
    this.transitionTo(this.estado_config);
  }

  get currentState() { return this.state; }

  estado_config = (ev) => {
    if (ev === ENTRY) { this.configValue = this.defaultValue; }
    else if (ev === EVENTS.DEC) {
      if (this.configValue > this.minValue) this.configValue--;
    } else if (ev === EVENTS.INC) {
      if (this.configValue < this.maxValue) this.configValue++;
    } else if (ev === EVENTS.START) {
      this.totalSeconds = this.configValue;
      this.remainingSeconds = this.totalSeconds;
      this.transitionTo(this.estado_armed);
    }
  };

  estado_armed = (ev) => {
    if (ev === ENTRY) { this.myTimer.start(); }
    else if (ev === EVENTS.TICK) {
      if (this.remainingSeconds > 0) {
        this.remainingSeconds--;
        if (this.remainingSeconds === 0) {
          this.transitionTo(this.estado_timeout);
        } else { this.myTimer.start(); }
      }
    } else if (ev === EXIT) { this.myTimer.stop(); }
  };

  estado_timeout = (ev) => {
    if (ev === ENTRY) { console.log("¡TIEMPO!"); }
    else if (ev === EVENTS.DEC) { this.transitionTo(this.estado_config); }
  };
}

// --- FUNCIONES DE P5.JS ---

function setup() {
  createCanvas(windowWidth, windowHeight);
  
  // Inicialización serial
  serial = createSerial();
  
  portButton = createButton('CONECTAR MICRO:BIT');
  portButton.position(20, 20);
  portButton.mousePressed(connectSerial);

  temporizador = new Temporizador(
    TIMER_LIMITS.min,
    TIMER_LIMITS.max,
    TIMER_LIMITS.defaultValue
  );

  textAlign(CENTER, CENTER);

  renderer.set(temporizador.estado_config, () => drawConfig(temporizador.configValue));
  renderer.set(temporizador.estado_armed, () => drawArmed(temporizador.remainingSeconds, temporizador.totalSeconds));
  renderer.set(temporizador.estado_timeout, () => drawTimeout());
}

function draw() {
  // LECTURA ROBUSTA: Leemos todo lo disponible y buscamos los comandos
  if (serial.opened() && serial.available() > 0) {
    let rawData = serial.read(); // Lee el contenido del búfer
    
    if (rawData) {
      let msg = str(rawData).toUpperCase(); // Lo convertimos a texto
      console.log("Datos recibidos:", msg); // Esto nos servirá para debuggear
      
      // Buscamos si el mensaje contiene alguna de nuestras letras
      if (msg.includes("A")) temporizador.postEvent("A");
      if (msg.includes("B")) temporizador.postEvent("B");
      if (msg.includes("S")) temporizador.postEvent("S");
    }
  }

  temporizador.update();
  renderer.get(temporizador.currentState)?.();
}

// --- FUNCIONES DE DIBUJO ---

function drawConfig(val) {
  background(20, 40, 80);
  fill(255);
  textSize(120);
  text(val, width / 2, height / 2);
  textSize(18);
  fill(200);
  text("MICRO:BIT -> A(-), B(+), Agitar(Start)", width / 2, height / 2 + 100);
}

function drawArmed(val, total) {
  background(20, 20, 20);
  let pulse = sin(frameCount * 0.1) * 10;
  noFill();
  strokeWeight(20);
  stroke(255, 100, 0, 50);
  ellipse(width / 2, height / 2, 250);
  stroke(255, 150, 0);
  let angle = map(val, 0, total, 0, TWO_PI);
  arc(width / 2, height / 2, 250, 250, -HALF_PI, angle - HALF_PI);
  fill(255);
  noStroke();
  textSize(100 + pulse);
  text(val, width / 2, height / 2);
}

function drawTimeout() {
  let bg = frameCount % 20 < 10 ? color(150, 0, 0) : color(255, 0, 0);
  background(bg);
  fill(255);
  textSize(100);
  text("¡TIEMPO!", width / 2, height / 2);
}

// --- CONTROL ADICIONAL ---

function keyPressed() {
  if (key === "a" || key === "A") temporizador.postEvent(EVENTS.DEC);
  if (key === "b" || key === "B") temporizador.postEvent(EVENTS.INC);
  if (key === "s" || key === "S") temporizador.postEvent(EVENTS.START);
}

async function connectSerial() {
  if (!serial.opened()) {
    await serial.open(9600);
    portButton.hide();
  }
}

function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
}

```
**index.html**
``` html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0">

    <title>Sketch</title>

    <link rel="stylesheet" type="text/css" href="style.css">

    <script src="https://cdn.jsdelivr.net/npm/p5@1.11.11/lib/p5.js"></script>
    
    <script src="https://unpkg.com/@gohai/p5.webserial@latest/libraries/p5.webserial.js"></script>
  </head>

  <body>
    <script src="fsm.js"></script>
    <script src="sketch.js"></script>
  </body>
</html>
```

**fsm.js**
``` js
const ENTRY = "ENTRY";
const EXIT = "EXIT";

class Timer {
  constructor(owner, eventToPost, duration) {
    this.owner = owner;
    this.event = eventToPost;
    this.duration = duration;
    this.startTime = 0;
    this.active = false;
  }

  start(newDuration = null) {
    if (newDuration !== null) this.duration = newDuration;
    this.startTime = millis();
    this.active = true;
  }

  stop() {
    this.active = false;
  }

  update() {
    if (this.active && millis() - this.startTime >= this.duration) {
      this.active = false;
      this.owner.postEvent(this.event);
    }
  }
}

class FSMTask {
  constructor() {
    this.queue = [];
    this.timers = [];
    this.state = null;
  }

  postEvent(ev) {
    this.queue.push(ev);
  }

  addTimer(event, duration) {
    let t = new Timer(this, event, duration);
    this.timers.push(t);
    return t;
  }

  transitionTo(newState) {
    if (this.state) this.state(EXIT);
    this.state = newState;
    this.state(ENTRY);
  }

  update() {
    for (let t of this.timers) {
      t.update();
    }
    while (this.queue.length > 0) {
      let ev = this.queue.shift();
      if (this.state) this.state(ev);
    }
  }
}
```
**Micro:Bit**
``` py
from microbit import *

uart.init(baudrate=9600)
display.show(Image.BUTTERFLY)

while True:
    if button_a.was_pressed():
        uart.write('A')
       
    if button_b.was_pressed():
        uart.write('B')
        
    if accelerometer.was_gesture('shake'):
        uart.write('S')
        
 ```   
                
## Bitácora de reflexión




