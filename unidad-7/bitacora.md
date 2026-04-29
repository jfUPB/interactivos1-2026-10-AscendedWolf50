# Unidad 7

## Bitácora de proceso de aprendizaje
### Actividad 01
##### ¿Qué diferencia hay entre un evento musical y un mensaje de control?**
Un evento musical es un suceso puntual atado al tiempo (como una nota). Un mensaje de control modifica el estado o las propiedades del sistema (como cambiar un color o ajustar el volumen) y es independiente del ritmo.

##### ¿Qué quiere decir que un parámetro del sistema sea persistente?
Significa que el sistema tiene memoria para guardar un valor. El último color recibido se mantiene constante en la pantalla hasta que el sistema reciba un nuevo mensaje ordenando otro cambio.

##### ¿Qué partes del sistema de la unidad 6 permanecen intactas en este nuevo caso?
**La parte visual:** La forma base en que se dibujan los gráficos sigue igual, solo se adaptó para leer el dato externo del color.

**El servidor (bridgeServer.js):** Sigue funcionando como intermediario enviando datos al navegador a través de WebSockets.

**La estructura:** La aplicacion visual se mantiene aislada; sigue sin estar conectada a Open Stage Control porque solo se comunica con el servidor puente.

#### Paso 1
##### Si Open Stage Control fuera “el dispositivo” de esta unidad, ¿Cuál sería su protocolo?
El protocolo que utiliza Open Stage Control es OSC (Open Sound Control), transmitido a través de la red (como se mencionó en la actividad anterior, usando UDP).

##### ¿Qué parte de ese protocolo te interesa conservar y cuál te gustaría normalizar?
**Conservar:** La dirección (address, que indica que parametro modificar) y los valores exactos (args, que indican cual es el nuevo valor).

**Normalizar:** El envoltorio y el transporte. Necesitamos traducir el paquete de OSC y UDP a un formato estandar de la web (como un JSON), para que la aplicacion visual lo procese directamente sin necesidad de entender la estructura tecnica de OSC.

#### Paso 2

##### ¿Por qué no conviene procesar un mensaje OSC igual que un mensaje de Strudel?
Porque cumplen funciones distintas. Strudel envia eventos ritmicos que ocurren en un instante exacto. OSC envia cambios de parámetros que deben mantenerse constantes hasta recibir una nueva orden.

##### ¿Qué variables del sistema deberían vivir como estado persistente y no como evento efímero?
Cualquier valor que defina la apariencia o comportamiento base del sistema. Por ejemplo: colores, posicin de la camara.

#### Paso 3

##### ¿Qué componentes de la arquitectura necesitas conservar obligatoriamente?
El servidor puente, la capa de renderizado y el Adapter encargado de traducir los datos.

##### ¿Qué nuevas estructuras de estado necesitas introducir para soportar control paramétrico?
Un objeto o estado global. Debe almacenar en memoria los ultimos valores de control recibidos para que el motor de renderizado los lea continuamente.







## Bitácora de aplicación 
### Actividad 02
**OSCAdapter.js**
```js
// adapters/OscAdapter.js

//node bridgeServer.js --device strudel
const BaseAdapter = require("./BaseAdapter");
const { Server } = require("node-osc");

class OscAdapter extends BaseAdapter {
  constructor({ port = 8000 } = {}) {
    super();
    this.port = port;
    this.oscServer = null;
  }

  async connect() {
    if (this.connected) return;

    return new Promise((resolve, reject) => {
      try {
        this.oscServer = new Server(this.port, '0.0.0.0', () => {
          this.connected = true;
          this.onConnected?.(`OSC Server escuchando en el puerto UDP ${this.port}`);
          resolve();
        });

        this.oscServer.on('message', (msg) => {
          this._onMessage(msg);
        });

      } catch (e) {
        this.onError?.(e.message);
        reject(e);
      }
    });
  }

  async disconnect() {
    if (!this.connected) return;
    this.connected = false;
    if (this.oscServer) {
      this.oscServer.close();
      this.oscServer = null;
    }
    this.onDisconnected?.("OSC Server cerrado");
  }

  getConnectionDetail() {
    return `OSC UDP open (Port ${this.port})`;
  }

  _onMessage(msg) {
    console.log("OSC LLEGANDO:", msg);
    // msg suele ser un arreglo: [address, arg1, arg2, ...]
    const address = msg[0];
    const args = msg.slice(1);

    // Normalizamos el mensaje según el contrato sugerido
    const normalizedMsg = {
      type: "osc",
      payload: {
        address: address,
        args: args
      }
    };

    // Emitimos los datos usando el método heredado de BaseAdapter
    this.onData?.(normalizedMsg);
  }
}

module.exports = OscAdapter;
```
**bridgeServer.js**
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
//        {type:"osc", payload:{ address:string, args:array }}
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

  // === INTEGRACIÓN OSC (UNIDAD 7) ===
  // Si estamos en modo strudel, también levantamos el adapter de OSC
  let oscAdapter = null;
  if (DEVICE === "strudel") {
    oscAdapter = new OscAdapter({ port: 8000 });
    
    oscAdapter.onConnected = (detail) => log.info(`[OSC ADAPTER] Connected: ${detail}`);
    oscAdapter.onDisconnected = (detail) => log.warn(`[OSC ADAPTER] Disconnected: ${detail}`);
    oscAdapter.onError = (detail) => log.error(`[OSC ADAPTER] Error: ${detail}`);
    
    oscAdapter.onData = (d) => {
      // El OscAdapter ya normaliza el formato a { type: "osc", payload: {...} }
      broadcast(wss, d);
    };
  }
  // ===================================

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
    // Clasificamos el payload según el dispositivo principal seleccionado
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
          // Conectar OSC si existe
          if (oscAdapter) await oscAdapter.connect();
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
          log.info(`[HW-POLICY] Adapter kept open. Shared with ${wss.clients.size - 1} other active client(s).`);
          ws.send(JSON.stringify({ type: "status", state: "disconnected", detail: "logical disconnect only", t: nowMs() }));
          return;
        }
        
        try {
          await adapter.disconnect();
          // Desconectar OSC si existe
          if (oscAdapter) await oscAdapter.disconnect();
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
        if (oscAdapter) oscAdapter.disconnect();
      }
    });
  });

  // Strudel levanta su propio servidor WS, por lo que lo iniciamos automáticamente
  // Al igual que el de OSC
  if (DEVICE === "sim" || DEVICE === "strudel") {
    await adapter.connect();
    if (oscAdapter) await oscAdapter.connect();
  }
}

