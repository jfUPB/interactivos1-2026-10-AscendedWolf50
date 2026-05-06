# Unidad 8

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 
### Actividad 01

<img width="1223" height="785" alt="image" src="https://github.com/user-attachments/assets/63393ab7-0220-4d2a-acf9-3a33ed2bdfec" />


#### Los Adapters que vas a usar
Se están utilizando tres adaptadores independientes para respetar el principio de responsabilidad única en el backend:

**MicrobitAsciiAdapter:** Se encarga de abrir el puerto serial (a 115200 baudios), leer las tramas en formato CSV plano y normalizar los datos del acelerómetro y botones.

**StrudelAdapter:** Levanta un servidor WebSocket local en el puerto 8080 exclusivo para escuchar los eventos rítmicos generados por el motor de audio.

**OscAdapter:** Levanta un servidor UDP en el puerto 8000 dedicado a interceptar los paquetes de control enviados desde la interfaz y extraer sus direcciones (address) y argumentos (args).

#### El contrato de mensajes de cada fuente
El servidor bridgeServer.js empaqueta la salida de los tres adaptadores bajo un estándar predecible para el cliente web:

**Contrato de Micro:bit:**
{ type: "microbit", x: int, y: int, btnA: bool, btnB: bool, t: ms }

**Contrato de Strudel:**
{ type: "strudel", timestamp: ms, payload: { s: string, delta: float } }

**Contrato de Open Stage Control:**
{ type: "osc", payload: { address: string, args: array } }

#### Pruebas técnicas de integración
Se valida la concurrencia del sistema ejecutando el comando --device all. Tras confirmar la apertura de los tres puertos (Serial, WS 8080 y UDP 8000), se enviaron datos simultáneos desde las tres fuentes. El frontend procesó y enrutó la información correctamente a sus respectivos componentes (updateLogic para Micro:bit, eventQueue para Strudel y persistentState para OSC), logrando un renderizado estable y sin pérdida de rendimiento.

#### Problemas encontrados y soluciones

Exclusion de dispositivos en el backend: El servidor original solo permitía ejecutar un dispositivo a la vez.

**Solucion:** Se refactorizó la funcion main() implementando un arreglo de adaptadores activos, lo que permite la inicialización simultánea de los tres módulos mediante el comando --device all.

**Error de parseo en Micro:bit:** El uso de una trama compleja con checksum generaba errores de validación al leer los datos.

**Solucion:** Se simplificó el script de la placa para que emita una trama en formato CSV estándar (X,Y,btnA,btnB\n), asegurando la compatibilidad directa con el MicrobitAsciiAdapter.

