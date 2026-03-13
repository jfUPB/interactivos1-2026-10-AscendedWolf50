# Unidad 4

## Bitácora de proceso de aprendizaje
### Actividad 01
**Hardware (micro:bit):** Se comunica a 115200 baudios y actualiza datos 10 veces por segundo. Envía un paquete de texto, terminando siempre con un salto de línea (\n).

**Arte Generativo (p5.js):** El reto visual es reemplazar los controles originales del sketch (el mouse o teclado) con los sensores físicos. Será indispensable usar la función map() para adaptar el rango numérico del acelerómetro al tamaño del lienzo (ancho y altura).

**Integración (Repositorio):** El desafío técnico principal es recibir la cadena de texto en JavaScript, separarla por las comas (usando .split(',')) y convertir ese texto en variables numéricas que la función draw() de p5.js pueda utilizar.


## Bitácora de aplicación 
### Actividad 02

#### Capa de Backend: Implementación del Patrón Adapter
**Objetivo:** Traducir el nuevo protocolo serial ($T:tiempo|X:acel_x|Y:acel_y|A:estado_a|B:estado_b|CHK:checksum\n) a un objeto JSON estándar sin modificar la lógica central del servidor.

**Decisión de Arquitectura:** Se creó un nuevo archivo MicrobitV2Adapter.js dentro de la carpeta adapters que hereda de BaseAdapter para cumplir con el contrato de la arquitectura.

**Lógica de Validación:** El dispositivo envía un Checksum (CHK). El adaptador extrae la cadena descartando el $ inicial, usa .split('|') para separar los bloques, y extrae los valores numéricos. Para asegurar que la trama no está corrupta, se calcula el Checksum localmente con la fórmula matemática: Math.abs(X) + Math.abs(Y) + A + B. Si el cálculo no coincide con el CHK recibido, la trama se descarta silenciosamente.

**MicrobitV2Adapter.js**
``` js
const BaseAdapter = require('./BaseAdapter');

class MicrobitV2Adapter extends BaseAdapter {
    constructor() {
        super();
    }

    processData(dataString) {
        // 1. Quitar los espacios y saltos de línea
        const cleanTrama = dataString.trim();

        // 2. Verificar que la trama empiece con "$"
        if (!cleanTrama.startsWith('$')) {
            return; 
        }

        // 3. Quitar el "$" del principio
        const tramaSinDolar = cleanTrama.substring(1);

        // 4. Separar los pedazos usando el pipe "|"
        const partes = tramaSinDolar.split('|');

        // 5. Extraer los valores numéricos
        const valorX = parseInt(partes[1].split(':')[1]);
        const valorY = parseInt(partes[2].split(':')[1]);
        const valorA = parseInt(partes[3].split(':')[1]);
        const valorB = parseInt(partes[4].split(':')[1]);
        const valorCHK = parseInt(partes[5].split(':')[1]);

        // Validar el Checksum
        const miChecksumCalculado = Math.abs(valorX) + Math.abs(valorY) + valorA + valorB;

        if (miChecksumCalculado !== valorCHK) {
            console.warn("⚠️ Trama corrupta descartada. CHK esperado:", valorCHK, "Calculado:", miChecksumCalculado);
            return; 
        }

        // 6. Emitir los datos limpios
        this.onData?.({
            x: valorX,
            y: valorY,
            btnA: valorA === 1,
            btnB: valorB === 1
        });
    }
}

module.exports = MicrobitV2Adapter;
```
**Conexión:** Se modificó el archivo bridgeServer.js para importar este nuevo adaptador: const MicrobitAdapter = require('./adapters/MicrobitV2Adapter');

#### Capa de Frontend: Máquina de Estados (FSM) y Arte Generativo
**Objetivo:** Adaptar el sketch de p5.js (P_2_0_02) a la clase PainterTask usando una Máquina de Estados, separando la lógica de cálculo del renderizado gráfico.

**Decisión de Arquitectura:** El código procedimental de p5.js se dividió en dos partes fundamentales para respetar la FSM:

**updateLogic(data):** Encargada exclusivamente de la ingesta y transformación matemática. Se utilizó la función map() para escalar los valores del acelerómetro (rango de hardware: -2048 a 2047) a las dimensiones del lienzo (Resolución de 2 a 10 lados, y Radio basado en el width).

**drawRunning():** Encargada del renderizado "tonto". Solo lee las variables calculadas previamente y ejecuta las funciones de dibujo (beginShape, trigonometría con sin y cos).