main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
});

```
**bridgeClient.js**
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

      if (msg.type === "microbit" || msg.type === "strudel" || msg.type === "osc") {
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
**sketch.js**
``` js
const EVENTS = {
    CONNECT: "CONNECT",
    DISCONNECT: "DISCONNECT",
    DATA: "DATA",
    STRUDEL_DATA: "STRUDEL_DATA",
    OSC_DATA: "OSC_DATA", // Nuevo evento para el control de interfaz
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

        // NUEVO: Estado persistente para Open Stage Control
        this.persistentState = {
            colorRGB: null,     // Control 1: Reemplaza el color base si no es nulo
            globalScale: 1.0,   // Control 2: Multiplicador de tamaño
            glitchMode: false   // Control 3: Alteración de coordenadas de dibujo
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

        // NUEVO: Procesar los datos persistentes de OSC
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

    // NUEVO: Lógica de actualización de variables de estado de OSC
    updateOscLogic(payload) {
        switch(payload.address) {
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
        // Enrutador de datos según la fuente
        if (data.type === "strudel") {
            painter.postEvent({ type: EVENTS.STRUDEL_DATA, payload: data });
        } else if (data.type === "osc") { // NUEVO: Enrutamiento OSC
            painter.postEvent({ type: EVENTS.OSC_DATA, payload: data.payload });
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
            // Se inyecta el estado persistente a la función de dibujo
            dibujarElemento(anim, progress, painter.persistentState);
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

// ==========================================
// FUNCIONES DE DIBUJO STRUDEL
// ==========================================

// Función auxiliar para aplicar el modo glitch a las coordenadas
function aplicarGlitch(x, y, isGlitch) {
    if (isGlitch) {
        return { x: x + random(-20, 20), y: y + random(-20, 20) };
    }
    return { x, y };
}

function dibujarElemento(anim, p, estadoPersistente) {
    push();
    // Reemplaza el color original del instrumento si Open Stage Control envió uno
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

##### Cómo configuraste Open Stage Control;
Se configuro el envío de datos (el campo "send") hacia la direccion local y el puerto del adaptador: 127.0.0.1:8000 mediante protocolo UDP. Ademas, se cambio el puerto local de la interfaz a 8005 para evitar conflictos de red con el servidor de strudel.

##### Qué widgets decidiste usar y por qué;
**Color (RGB):** Para enviar arreglos de valores y modificar la paleta de colores.

**Fader:** Para enviar valores de punto flotante y alterar la escala global de los visuales

**Botón:** Para enviar booleanos (0 o 1) y activar/desactivar el modo glitch.
  

##### Qué estructura final de mensaje decidiste usar para los controles;
Se utilizó un objeto JSON normalizado desde el adaptador en el servidor, con la siguiente estructura:
```
{ type: "osc", payload: { address: "/nombre_del_control", args: [valores] } }
```


##### Cómo conectaste bridgeClient.js, FSMTask, updateLogic y drawRunning;
Se modifico bridgeClient.js para permitir el paso de mensajes tipo "osc". Estos se envían a FSMTask, donde un nuevo evento (OSC_DATA) dirige la información a la función updateOscLogic para sobreescribir el objeto persistentState. Finalmente, drawRunning lee este estado para aplicar los valores a las funciones de dibujo.

##### Cómo integraste ambas fuentes de datos en el mismo frontend;
Se separó el concepto de tiempo y estado. Los mensajes de Strudel se almacenan en una cola de eventos (eventQueue) para disparar animaciones cortas, mientras que los mensajes de OSC modifican variables (persistentState).

##### Qué pruebas hiciste para verificar que el control paramétrico funciona sin romper la sincronización de Strudel;
Se ejecuto un patron ritmico en strudel mientras yo manipulaba los controles de OSC (slider, color, toggle) de forma simultanea. Comprobe visualmente que las figuras cambiaban sus atributos en tiempo real con fluidez.

##### Qué problemas encontraste y cómo los solucionaste.
**Conflicto de puertos:** Open Stage Control y Strudel intentaban ocupar el puerto 8080 al mismo tiempo. Se solucionó asignando el puerto 8005 a la interfaz gráfica.

**Bloqueo en el cliente web:** El frontend no reconocia la informacion. Se solucionó agregando la validación explicita para mensajes tipo "osc" en bridgeClient.js.
## Bitácora de reflexión