**bridgeServer.js**
```js
//   Uso Unidad 8 (Performance):
//     node bridgeServer.js --device all

const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");
const SimAdapter = require("./adapters/SimAdapter");
const MicrobitAsciiAdapter = require("./adapters/MicrobitASCIIAdapter");
const MicrobitV2Adapter = require("./adapters/MicrobitV2Adapter");
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");
const StrudelAdapter = require("./adapters/StrudelAdapter");
const OscAdapter = require("./adapters/OSCAdapter");

const log = {
  info: (...args) => console.log(`[${new Date().toISOString()}] [INFO]`, ...args),
  warn: (...args) => console.warn(`[${new Date().toISOString()}] [WARN]`, ...args),
  error: (...args) => console.error(`[${new Date().toISOString()}] [ERROR]`, ...args)
};

function getArg(name, def = null) {
  const i = process.argv.indexOf(`--${name}`);
  if (i >= 0 && i + 1 < process.argv.length) return process.argv[i + 1];
  return def;
}

function hasFlag(name) { return process.argv.includes(`--${name}`); }
function nowMs() { return Date.now(); }

function safeJsonParse(s) {
  try { return JSON.parse(s); } 
  catch (e) { log.warn("Failed to parse JSON: ", s, e); return null; }
}

function broadcast(wss, obj) {
  const text = JSON.stringify(obj);
  for (const client of wss.clients) {
    if (client.readyState === 1) client.send(text);
  }
}

function status(wss, state, detail = "") {
  broadcast(wss, { type: "status", state, detail, t: nowMs() });
}

const DEVICE = (getArg("device", "sim") || "sim").toLowerCase();
const WS_PORT = parseInt(getArg("wsPort", "8081"), 10);
const SERIAL_PATH = getArg("serialPort", null);
const BAUD = parseInt(getArg("baud", "115200"), 10);
const SIM_HZ = parseInt(getArg("hz", "30"), 10);
const VERBOSE = hasFlag("verbose");

async function findMicrobitPort() {
  const ports = await SerialPort.list();
  const microbit = ports.find(p => p.vendorId && parseInt(p.vendorId, 16) === 0x0D28);
  return microbit?.path ?? null;
}

async function main() {
  const wss = new WebSocketServer({ port: WS_PORT });
  log.info(`WS listening on ws://127.0.0.1:${WS_PORT} device=${DEVICE}`);

  let activeAdapters = [];

  // === MODO PERFORMANCE (UNIDAD 8) ===
  if (DEVICE === "all") {
    log.info("🚀 INICIANDO MODO PERFORMANCE: Micro:bit + Strudel + OSC");
    
    // 1. Instanciar Micro:bit
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (path) {
      log.info(`Micro:bit detectado en ${path}`);
      activeAdapters.push(new MicrobitAsciiAdapter({ path, baud: BAUD, verbose: VERBOSE }));
    } else {
      log.warn("Micro:bit no encontrado. Cargando Simulador de respaldo.");
      activeAdapters.push(new SimAdapter({ hz: SIM_HZ }));
    }

    // 2. Instanciar Strudel
    activeAdapters.push(new StrudelAdapter({ port: 8080 }));

    // 3. Instanciar OSC
    activeAdapters.push(new OscAdapter({ port: 8000 }));

  } else {
    // Mantenemos retrocompatibilidad con unidades anteriores si no se usa --device all
    if (DEVICE === "strudel") {
      activeAdapters.push(new StrudelAdapter({ port: 8080 }));
      activeAdapters.push(new OscAdapter({ port: 8000 }));
    } else {
       const path = SERIAL_PATH ?? await findMicrobitPort();
       if(path) activeAdapters.push(new MicrobitAsciiAdapter({ path, baud: BAUD, verbose: VERBOSE }));
       else activeAdapters.push(new SimAdapter({ hz: SIM_HZ }));
    }
  }

  // Configurar los eventos para todos los adaptadores activos
  for (const adapter of activeAdapters) {
    adapter.onConnected = (detail) => {
      log.info(`[ADAPTER] Connected: ${detail}`);
      status(wss, "connected", detail);
    };

    adapter.onDisconnected = (detail) => {
      log.warn(`[ADAPTER] Disconnected: ${detail}`);
    };

    adapter.onError = (detail) => {
      log.error(`[ADAPTER] Error: ${detail}`);
    };

    adapter.onData = (d) => {
      // Normalización de salida: identificamos de qué adaptador viene y lo mandamos a la web
      if (d.type === "osc") {
        broadcast(wss, d);
      } else if (d.strudelTimestamp) {
        broadcast(wss, { type: "strudel", timestamp: d.strudelTimestamp, payload: { s: d.s, delta: d.delta } });
      } else {
        broadcast(wss, { type: "microbit", x: d.x, y: d.y, btnA: !!d.btnA, btnB: !!d.btnB, t: nowMs() });
      }
    };
  }

  status(wss, "ready", `bridge up (${DEVICE})`);

  wss.on("connection", (ws, req) => {
    log.info(`[NETWORK] Remote Client connected. Total clients: ${wss.clients.size}`);
    
    // Al conectarse un cliente enviamos el status
    ws.send(JSON.stringify({ type: "status", state: "connected", detail: `bridge (${DEVICE})`, t: nowMs() }));

    ws.on("message", async (raw) => {
      const msg = safeJsonParse(raw.toString("utf8"));
      if (!msg) return;

      if (msg.cmd === "connect") {
        log.info(`[NETWORK] Client requested connect`);
        for (const adapter of activeAdapters) {
            try { await adapter.connect(); } 
            catch (e) { log.error(`[ADAPTER] Connect failed: ${e.message || e}`); }
        }
        return;
      }

      if (msg.cmd === "disconnect") {
        if (wss.clients.size > 1) return;
        for (const adapter of activeAdapters) {
            try { await adapter.disconnect(); } 
            catch (e) { log.error(`[ADAPTER] Disconnect failed: ${e.message || e}`); }
        }
        return;
      }
    });

    ws.on("close", () => {
      if (wss.clients.size === 0) {
        log.info("[HW-POLICY] No clients left. Disconnecting adapters.");
        for (const adapter of activeAdapters) adapter.disconnect();
      }
    });
  });

  // Autoconectar adaptadores de red (OSC, Strudel) y Sim al iniciar
  for (const adapter of activeAdapters) {
      if (adapter instanceof StrudelAdapter || adapter instanceof OscAdapter || adapter instanceof SimAdapter) {
          await adapter.connect();
      }
  }
}

main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
});

```

**sketch.js**
```js
const EVENTS = {
    CONNECT: "CONNECT",
    DISCONNECT: "DISCONNECT",
    DATA: "DATA",
    STRUDEL_DATA: "STRUDEL_DATA",
    OSC_DATA: "OSC_DATA", 
    KEY_PRESSED: "KEY_PRESSED",
    KEY_RELEASED: "KEY_RELEASED",
};

