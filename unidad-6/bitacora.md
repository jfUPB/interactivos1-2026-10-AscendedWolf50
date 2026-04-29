# Unidad 6

## Bitácora de proceso de aprendizaje
### Actividad 01
**1. Diferencia entre recibir un mensaje y ejecutarlo:**
Recibir es el acto físico de que el dato llegue por la red (capa de transporte). Ejecutar es activar la animación en la pantalla en el momento exacto en que debe ocurrir (capa de scheduling o estado).

**2. Necesidad del timestamp en sistemas audiovisuales:**
Garantiza la sincronización perfecta. Si hay lag o latencia en la red, el timestamp le dice al sistema: "No me dibujes cuando llegué, dibújame exactamente en este milisegundo para coincidir con el audio".

**3. Aspectos intactos de la arquitectura (U4 y U5):**
Se mantiene la separación de responsabilidades: sigue existiendo una fuente de datos externa, un Adapter que traduce esos datos, un Bridge que los transporta y un Frontend que solo lee el estado limpio para dibujar.

**4. El protocolo de Strudel como "dispositivo":**
Su protocolo consiste en enviar mensajes empaquetados por WebSocket, conteniendo una dirección (ej. /dirt/play), un arreglo de argumentos (sonido, ciclos, efectos) y una marca de tiempo absoluta.

**5. Variables mínimas a extraer para la visualización:**
Se necesitan tres básicas: s (el tipo de sonido, para decidir la forma o color), delta o duración (para el ciclo de vida de la animación), y el timestamp (para saber el instante exacto de inicio).

**6. Problema que resuelve la cola de eventos:**
Resuelve la latencia y la inestabilidad de la red. Almacena las "notas futuras" en orden, evitando que un pico de lag haga que todas las animaciones se disparen a destiempo o amontonadas.

**7. Por qué la cola no pertenece al bridge:**
Porque el Bridge debe ser un simple mensajero (agnóstico a lo que transporta). El manejo del tiempo y el momento de dibujar dependen estrictamente del reloj interno del Frontend que es quien "interpreta" la partitura.

**8. Papel del Adapter en las Unidades 4 y 5:**
Su función era actuar como traductor: tomaba datos "sucios" o crudos del hardware (cadenas ASCII o tramas de bytes) y los convertía en variables limpias para el sistema.

**9. Adapter necesario para Strudel:**
Un Adapter que intercepte el mensaje JSON complejo de Strudel, busque dentro de su lista plana de args y construya un objeto de datos estandarizado y fácil de leer para el Frontend.

## Bitácora de aplicación 
### Actividad 02
**StrudelAdapter**
```js
const BaseAdapter = require("./BaseAdapter");
const WebSocket = require('ws');

class StrudelAdapter extends BaseAdapter {
  constructor({ port = 8080 } = {}) {
    super();
    this.port = port;
    this.wss = null;
  }

  async connect() {
    if (this.connected) return;

    return new Promise((resolve, reject) => {
      try {
        // Levantamos un WebSocket Server específicamente para Strudel
        this.wss = new WebSocket.Server({ port: this.port }, () => {
          this.connected = true;
          this.onConnected?.(`Strudel WebSocket escuchando en el puerto ${this.port}`);
          resolve();
        });

        this.wss.on('connection', (ws) => {
          ws.on('message', (message) => this._onMessage(message));
        });

        this.wss.on('error', (err) => {
          this.onError?.(err.message);
          reject(err);
        });
      } catch (e) {
        reject(e);
      }
    });
  }

  async disconnect() {
    if (!this.connected) return;
    this.connected = false;
    if (this.wss) {
      this.wss.close();
      this.wss = null;
    }
    this.onDisconnected?.("Strudel WebSocket cerrado");
  }

  getConnectionDetail() {
    return `Strudel WS open (Port ${this.port})`;
  }

  _onMessage(message) {
    try {
      const rawMsg = JSON.parse(message);
      let params = {};
      
      // Parsear los argumentos planos de Strudel
      if (rawMsg.args && Array.isArray(rawMsg.args)) {
        for (let i = 0; i < rawMsg.args.length; i += 2) {
          params[rawMsg.args[i]] = rawMsg.args[i+1];
        }
      }
      
      // Emitimos usando el método de BaseAdapter
      // bridgeServer.js le añadirá automáticamente la propiedad {type: "strudel"}
      this.onData?.({
        strudelTimestamp: rawMsg.timestamp,
        s: params.s || "unknown",
        delta: params.delta || 0.25
      });
    } catch (e) {
      // Si el mensaje no es JSON válido o es ruido, se ignora silenciosamente
    }
  }
}

module.exports = StrudelAdapter;
```
 **bridgeServer**
