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


## Bitácora de reflexión
