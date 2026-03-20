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
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

class ParseError extends Error {}

function parseFrame(line) {

  if (!line.startsWith("$")) {
    throw new ParseError("Frame must start with $");
  }

  const body = line.slice(1);
  const parts = body.split("|");

  const data = {};

  for (const p of parts) {
    const [k, v] = p.split(":");
    data[k] = v;
  }

  const x = parseInt(data.X);
  const y = parseInt(data.Y);
  const a = parseInt(data.A);
  const b = parseInt(data.B);
  const chk = parseInt(data.CHK);

  if (!Number.isFinite(x) || !Number.isFinite(y)) {
    throw new ParseError("Invalid accelerometer data");
  }

  if (![0, 1].includes(a) || ![0, 1].includes(b)) {
    throw new ParseError("Invalid button values");
  }

  const calc = Math.abs(x) + Math.abs(y) + a + b;

  if (calc !== chk) {
    throw new ParseError(`Checksum mismatch expected ${chk} got ${calc}`);
  }

  return {
    x,
    y,
    btnA: a === 1,
    btnB: b === 1
  };
}

class MicrobitV2Adapter extends BaseAdapter {

  constructor({ path, baud = 115200, verbose = false } = {}) {
    super();
    this.path = path;
    this.baud = baud;
    this.port = null;
    this.buf = "";
    this.verbose = verbose;
  }

  async connect() {

    if (this.connected) return;
    if (!this.path) throw new Error("serialPort is required");

    this.port = new SerialPort({
      path: this.path,
      baudRate: this.baud,
      autoOpen: false,
    });

    await new Promise((resolve, reject) => {
      this.port.open((err) => (err ? reject(err) : resolve()));
    });

    this.connected = true;
    this.onConnected?.(`serial open ${this.path} @${this.baud}`);

    this.port.on("data", (chunk) => this._onChunk(chunk));
    this.port.on("error", (err) => this._fail(err));
    this.port.on("close", () => this._closed());
  }

  async disconnect() {

    if (!this.connected) return;

    this.connected = false;

    if (this.port && this.port.isOpen) {
      await new Promise((resolve, reject) => {
        this.port.close((err) => {
          if (err) reject(err);
          else resolve();
        });
      });
    }

    this.port = null;
    this.buf = "";

    this.onDisconnected?.("serial closed");
  }

  getConnectionDetail() {
    return `serial open ${this.path}`;
  }

  _onChunk(chunk) {

    this.buf += chunk.toString("utf8");

    let idx;

    while ((idx = this.buf.indexOf("\n")) >= 0) {

      const line = this.buf.slice(0, idx).trim();
      this.buf = this.buf.slice(idx + 1);

      if (!line) continue;

      try {

        const parsed = parseFrame(line);

        this.onData?.(parsed);

      } catch (e) {

        if (e instanceof ParseError) {

          if (this.verbose) {
            console.warn("Corrupt frame:", e.message, "raw:", line);
          }

        } else {

          this._fail(e);

        }
      }
    }

    if (this.buf.length > 4096) this.buf = "";
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }

  _closed() {

    if (!this.connected) return;

    this.connected = false;
    this.port = null;
    this.buf = "";

    this.onDisconnected?.("serial closed (event)");
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

**Sketch.js**
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

        this.c = color(181, 157, 0);
        this.lineSize = 100;
        this.angle = 0;
        this.clickPosX = 0;
        this.clickPosY = 0;

        this.rxData = {
            x: 0,
            y: 0,
            btnA: false,
            btnB: false,
            prevA: false,
            prevB: false,
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
            strokeWeight(0.75);
            background(255);
            console.log("Microbit ready to draw");
            this.rxData = {
                x: 0,
                y: 0,
                btnA: false,
                btnB: false,
                prevA: false,
                prevB: false,
                ready: false
            };
        }

        else if (ev.type === EVENTS.DISCONNECT) {
            this.transitionTo(this.estado_esperando);
        }

        else if (ev.type === EVENTS.DATA) {
            this.updateLogic(ev.payload);
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
        this.rxData.x = map(data.x,-2048,2047,0,width);
        this.rxData.y = map(data.y,-2048,2047,0,height);
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

    renderer.set(painter.estado_corriendo, drawRunning);
}

function draw() {
    painter.update();
    renderer.get(painter.state)?.();
}

function drawRunning() {
    let mb = painter.rxData;

    if (!mb.ready) return;

    // equivalente a mouseIsPressed (botón A)
    if (mb.btnA) {
        push();
        translate(width / 2, height / 2);

        // equivalente a mouseY (resolución del círculo)
        let circleResolution = int(map(mb.y + 100, 0, height, 2, 10));

        // equivalente a mouseX (radio)
        let radius = mb.x - width / 2;

        let angle = TWO_PI / circleResolution;

        // equivalente a keyIsPressed (botón B activa relleno)
        if (mb.btnB) {
            fill(34, 45, 122, 50);
        } else {
            noFill();
        }

        stroke(0, 25);
        strokeWeight(2);

        beginShape();
        for (let i = 0; i <= circleResolution; i++) {
            let x = cos(angle * i) * radius;
            let y = sin(angle * i) * radius;
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
**bridgeserver**
``` js
//   Uso:
//     node bridgeServer.js --device sim --wsPort 8081 --hz 30
//     node bridgeServer.js --device microbit --wsPort 8081 --serialPort COM5 --baud 115200

//   WS contract:
//    * bridge To client:
//        {type:"status", state:"ready|connected|disconnected|error", detail:"..."}
//        {type:"microbit", x:int, y:int, btnA:bool, btnB:bool, t:ms}
//    * client To bridge:
//        {cmd:"connect"} | {cmd:"disconnect"}
//        {cmd:"setSimHz", hz:30}
//        {cmd:"setLed", x:2, y:3, value:9}


const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");
const SimAdapter = require("./adapters/SimAdapter");
const MicrobitASCIIAdapter = require("./adapters/MicrobitASCIIAdapter");
const MicrobitV2Adapter = require("./adapters/MicrobitV2Adapter");
// const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");

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

function hasFlag(name) {
  return process.argv.includes(`--${name}`);
}

function nowMs() { return Date.now(); }

function safeJsonParse(s) {
  try {
    return JSON.parse(s);

  } catch (e) {
    log.warn("Failed to parse JSON: ", s, e);
    return null;
  }
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
  const microbit = ports.find(p =>
    p.vendorId && parseInt(p.vendorId, 16) === 0x0D28
  );
  return microbit?.path ?? null;
}

async function createAdapter() {
  if (DEVICE === "microbit2") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit found at ${path}`);
    return new MicrobitV2Adapter({ path, baud: BAUD, verbose: VERBOSE });
  }

    if (DEVICE === "microbit") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit found at ${path}`);
    return new MicrobitASCIIAdapter({ path, baud: BAUD, verbose: VERBOSE });
  }

  // if (DEVICE === "microbit-bin") {
  //   const path = SERIAL_PATH ?? await findMicrobitPort();
  //   if (!path) {
  //     log.error("micro:bit not found. Use --serialPort to specify manually.");
  //     process.exit(1);
  //   }
  //   return new MicrobitBinaryAdapter({ path, baud: BAUD });
  // }

  return new SimAdapter({ hz: SIM_HZ });
}

async function main() {
  const wss = new WebSocketServer({ port: WS_PORT });
  log.info(`WS listening on ws://127.0.0.1:${WS_PORT} device=${DEVICE}`);

  const adapter = await createAdapter();

  adapter.onConnected = (detail) => {
    log.info(`[ADAPTER] Device Connected: ${detail}`);
    status(wss, "connected", detail);
  };

  adapter.onDisconnected = (detail) => {
    log.warn(`[ADAPTER] Device Disconnected: ${detail}`);
    status(wss, "disconnected", detail);
  };

  adapter.onError = (detail) => {
    log.error(`[ADAPTER] Device Error: ${detail}`);
    status(wss, "error", detail);
  };

  adapter.onData = (d) => {
    broadcast(wss, {
      type: "microbit",
      x: d.x,
      y: d.y,
      btnA: !!d.btnA,
      btnB: !!d.btnB,
      t: nowMs(),
    });
  };

  status(wss, "ready", `bridge up (${DEVICE})`);

  wss.on("connection", (ws, req) => {
    log.info(`[NETWORK] Remote Client connected from ${req.socket.remoteAddress}. Total clients: ${wss.clients.size}`);

    const state = adapter.connected ? "connected" : "ready";

    const detail = adapter.connected
      ? adapter.getConnectionDetail()
      : `bridge (${DEVICE})`;

    ws.send(JSON.stringify({ type: "status", state, detail, t: nowMs() }));

    ws.on("message", async (raw) => {
      const msg = safeJsonParse(raw.toString("utf8"));
      if (!msg) return;

      if (msg.cmd === "connect") {
        log.info(`[NETWORK] Client requested adapter connect`);

        if (adapter.connected) {
          log.info(`[HW-POLICY] Adapter already open. Sending current status to incoming client.`);
          ws.send(JSON.stringify({ type: "status", state: "connected", detail: adapter.getConnectionDetail(), t: nowMs() }));
          return;
        }
        
        try {
          await adapter.connect();
        } catch (e) {
          const detail = `connect failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
          status(wss, "error", detail);
        }
        return;
      }

      if (msg.cmd === "disconnect") {
        log.info(`[NETWORK] Client requested adapter disconnect`);
        if (wss.clients.size > 1) {
          log.info(`[HW-POLICY] Adapater kept open. Shared with ${wss.clients.size - 1} other active client(s).`);
          ws.send(JSON.stringify({ type: "status", state: "disconnected", detail: "logical disconnect only", t: nowMs() }));
          return;
        }
        
        try {
          await adapter.disconnect();
        } catch (e) {
          const detail = `disconnect failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
          status(wss, "error", detail);
        }
        return;
      }

      if (msg.cmd === "setSimHz" && adapter instanceof SimAdapter) {
        log.info(`Setting Sim Hz to ${msg.hz}`);
        await adapter.handleCommand(msg);
        status(wss, "connected", `sim hz=${adapter.hz}`);
        return;
      }

      if (msg.cmd === "setLed") {
        try {
          await adapter.handleCommand?.(msg);
        } catch (e) {
          const detail = `command failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
          status(wss, "error", detail);
        }
        return;
      }
    });

    ws.on("close", () => {
      log.info(`[NETWORK] Remote Client disconnected. Total clients left: ${wss.clients.size}`);
      if (wss.clients.size === 0) {
        log.info("[HW-POLICY] No more remote clients. Auto-disconnecting adapter device to free resources...");
        adapter.disconnect();
      }
    });
  });

  if (DEVICE === "sim") {
    await adapter.connect();
  }
}

main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
});
```
**Micro:Bit** 
```
from microbit import *
import time

uart.init(baudrate=115200)

while True:
    t = time.ticks_ms()
    x = accelerometer.get_x()
    y = accelerometer.get_y()
    
    # Convertimos los booleanos (True/False) a enteros (1/0) como pide la guía
    a = 1 if button_a.is_pressed() else 0
    b = 1 if button_b.is_pressed() else 0
    
    # Calculamos el Checksum sumando los valores absolutos
    chk = abs(x) + abs(y) + a + b
    
    # Formateamos la trama exactamente como exige la Actividad 02
    trama = "$T:{}|X:{}|Y:{}|A:{}|B:{}|CHK:{}\n".format(t, x, y, a, b, chk)
    
    uart.write(trama)
    sleep(100) # Frecuencia de 10 Hz
```
## Bitácora de reflexión
### Actividad 03 
<img width="1791" height="402" alt="image" src="https://github.com/user-attachments/assets/74376d76-7c83-4b20-aad1-9b49eab412e2" />