```js
//   Uso:
//     node bridgeServer.js --device sim --wsPort 8081 --hz 30
//     node bridgeServer.js --device microbit --wsPort 8081 --serialPort COM5 --baud 115200
//    node bridgeServer.js --device microbit --wsPort 8081
//   node bridgeServer.js --device microbit-v2 --wsPort 8081
//    node bridgeServer.js --device microbit-bin --wsPort 8081
//    node bridgeServer.js --device strudel
//   WS contract:
//    * bridge To client:
//        {type:"status", state:"ready|connected|disconnected|error", detail:"..."}
//        {type:"microbit", x:int, y:int, btnA:bool, btnB:bool, t:ms}
//        {type:"strudel", timestamp:ms, payload:{ s:string, delta:float }}
//    * client To bridge:
//        {cmd:"connect"} | {cmd:"disconnect"}
//        {cmd:"setSimHz", hz:30}
//        {cmd:"setLed", x:2, y:3, value:9}


const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");
const SimAdapter = require("./adapters/SimAdapter");
const MicrobitAsciiAdapter = require("./adapters/MicrobitASCIIAdapter");
const MicrobitV2Adapter = require("./adapters/MicrobitV2Adapter");
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");
const StrudelAdapter = require("./adapters/StrudelAdapter");

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
  if (DEVICE === "strudel") {
    log.info(`Strudel listener starting on port 8080`);
    return new StrudelAdapter({ port: 8080 });
  }

  if (DEVICE === "microbit") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit found at ${path}`);
    return new MicrobitAsciiAdapter({ path, baud: BAUD, verbose: VERBOSE });
  }

  if (DEVICE === "microbit-v2") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit found at ${path} (ASCII v2 Mode)`);
    return new MicrobitV2Adapter({ path, baud: BAUD, verbose: VERBOSE });
  }

   if (DEVICE === "microbit-bin") {
     const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
     log.error("micro:bit not found. Use --serialPort to specify manually.");
     process.exit(1);
    }
    return new MicrobitBinaryAdapter({ path, baud: BAUD });
   }

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
    // Clasificamos el payload según el dispositivo seleccionado
    if (DEVICE === "strudel") {
      broadcast(wss, {
        type: "strudel",
        timestamp: d.strudelTimestamp,
        payload: {
          s: d.s,
          delta: d.delta
        }
      });
    } else {
      broadcast(wss, {
        type: "microbit",
        x: d.x,
        y: d.y,
        btnA: !!d.btnA,
        btnB: !!d.btnB,
        t: nowMs(),
      });
    }
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

  // Strudel levanta su propio servidor WS, por lo que lo iniciamos automáticamente
  if (DEVICE === "sim" || DEVICE === "strudel") {
    await adapter.connect();
  }
}

main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
});
```
**bridgeClient**
```js
class BridgeClient {
  constructor(url = "ws://127.0.0.1:8081") {
    this._url = url;
    this._ws = null;
    this._isOpen = false;

    this._onData = null;
    this._onConnect = null;
    this._onDisconnect = null;
    this._onStatus = null;
  }

  get isOpen() {
    return this._isOpen;
  }

  onData(callback) { this._onData = callback; }
  onConnect(callback) { this._onConnect = callback; }
  onDisconnect(callback) { this._onDisconnect = callback; }
  onStatus(callback) { this._onStatus = callback; }

  open() {
    if (this._ws && this._ws.readyState === WebSocket.OPEN) {
      if (!this._isOpen) this.send({ cmd: "connect" });
      return;
    }

    if (this._ws) {
      this.close();
    }

    this._ws = new WebSocket(this._url);

    this._ws.onopen = () => {
      this.send({ cmd: "connect" });
    };

    this._ws.onmessage = (event) => {
      // Esperamos JSON normalizado desde el bridge
      let msg;
      try {
        msg = JSON.parse(event.data);
      } catch (e) {
        console.warn("WS message is not JSON:", event.data);
        return;
      }

      // Convención mínima:
      // - {type:"status", state:"...", detail:"..."}
      // - {type:"microbit", x:..., y:..., btnA:..., btnB:...}
      if (msg.type === "status") {
        this._onStatus?.(msg);

        if (msg.state === "connected") {
          this._isOpen = true;
          this._onConnect?.();
        }

        if (msg.state === "disconnected" || msg.state === "error" || msg.state === "ready") {
          this._isOpen = false; 
          this._onDisconnect?.();
          if (msg.state === "error") {
            this._ws?.close();
            this._ws = null;
          }          
        }
        return;
      }

      if (msg.type === "microbit" || msg.type === "strudel") {
        // payload ya normalizado
        this._onData?.(msg);
        return;
      }
    };

    this._ws.onerror = (err) => {
      console.warn("WS error:", err);
    };

    this._ws.onclose = () => {
      this._handleDisconnect();
    };
  }

  close() {
    if (!this._ws || this._ws.readyState !== WebSocket.OPEN) return;

    try {
      this.send({ cmd: "disconnect" });
      this._isOpen = false;
    } catch (e) {
      console.warn("Failed to send disconnect command:", e);
    }
  }

  send(obj) {
    if (!this._ws || this._ws.readyState !== WebSocket.OPEN) return;
    this._ws.send(JSON.stringify(obj));
  }

  _handleDisconnect() {
    this._isOpen = false;
    this._ws = null;
    this._onDisconnect?.();
  }
}

```
**sketch**
```js
const EVENTS = {
    CONNECT: "CONNECT",
    DISCONNECT: "DISCONNECT",
    DATA: "DATA",
    STRUDEL_DATA: "STRUDEL_DATA",
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
            console.log("System ready to draw (Microbit + Strudel)");
            
            // Reiniciar estado
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
            // Encolar el evento musical con su timestamp
            this.eventQueue.push({
                timestamp: ev.payload.timestamp,
                sound: ev.payload.payload.s,
                delta: ev.payload.payload.delta
            });
            // Ordenar para mantener la sincronización si llegan desordenados
            this.eventQueue.sort((a, b) => a.timestamp - b.timestamp);
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
        // Enrutador de datos según la fuente
        if (data.type === "strudel") {
            painter.postEvent({ type: EVENTS.STRUDEL_DATA, payload: data });
        } else {
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
    // Fondo con alfa para crear estela de las animaciones
    background(255, 40);

    // ==========================================
    // 1. RENDERIZADO STRUDEL (Scheduling)
    // ==========================================
    let now = Date.now() + painter.latencyCorrection;

    // A. Extraer eventos cuyo tiempo ya se cumplió
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

    // B. Dibujar animaciones activas
    for (let i = painter.activeAnimations.length - 1; i >= 0; i--) {
        let anim = painter.activeAnimations[i];
        let elapsed = now - anim.startTime;
        let progress = elapsed / anim.duration;

        if (progress <= 1.0) {
            dibujarElemento(anim, progress);
        } else {
            painter.activeAnimations.splice(i, 1);
        }
    }

    // ==========================================
    // 2. RENDERIZADO MICRO:BIT
    // ==========================================
    let mb = painter.rxData;
    if (!mb.ready) return;

    if (mb.btnA) {
        let x = mb.x;
        let y = mb.y;
        push();
        translate(x, y);
        rotate(radians(painter.angle));
        stroke(painter.c);
        strokeWeight(1.5);
        line(0, 0, painter.lineSize, painter.lineSize);
        painter.angle += 1;
        pop();
    }
}

// ==========================================
// FUNCIONES DE DIBUJO STRUDEL
// ==========================================
function dibujarElemento(anim, p) {
    push();
    const c = anim.c;
    
    switch (anim.type) {
        case 'tr909bd': dibujarBombo(anim, p, c); break;
        case 'tr909sd': dibujarCaja(anim, p, c); break;
        case 'tr909hh': 
        case 'tr909oh': dibujarHat(anim, p, c); break;
        default: dibujarDefault(anim, p, c); break;
    }
    pop();
}

function dibujarBombo(anim, p, c) {
    let d = lerp(100, 600, p);
    let alpha = lerp(255, 0, p);
    fill(c[0], c[1], c[2], alpha);
    noStroke();
    circle(anim.x, anim.y, d);
}

function dibujarCaja(anim, p, c) {
    let w = lerp(width, 0, p);
    let alpha = lerp(255, 0, p);
    fill(c[0], c[1], c[2], alpha);
    noStroke();
    rect(anim.x, anim.y, w, 50);
}

function dibujarHat(anim, p, c) {
    let sz = lerp(40, 0, p);
    fill(c[0], c[1], c[2]);
    noStroke();
    rect(anim.x, anim.y, sz, sz);
}

function dibujarDefault(anim, p, c) {
    let size = lerp(100, 0, p);
    let angle = p * TWO_PI;
    translate(anim.x, anim.y);
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
**Index**
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Painter – Web Serial API</title>
  <link rel="stylesheet" href="style.css" />
  <script src="https://cdn.jsdelivr.net/npm/p5@1.11.11/lib/p5.js"></script>
  <script src="fsm.js"></script>
  <script src="bridgeClient.js"></script>
  <script src="sketch.js"></script>
</head>
<body>
</body>
</html>

```
#### Documenta en tu bitacora
**1. Cómo configuraste Strudel para emitir eventos:**
Usé la función .osc() al final del patrón musical. La dejé vacía (pat.osc()) para evitar errores de sintaxis en Strudel, aprovechando que por defecto envía los datos al puerto 8080 del localhost.

