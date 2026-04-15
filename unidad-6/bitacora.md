# Unidad 6

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 
### Actividad 02
**StrudelAdapter**
```js
// StrudelAdapter.js

/**
 * Normaliza los mensajes crudos que llegan desde Strudel.
 * Strudel envía los argumentos como un arreglo plano: ['cps', 0.5, 's', 'tr909bd', 'delta', 0.25]
 */
function normalizeStrudelEvent(rawMessage) {
    let params = {};
    
    // Extraemos los pares clave-valor del arreglo 'args'
    if (rawMessage.args && Array.isArray(rawMessage.args)) {
        for (let i = 0; i < rawMessage.args.length; i += 2) {
            params[rawMessage.args[i]] = rawMessage.args[i+1];
        }
    }

    // Retornamos el contrato estandarizado que el frontend espera
    return {
        type: "strudel",
        timestamp: rawMessage.timestamp,
        payload: {
            eventType: "noteEvent",
            s: params.s || "unknown", // El tipo de sonido (ej. tr909bd)
            delta: params.delta || 0.25 // Duración rítmica
        }
    };
}

module.exports = { normalizeStrudelEvent };
```
 **bridgeServer**
```js
// bridgeServer.js
const WebSocket = require('ws');
const { normalizeStrudelEvent } = require('./StrudelAdapter'); // Importamos el adapter

// 1. CONFIGURACIÓN
const STRUDEL_PORT = 8080;   // Donde Strudel envía los datos
const P5JS_PORT = 8081;      // Donde p5.js escuchará

// 2. SERVIDORES WEBSOCKET
const wssStrudel = new WebSocket.Server({ port: STRUDEL_PORT });
const wssP5 = new WebSocket.Server({ port: P5JS_PORT });

console.log(`[SERVER] Escuchando a Strudel en ws://localhost:${STRUDEL_PORT}`);
console.log(`[SERVER] Transmitiendo al Frontend en ws://localhost:${P5JS_PORT}`);

// 3. RECEPCIÓN Y NORMALIZACIÓN
wssStrudel.on('connection', (ws) => {
    console.log('[STRUDEL] Conectado al Bridge');

    ws.on('message', (message) => {
        try {
            // 1. Parsear el mensaje crudo
            const rawMsg = JSON.parse(message);
            
            // 2. Pasar el mensaje por el Adapter
            const normalizedMsg = normalizeStrudelEvent(rawMsg);
            
            // 3. Preparar el payload final (contrato estable)
            const payload = JSON.stringify(normalizedMsg);
            
            // 4. Hacer broadcast a todos los clientes web (p5.js)
            wssP5.clients.forEach(client => {
                if (client.readyState === WebSocket.OPEN) {
                    client.send(payload);
                }
            });

        } catch (e) {
            console.error('[ERROR] Procesando mensaje de Strudel:', e);
        }
    });
});

wssP5.on('connection', (ws) => {
    console.log('[FRONTEND] p5.js se ha conectado al Bridge');
});
```
**bridgeClient**
```js
// bridgeClient.js

// Conectamos al puerto 8081 que configuramos en bridgeServer.js
const socket = new WebSocket('ws://localhost:8081');

socket.onopen = () => {
    console.log('[CLIENTE] Conectado al Bridge (Frontend)');
};

socket.onmessage = (event) => {
    const msg = JSON.parse(event.data);

    // Verificamos que sea un mensaje de nuestra fuente Strudel
    if (msg.type === "strudel") {
        
        // ¡Magia! Como ya lo normalizamos en el servidor, no hay que hacer bucles raros.
        // Simplemente pasamos el mensaje completo a nuestra capa de estado.
        // (Asegúrate de que esta función exista y esté accesible en tu sistema)
        encolarEventoStrudel(msg); 
    }
};

socket.onerror = (error) => {
    console.error('[CLIENTE] Error en WebSocket:', error);
};
```
**sketch**
```js
let eventQueue = [];
let activeAnimations = []; 
const LATENCY_CORRECTION = 0; 

function setup() {
  createCanvas(windowWidth, windowHeight);
  rectMode(CENTER);
  noStroke();

  // ¡OJO! Ya no hay WebSocket aquí. 
  // La conexión la maneja bridgeClient.js de forma independiente.
}

