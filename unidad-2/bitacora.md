# Unidad 2

## Bitácora de proceso de aprendizaje

### Actividad 01

#### 1. ¿Cuáles son los estados en el programa?
Los estados son WaitInOn y WaitInOff.

#### 2. ¿Cuáles son los eventos en el programa?
El evento de entrada y el evento de Timeout.

#### 3. ¿Cuáles son las acciones en el programa?

- Encender el píxel.
- Apagar el píxel.
- Iniciar el temporizador.
- Reiniciar el temporizador.
- Cambiar de estado (ON - OFF).

### Actividad 02
#### Vas a realizar una modificación. Cuando el semáforo esté en verde, si se presiona el botón A, el semáforo debe cambiar inmediatamente a amarillo (sin esperar a que termine el tiempo de verde). El evento que se debe postear es “A” (post_event(“A”)).

``` py
from microbit import *
import utime
from fsm import Timer, FSMTask, ENTRY

class Semaforo(FSMTask): 
    def __init__(self,_x,_y,_timeInRed,_timeInGreen,_timeInYellow):
        super().__init__()
        self.x = _x
        self.y = _y
        self.timeInRed = _timeInRed
        self.timeInGreen = _timeInGreen
        self.timeInYellow = _timeInYellow
        self.myTimer = self.add_timer("Timeout",self.timeInRed)
        self.transition_to(self.estado_waitInRed)

    def clear(self):
        display.set_pixel(self.x,self.y,0)
        display.set_pixel(self.x,self.y+1,0)
        display.set_pixel(self.x,self.y+2,0)
        
    def estado_waitInRed(self, ev):
         if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y,9)
            self.myTimer.start(self.timeInRed)
         if ev == "Timeout":
            display.set_pixel(self.x,self.y,0)
            self.transition_to(self.estado_waitInGreen)

    def estado_waitInGreen(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+2,9)
            self.myTimer.start(self.timeInGreen)

        if ev == "Timeout":
            display.set_pixel(self.x,self.y+2,0)
            self.transition_to(self.estado_waitInYellow)

        if ev == "A":
            display.set_pixel(self.x,self.y+2,0)
            self.transition_to(self.estado_waitInYellow)

    def estado_waitInYellow(self, ev):
         if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.myTimer.start(self.timeInYellow)
         if ev == "Timeout":
            display.set_pixel(self.x,self.y+1,0)
            self.transition_to(self.estado_waitInRed)

semaforo1 = Semaforo(0,0,2000,1000,500)

    
while True:
    #Input processing
    if button_a.was_pressed(): semaforo1.post_event("A")
        
    semaforo1.update()
    utime.sleep_ms(20)
```

**Maquina de estados en PlantUML**
```
@startuml
title Semáforo - UML State Machine

[*] --> WaitInRed : Semaforo() constructor

' ===== ESTADO ROJO =====
WaitInRed : entry /\n  clear()\n  display.set_pixel(x,y,9)\n  myTimer.start(timeInRed)
WaitInRed --> WaitInGreen : Timeout

' ===== ESTADO VERDE =====
WaitInGreen : entry /\n  clear()\n  display.set_pixel(x,y+2,9)\n  myTimer.start(timeInGreen)
WaitInGreen --> WaitInYellow : Timeout
WaitInGreen --> WaitInYellow : A

' ===== ESTADO AMARILLO =====
WaitInYellow : entry /\n  clear()\n  display.set_pixel(x,y+1,9)\n  myTimer.start(timeInYellow)
WaitInYellow --> WaitInRed : Timeout

@enduml

```

<img width="349" height="575" alt="image" src="https://github.com/user-attachments/assets/0b265721-c0b4-46ce-a8f2-1535fcb56221" />

### Actividad 03
Usé una máquina de estados para controlar las imágenes y el botón A. El timer y el botón solo mandan eventos, y el estado actual decide qué hacer, así el programa no se bloquea y puede responder rápido. Probé que las imágenes cambien en el orden correcto y que al presionar A se salte al estado que corresponde.

## Bitácora de aplicación 
### Actividad 04
#### Ejercicio de la bomba

**Maquina estados en plantUML**
```
@startuml
[*] --> CONFIG

CONFIG : A (n++)
CONFIG : B (n--)
CONFIG --> ARMED : S (shake)

ARMED  : Timeout (n--)
ARMED --> BOOM : n == 0

BOOM --> CONFIG : A (reset)

@enduml
```
<img width="235" height="410" alt="image" src="https://github.com/user-attachments/assets/5e2ab168-d75a-4491-a3a7-511780214a13" />




**Codigo de Micropython**
``` py
from microbit import *
import utime
import music

# -------- Imagenes ----------
def make_fill_images(on='9', off='0'):
    imgs = []
    for n in range(26):
        rows = []
        k = 0
        for y in range(5):
            row = []
            for x in range(5):
                row.append(on if k < n else off)
                k += 1
            rows.append(''.join(row))
        imgs.append(Image(':'.join(rows)))
    return imgs

FILL = make_fill_images()

# -------- Timer ----------
class Timer:
    def __init__(self, owner, event_to_post, duration):
        self.owner = owner
        self.event = event_to_post
        self.duration = duration
        self.start_time = 0
        self.active = False

    def start(self, new_duration=None):
        if new_duration is not None:
            self.duration = new_duration
        self.start_time = utime.ticks_ms()
        self.active = True

    def stop(self):
        self.active = False

    def update(self):
        if self.active:
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                self.active = False
                self.owner.post_event(self.event)

# -------- Maquina de estados Task ----------
class Task:
    def __init__(self):
        self.event_queue = []
        self.timers = []
        self.count = 20   # initial pixels
        
        self.timer = self.createTimer("Timeout", 1000)

        self.estado_actual = None
        self.transicion_a(self.estado_config)

    def createTimer(self, event, duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        for t in self.timers:
            t.update()

        while self.event_queue:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual:
            self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    # -------- Estado "Config" ----------
    def estado_config(self, ev):
        if ev == "ENTRY":
            display.show(FILL[self.count])

        if ev == "A":
            if self.count < 25:
                self.count += 1
                display.show(FILL[self.count])

        if ev == "B":
            if self.count > 15:
                self.count -= 1
                display.show(FILL[self.count])

        if ev == "S":   # shake → arm
            self.transicion_a(self.estado_Armed)

    # -------- Estado "Armed" ----------
    def estado_Armed(self, ev):
        if ev == "ENTRY":
            self.timer.start(1000)

        if ev == "Timeout":
            self.count -= 1
            display.show(FILL[self.count])

            if self.count == 0:
                self.transicion_a(self.estado_BOOM)
            else:
                self.timer.start(1000)

        if ev == "EXIT":
            self.timer.stop()

    # -------- Estado "BOOM" ----------
    def estado_BOOM(self, ev):
     if ev == "ENTRY":
        display.show(Image.SKULL)
        music.play(music.WAWAWAWAA)

     if ev == "A":
        self.count = 20
        self.transicion_a(self.estado_config)


# -------- Loop principal ----------
task = Task()

while True:
    if button_a.was_pressed():
        task.post_event("A")
    if button_b.was_pressed():
        task.post_event("B")
    if accelerometer.was_gesture("shake"):
        task.post_event("S")

    task.update()
    utime.sleep_ms(20)

```




## Bitácora de reflexión