**2. Qué estructura final de mensaje decidiste usar:**
Un JSON estandarizado con tres claves principales: type (origen), timestamp (tiempo de ejecución) y payload (datos útiles como el sonido s y la duración delta).

**3. Cómo conectaste bridgeClient.js, FSMTask, updateLogic y drawRunning:**
bridgeClient recibe el JSON y lo envía a la cola. updateLogic revisa si el Date.now() ya alcanzó el timestamp del evento y lo mueve a la lista de animaciones activas. Finalmente, drawRunning lee esa lista y dibuja.

**4. Cómo separaste recepción, cola temporal y renderizado:**
Asignando responsabilidad única: bridgeClient.js solo maneja red; updateLogic es el único que maneja arreglos y calcula el tiempo; y las funciones de dibujo ignoran la red/tiempo y solo pintan lo que reciben.

**5. Qué pruebas hiciste para verificar la sincronización:**
Ejecuté un ritmo básico de Strudel (bombo y caja) y observé si el destello visual coincidía con el ataque del sonido. Usé la variable LATENCY_CORRECTION para ajustar cualquier pequeño desfase de audio/video.

**6. Qué problemas encontraste y cómo los solucionaste:**

Problema: El sketch.js original mezclaba red, tiempo y dibujo. Solución: Refactorizar el código separándolo en cliente, lógica de estado y renderizado.

Problema: Strudel daba error al leer la URL "ws://localhost:8080". Solución: Usar .osc() vacío para que use la ruta por defecto.



## Bitácora de reflexión
<img width="603" height="1145" alt="image" src="https://github.com/user-attachments/assets/4b0b9579-6daa-4e52-9690-2d1a3c3c89a3" />