// ==========================================
// 1. RECEPCIÓN DE DATOS (Llamado por bridgeClient.js)
// ==========================================
function encolarEventoStrudel(msg) {
  // Como tu StrudelAdapter.js ya normalizó el mensaje, 
  // aquí entra limpiecito (msg.payload).
  eventQueue.push({ 
      timestamp: msg.timestamp, 
      sound: msg.payload.s,
      delta: msg.payload.delta
  });

  // Ordenamos para asegurar que los eventos se ejecuten en el orden correcto
  eventQueue.sort((a, b) => a.timestamp - b.timestamp);
}

// ==========================================
// 2. SCHEDULING / ESTADO (Tu updateLogic)
// ==========================================
function updateLogic() {
  let now = Date.now() + LATENCY_CORRECTION;

  // Revisamos si el tiempo de algún evento en la cola ya llegó
  while (eventQueue.length > 0 && now >= eventQueue[0].timestamp) {
      let ev = eventQueue.shift();

      // Nace una nueva animación activa
      activeAnimations.push({
          startTime: ev.timestamp,
          duration: ev.delta * 1000, 
          type: ev.sound,
          x: random(width * 0.2, width * 0.8), 
          y: random(height * 0.2, height * 0.8),
          color: getColorForSound(ev.sound)
      });
  }
}

// ==========================================
// 3. MOTOR DE JUEGO / LOOP PRINCIPAL
// ==========================================
function draw() {
  background(0, 30); 

  // PASO A: Actualizar la lógica y el tiempo
  updateLogic();

  // PASO B: Renderizar (Lo que sería tu drawRunning)
  let now = Date.now() + LATENCY_CORRECTION;
  
  for (let i = activeAnimations.length - 1; i >= 0; i--) {
    let anim = activeAnimations[i];
    let elapsed = now - anim.startTime;
    let progress = elapsed / anim.duration;

    if (progress <= 1.0) {
      dibujarElemento(anim, progress);
    } else {
      activeAnimations.splice(i, 1); // Muere la animación
    }
  }
}

// ==========================================
// 4. FUNCIONES DE DIBUJO (Se mantienen intactas)
// ==========================================
function dibujarElemento(anim, p) {
  push();
  const color = anim.color;
  
  switch (anim.type) {
    case 'tr909bd':
      dibujarBombo(p, color);
      break;
    case 'tr909sd':
      dibujarCaja(p, color);
      break;
    case 'tr909hh':
    case 'tr909oh':
      dibujarHat(anim, p, color);
      break;
    default:
      dibujarDefault(anim, p, color);
      break;
  }
  pop();
}

function dibujarBombo(p, c) {
  let d = lerp(100, 600, p);
  let alpha = lerp(255, 0, p);
  fill(c[0], c[1], c[2], alpha);
  circle(width / 2, height / 2, d);
}

function dibujarCaja(p, c) {
  let w = lerp(width, 0, p);
  let alpha = lerp(255, 0, p);
  fill(c[0], c[1], c[2], alpha);
  rect(width / 2, height / 2, w, 50);
}

function dibujarHat(anim, p, c) {
  let sz = lerp(40, 0, p);
  fill(c[0], c[1], c[2]);
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
  noStroke();
  fill(255, 150);
  textSize(20);
  text(anim.type, 10, 10);
}

function getColorForSound(s) {
  const colors = {
    'tr909bd': [255, 0, 80],
    'tr909sd': [0, 200, 255],
    'tr909hh': [255, 255, 0],
    'tr909oh': [255, 150, 0]
  };
  if (colors[s]) return colors[s];
  let charCode = s.charCodeAt(0) || 0;
  return [(charCode * 123) % 255, (charCode * 456) % 255, (charCode * 789) % 255];
}  

function windowResized() { resizeCanvas(windowWidth, windowHeight); }
```
**Index**
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>P5 Visuals</title>
  <link rel="stylesheet" href="style.css" />
  <script src="https://cdn.jsdelivr.net/npm/p5@1.11.11/lib/p5.js"></script>
  <script src="bridgeClient.js"></script>
  <script src="sketch.js"></script>
</head>
<body>
</body>
</html>

```



## Bitácora de reflexión
