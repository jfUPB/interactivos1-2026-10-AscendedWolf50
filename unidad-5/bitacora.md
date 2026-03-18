# Unidad 5
## Bitácora de proceso de aprendizaje
### Actividad 01
#### Paso 01
**¿Qué ventajas y desventajas ves en usar un formato binario en lugar de texto ASCII?**
-----------------------------------------------------------------------------------------------------------------------
Lo que mas noto es que el formato binario es mas eficiente. Al enviar los datos directamente en bytes en lugar de caracteres de texto, el tamaño del paquete se mucho (en el ejemplo, pasamos de usar unos 19 bytes a 6 bytes). Esto ahorra memoria y hace que la transmisin sea mas rápida. Ademas, como el tamaño del paquete es fijo, el sistema es mas estable y predecible.

La desventaja mas grande es que la informacion ya no es facil de leer para nosotros. Con el protocolo ASCII, si yo imprimia los datos en la consola, podía leer "500,524,True,False". Con el binario, si lo leo sin procesar, solo vere caracteres extraños o ilegibles.

**Si xValue=500, yValue=524, aState=True, bState=False, ¿cómo se vería el paquete en hexadecimal? (Pista: convierte cada valor según su tipo y anota los bytes en orden.) Respuesta esperada: 01 F4 02 0C 01 00**
-----------------------------------------------------------------------------------------------------------------------
xValue (500): Como usa 2 bytes (h), el número 500 en hexadecimal es 01F4. Queda: 01 F4

yValue (524): También usa 2 bytes (h), el número 524 en hexadecimal es 020C. Queda: 02 0C

aState (True): Es un entero sin signo de 1 byte (B). True equivale a 1, en hexadecimal queda: 01

bState (False): También es 1 byte (B). False equivale a 0, en hexadecimal queda: 00

## Bitácora de aplicación 


## Bitácora de reflexión
