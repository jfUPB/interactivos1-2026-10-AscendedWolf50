# Unidad 5
## Bitácora de proceso de aprendizaje
### Actividad 01
#### Paso 01
**¿Qué ventajas y desventajas ves en usar un formato binario en lugar de texto ASCII?¨**

Lo que mas noto es que el formato binario es mas eficiente. Al enviar los datos directamente en bytes en lugar de caracteres de texto, el tamaño del paquete se mucho (en el ejemplo, pasamos de usar unos 19 bytes a 6 bytes). Esto ahorra memoria y hace que la transmisin sea mas rápida. Ademas, como el tamaño del paquete es fijo, el sistema es mas estable y predecible.

La desventaja mas grande es que la informacion ya no es facil de leer para nosotros. Con el protocolo ASCII, si yo imprimia los datos en la consola, podía leer "500,524,True,False". Con el binario, si lo leo sin procesar, solo vere caracteres extraños o ilegibles.

**Si xValue=500, yValue=524, aState=True, bState=False, ¿cómo se vería el paquete en hexadecimal? (Pista: convierte cada valor según su tipo y anota los bytes en orden.) Respuesta esperada: 01 F4 02 0C 01 00**

xValue (500): Como usa 2 bytes (h), el número 500 en hexadecimal es 01F4. Queda: 01 F4

yValue (524): También usa 2 bytes (h), el número 524 en hexadecimal es 020C. Queda: 02 0C

aState (True): Es un entero sin signo de 1 byte (B). True equivale a 1, en hexadecimal queda: 01

bState (False): También es 1 byte (B). False equivale a 0, en hexadecimal queda: 00

#### Paso 02
**¿Por qué el protocolo ASCII de la unidad anterior no tenía este problema de sincronización? (Pista: piensa en qué rol cumplía el carácter \n.)**

Porque al poder usar el caracter "\n." (el cual funciona como un ENTER) se indicaba cual era el inicio del paquete para evitar confusiones cuando se repetian algun byte.

**¿Por qué en binario no podemos usar \n como delimitador?**

Porque puede ser interpretado como otro valor en lugar de funcionar como un "separador".

## Bitácora de aplicación 
### Actividad 02
**Micro:bit**
```py
from microbit import *
import struct

uart.init(115200)
display.set_pixel(0, 0, 9)

while True:
    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()
    aState = button_a.is_pressed()
    bState = button_b.is_pressed()
    
    # Empaquetar: 2 enteros con signo (2 bytes c/u) y 2 enteros sin signo (1 byte c/u)
    data = struct.pack('>2h2B', xValue, yValue, int(aState), int(bState))
    
    # Calcular checksum: (suma de bytes 1 a 6) % 256
    checksum = sum(data) % 256
    
    # Trama final: Header (0xAA) + Datos + Checksum
    packet = b'\xAA' + data + bytes([checksum])
    
    uart.write(packet)
    sleep(100)
```
**MicrobitBinaryAdapter**
```js
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

class MicrobitBinaryAdapter extends BaseAdapter {
  constructor({ path, baud = 115200, verbose = false } = {}) {
    super();
    this.path = path;
    this.baud = baud;
    this.port = null;
    this.buffer = Buffer.alloc(0); // Usamos Buffer de bytes en lugar de string
    this.verbose = verbose;
  }

  async connect() {
    if (this.connected) return;
    if (!this.path) throw new Error("serialPort is required for microbit device mode");

    this.port = new SerialPort({
      path: this.path,
      baudRate: this.baud,
      autoOpen: false,
    });

    await new Promise((resolve, reject) => {
      this.port.open((err) => (err ? reject(err) : resolve()));
    });

    this.connected = true;
    this.onConnected?.(`serial open ${this.path} @${this.baud} (Binary)`);

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
    this.buffer = Buffer.alloc(0);
    this.onDisconnected?.("serial closed");
  }

  getConnectionDetail() {
    return `serial open ${this.path} (Binary)`;
  }

  _onChunk(chunk) {
    // 1. Concatenar los nuevos bytes al buffer existente
    this.buffer = Buffer.concat([this.buffer, chunk]);

    // 2. Procesar mientras tengamos al menos 8 bytes (el tamaño fijo de nuestra trama)
    while (this.buffer.length >= 8) {
      // Buscar el header de sincronización
      const syncIndex = this.buffer.indexOf(0xAA);

      if (syncIndex === -1) {
        // No hay header a la vista, limpiamos el buffer para no acumular basura
        this.buffer = Buffer.alloc(0);
        break;
      }

      if (syncIndex > 0) {
        // El header no está al principio, descartamos los bytes anteriores
        this.buffer = this.buffer.subarray(syncIndex);
        if (this.buffer.length < 8) break; // Esperamos a que lleguen más datos
      }

      // 3. Extraer la trama de 8 bytes
      const packet = this.buffer.subarray(0, 8);

      // 4. Calcular el checksum (suma de los bytes de datos: índices 1 al 6)
      let sum = 0;
      for (let i = 1; i <= 6; i++) {
        sum += packet[i];
      }
      const calculatedChecksum = sum % 256;

      // 5. Verificar integridad
      if (calculatedChecksum === packet[7]) {
        // Trama válida: Extraer datos (BE = Big Endian)
        const x = packet.readInt16BE(1);
        const y = packet.readInt16BE(3);
        const btnA = packet[5] === 1;
        const btnB = packet[6] === 1;

        // Emitir los datos formateados idénticos al adapter ASCII
        this.onData?.({ x, y, btnA, btnB });

        // Avanzar el buffer 8 bytes
        this.buffer = this.buffer.subarray(8);
      } else {
        // Trama corrupta
        if (this.verbose) console.warn("Checksum inválido. Trama descartada.");
        // Avanzamos 1 byte para no quedarnos atascados en este falso header
        this.buffer = this.buffer.subarray(1);
      }
    }

    // Prevención de desbordamiento de memoria
    if (this.buffer.length > 4096) {
      this.buffer = Buffer.alloc(0);
    }
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }

  _closed() {
    if (!this.connected) return;
    this.connected = false;
    this.port = null;
    this.buffer = Buffer.alloc(0);
    this.onDisconnected?.("serial closed (event)");
  }

  async writeLine(line) {
    if (!this.port || !this.port.isOpen) return;
    await new Promise((resolve, reject) => {
      this.port.write(line, (err) => (err ? reject(err) : resolve()));
    });
  }

  async handleCommand(cmd) {
    // Mantenemos el envío del comando LED en ASCII para compatibilidad si el firmware no lo cambió
    if (cmd?.cmd === "setLed") {
      const x = Math.max(0, Math.min(4, Math.trunc(cmd.x)));
      const y = Math.max(0, Math.min(4, Math.trunc(cmd.y)));
      const v = Math.max(0, Math.min(9, Math.trunc(cmd.value)));
      await this.writeLine(`LED,${x},${y},${v}\n`);
    }
  }
}

module.exports = MicrobitBinaryAdapter;
```

**bridgeserver.js**
Se descomentaron los bloques:
```js
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");
```
```js
async function createAdapter() {
  

  if (DEVICE === "microbit-bin") { // O "microbitBinary", asegúrate de usar el mismo al ejecutar
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit found at ${path} (Binary Mode)`);
    return new MicrobitBinaryAdapter({ path, baud: BAUD, verbose: VERBOSE });
  }

  
}
```
```js
node bridgeServer.js --device microbit-bin --wsPort 8081
```
**Para levantar el servidor**
## Bitácora de reflexión