class PainterTask extends FSMTask {
    constructor() {
        super();

        // Variables visuales Micro:bit
        this.c = color(181, 157, 0);
        this.lineSize = 100;
        this.angle = 0;
        this.clickPosX = 0;
        this.clickPosY = 0;

        // Estado de recepción Micro:bit
        this.rxData = {
            x: 0,
            y: 0,
            btnA: false,
            btnB: false,
            prevA: false,
            prevB: false,
            ready: false
        };

        // Estado y Scheduling para Strudel
        this.eventQueue = [];
        this.activeAnimations = [];
        this.latencyCorrection = 0;

        // Estado persistente para Open Stage Control
        this.persistentState = {
            colorRGB: null,     
            globalScale: 1.0,   
            glitchMode: false   
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
            strokeWeight(0.75);
            background(255);
            console.log("System ready to draw (Microbit + Strudel + OSC)");
            
            this.rxData.ready = false;
            this.eventQueue = [];
            this.activeAnimations = [];
        }

        else if (ev.type === EVENTS.DISCONNECT) {
            this.transitionTo(this.estado_esperando);
        }

        else if (ev.type === EVENTS.DATA) {
            this.updateLogic(ev.payload);
        }

        else if (ev.type === EVENTS.STRUDEL_DATA) {
            this.eventQueue.push({
                timestamp: ev.payload.timestamp,
                sound: ev.payload.payload.s,
                delta: ev.payload.payload.delta
            });
            this.eventQueue.sort((a, b) => a.timestamp - b.timestamp);
        }

        else if (ev.type === EVENTS.OSC_DATA) {
            this.updateOscLogic(ev.payload);
        }

        else if (ev.type === EVENTS.KEY_PRESSED) {
            this.handleKeys(ev.keyCode, ev.key);
        }

        else if (ev.type === EVENTS.KEY_RELEASED) {
            this.handleKeyRelease(ev.keyCode, ev.key);
        }

        else if (ev.type === "EXIT") {
            cursor();
        }
    };

    updateLogic(data) {
        this.rxData.ready = true;
        this.rxData.x = map(data.x, -2048, 2047, 0, width);
        this.rxData.y = map(data.y, -2048, 2047, 0, height);
        this.rxData.btnA = data.btnA;
        this.rxData.btnB = data.btnB;

        if (this.rxData.btnA && !this.prevA) {
            this.lineSize = random(50, 160);
            this.clickPosX = this.rxData.x;
            this.clickPosY = this.rxData.y;
            console.log("A pressed");
        }

        if (!this.rxData.btnB && this.prevB) {
            this.c = color(random(255), random(255), random(255), random(80, 100));
            console.log("B released");
        }

        this.prevA = this.rxData.btnA;
        this.prevB = this.rxData.btnB;
    }

    updateOscLogic(payload) {
        let addressLimpio = payload.address.replace('//', '/');

        switch(addressLimpio) {
            case "/rgb_1":
                this.persistentState.colorRGB = payload.args;
                break;
            case "/slider_scale":
                this.persistentState.globalScale = payload.args[0];
                break;
            case "/toggle_glitch":
                this.persistentState.glitchMode = (payload.args[0] === 1);
                break;
        }
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
        // Enrutador estricto según la fuente (Unidad 8)
        if (data.type === "strudel") {
            painter.postEvent({ type: EVENTS.STRUDEL_DATA, payload: data });
        } else if (data.type === "osc") { 
            painter.postEvent({ type: EVENTS.OSC_DATA, payload: data.payload });
        } else if (data.type === "microbit") { 
            painter.postEvent({
                type: EVENTS.DATA, payload: {
                    x: data.x,
                    y: data.y,
                    btnA: data.btnA,
                    btnB: data.btnB
                }
            });
        }
    });

    connectBtn = createButton("Connect");
    connectBtn.position(10, 10);
    connectBtn.mousePressed(() => {
        if (bridge.isOpen) bridge.close();
        else bridge.open();
    });

    renderer.set(painter.estado_corriendo, drawRunning);
}

function draw() {
    painter.update();
    renderer.get(painter.state)?.();
}