``` js
const EVENTS = {
    CONNECT: "CONNECT",
    DISCONNECT: "DISCONNECT",
    DATA: "DATA",
    KEY_PRESSED: "KEY_PRESSED",
    KEY_RELEASED: "KEY_RELEASED",
};

class PainterTask extends FSMTask {
    constructor() {
        super();

        // Variables de estado para el arte generativo
        this.circleResolution = 2; 
        this.radius = 0;           
        this.isDrawing = false;    
        this.isFilled = false;     

        this.rxData = {
            ready: false
        };

        this.transitionTo(this.estado_esperando);
    }

    estado_esperando = (ev) => {
        if (ev.type === "ENTRY") {
            cursor();
            console.log("Waiting for connection...");
        } else if (ev.type === EVENTS.CONNECT) {
            this.transitionTo(this.estado_corriendo);
        }
    };

    estado_corriendo = (ev) => {
        if (ev.type === "ENTRY") {
            noCursor();
            // Estilos iniciales de la pieza de arte original
            strokeWeight(2);
            stroke(0, 25);
            background(255);
            console.log("Microbit ready to draw");
            this.rxData.ready = false;
        }

        else if (ev.type === EVENTS.DISCONNECT) {
            this.transitionTo(this.estado_esperando);
        }

        else if (ev.type === EVENTS.DATA) {
            this.updateLogic(ev.payload);
        }

        else if (ev.type === "EXIT") {
            cursor();
        }
    };

    updateLogic(data) {
        this.rxData.ready = true;

        // 1. Mapear el acelerómetro Y a la resolución (de 2 a 10 lados)
        this.circleResolution = int(map(data.y, -2048, 2047, 2, 10));

        // 2. Mapear el acelerómetro X al radio (-width/2 a width/2)
        this.radius = map(data.x, -2048, 2047, -width / 2, width / 2);

        // 3. Guardar el estado de los botones
        this.isDrawing = data.btnA; // Controla si se dibuja (como click del mouse)
        this.isFilled = data.btnB;  // Controla el relleno (como presionar tecla)
    }
}

let painter;
let bridge;
let connectBtn;
const renderer = new Map();

function setup() {
    createCanvas(windowWidth, windowHeight);
    background(255);
    painter = new PainterTask();
    bridge = new BridgeClient();

    bridge.onConnect(() => {
        connectBtn.html("Disconnect");
        painter.postEvent({ type: EVENTS.CONNECT });
    });

    bridge.onDisconnect(() => {
        connectBtn.html("Connect");
        painter.postEvent({ type: EVENTS.DISCONNECT });
    });

    bridge.onStatus((s) => {
        console.log("BRIDGE STATUS:", s.state, s.detail ?? "");
    });

    bridge.onData((data) => {
        painter.postEvent({
            type: EVENTS.DATA, payload: {
                x: data.x,
                y: data.y,
                btnA: data.btnA,
                btnB: data.btnB
            }
        });
    });

    connectBtn = createButton("Connect");
    connectBtn.position(10, 10);
    connectBtn.mousePressed(() => {
        if (bridge.isOpen) bridge.close();
        else bridge.open();
    });

    // Conectar el estado de la FSM con la función de renderizado tonto
    renderer.set(painter.estado_corriendo, drawRunning);
}

function draw() {
    painter.update();
    renderer.get(painter.state)?.();
}

function drawRunning() {
    // Si no han llegado datos válidos, no hacemos nada
    if (!painter.rxData.ready) return;

    // Solo dibujamos si el botón A está presionado
    if (painter.isDrawing) {
        push();
        translate(width / 2, height / 2);

        let angle = TAU / painter.circleResolution;

        // Si el botón B está presionado, rellenamos de color
        if (painter.isFilled) {
            fill(34, 45, 122, 50);
        } else {
            noFill();
        }

        // Trazado de la figura geométrica
        beginShape();
        for (let i = 0; i <= painter.circleResolution; i++) {
            let x = cos(angle * i) * painter.radius;
            let y = sin(angle * i) * painter.radius;
            vertex(x, y);
        }
        endShape();

        pop();
    }
}

function windowResized() {
    resizeCanvas(windowWidth, windowHeight);
}
```

Se utilizó la extensión Live Server en VS Code para montar un servidor web estático en el puerto 5500. Esto permitió cargar la interfaz gráfica (index.html y el lienzo de p5.js) correctamente, desde donde el script establece la conexión limpia por debajo con el puerto 8081.
## Bitácora de reflexión