function drawRunning() {
    background(255, 40);

    // 1. RENDERIZADO STRUDEL
    let now = Date.now() + painter.latencyCorrection;

    while (painter.eventQueue.length > 0 && now >= painter.eventQueue[0].timestamp) {
        let ev = painter.eventQueue.shift();

        painter.activeAnimations.push({
            startTime: ev.timestamp,
            duration: ev.delta * 1000,
            type: ev.sound,
            x: random(width * 0.2, width * 0.8),
            y: random(height * 0.2, height * 0.8),
            c: getColorForSound(ev.sound)
        });
    }

    for (let i = painter.activeAnimations.length - 1; i >= 0; i--) {
        let anim = painter.activeAnimations[i];
        let elapsed = now - anim.startTime;
        let progress = elapsed / anim.duration;

        if (progress <= 1.0) {
            dibujarElemento(anim, progress, painter.persistentState);
        } else {
            painter.activeAnimations.splice(i, 1);
        }
    }

    // 2. RENDERIZADO MICRO:BIT
    let mb = painter.rxData;
    if (!mb.ready) return;

    if (mb.btnA) {
        let x = mb.x;
        let y = mb.y;
        
        if (painter.persistentState.glitchMode) {
            x += random(-15, 15);
            y += random(-15, 15);
        }

        push();
        translate(x, y);
        rotate(radians(painter.angle));
        
        let cLine = painter.persistentState.colorRGB 
            ? color(painter.persistentState.colorRGB[0], painter.persistentState.colorRGB[1], painter.persistentState.colorRGB[2]) 
            : painter.c;
            
        stroke(cLine);
        strokeWeight(1.5 * painter.persistentState.globalScale);
        
        let lSize = painter.lineSize * painter.persistentState.globalScale;
        line(0, 0, lSize, lSize);
        painter.angle += 1;
        pop();
    }
}

// FUNCIONES DE DIBUJO STRUDEL
function aplicarGlitch(x, y, isGlitch) {
    if (isGlitch) {
        return { x: x + random(-20, 20), y: y + random(-20, 20) };
    }
    return { x, y };
}

function dibujarElemento(anim, p, estadoPersistente) {
    push();
    const c = estadoPersistente.colorRGB || anim.c; 
    
    switch (anim.type) {
        case 'tr909bd': dibujarBombo(anim, p, c, estadoPersistente); break;
        case 'tr909sd': dibujarCaja(anim, p, c, estadoPersistente); break;
        case 'tr909hh': 
        case 'tr909oh': dibujarHat(anim, p, c, estadoPersistente); break;
        default: dibujarDefault(anim, p, c, estadoPersistente); break;
    }
    pop();
}

function dibujarBombo(anim, p, c, estado) {
    let d = lerp(100, 600, p) * estado.globalScale;
    let alpha = lerp(255, 0, p);
    let pos = aplicarGlitch(anim.x, anim.y, estado.glitchMode);
    
    fill(c[0], c[1], c[2], alpha);
    noStroke();
    circle(pos.x, pos.y, d);
}

function dibujarCaja(anim, p, c, estado) {
    let w = lerp(width, 0, p) * estado.globalScale;
    let h = 50 * estado.globalScale;
    let alpha = lerp(255, 0, p);
    let pos = aplicarGlitch(anim.x, anim.y, estado.glitchMode);
    
    fill(c[0], c[1], c[2], alpha);
    noStroke();
    rect(pos.x, pos.y, w, h);
}

function dibujarHat(anim, p, c, estado) {
    let sz = lerp(40, 0, p) * estado.globalScale;
    let pos = aplicarGlitch(anim.x, anim.y, estado.glitchMode);
    
    fill(c[0], c[1], c[2]);
    noStroke();
    rect(pos.x, pos.y, sz, sz);
}

function dibujarDefault(anim, p, c, estado) {
    let size = lerp(100, 0, p) * estado.globalScale;
    let angle = p * TWO_PI;
    let pos = aplicarGlitch(anim.x, anim.y, estado.glitchMode);
    
    translate(pos.x, pos.y);
    rotate(angle);
    stroke(c[0], c[1], c[2]);
    strokeWeight(2);
    noFill();
    rect(0, 0, size, size);
    line(-size, 0, size, 0);
    line(0, -size, 0, size);
}

function getColorForSound(s) {
    if (s.includes('bd')) return [255, 0, 80];
    if (s.includes('sd') || s.includes('cp')) return [0, 200, 255];
    if (s.includes('hh') || s.includes('oh')) return [255, 255, 0];
    return [0, 0, 0];
}

function windowResized() {
    resizeCanvas(windowWidth, windowHeight);
}
```

**Micro:bit** 
```py
from microbit import *

# Inicializar puerto serial a la velocidad correcta
uart.init(baudrate=115200)

while True:
    x = accelerometer.get_x()
    y = accelerometer.get_y()
    
    # El adaptador estándar espera texto: "true" o "false"
    a = "true" if button_a.is_pressed() else "false"
    b = "true" if button_b.is_pressed() else "false"
    
    # Formato CSV: X,Y,btnA,btnB
    trama = "{},{},{},{}\n".format(x, y, a, b)
    
    uart.write(trama)
    sleep(50) # 20Hz (milisegundos) para un dibujo fluido
```






## Bitácora de reflexión
