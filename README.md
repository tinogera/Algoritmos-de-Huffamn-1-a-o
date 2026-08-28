# Algoritmos de Huffman

Compresor y descompresor de archivos basado en el **algoritmo de Huffman**, desarrollado en Java como Trabajo Práctico Final grupal para la materia *Algoritmos y Estructuras de Datos*.

Inspirado en compresores como WinRAR y 7-Zip, el programa toma cualquier archivo, calcula la frecuencia de aparición de cada byte, construye el árbol de Huffman correspondiente y genera un archivo `.huf` comprimido sin pérdida. El mismo programa permite revertir el proceso y reconstruir el archivo original de forma exacta.

## Tabla de contenidos

- [Descripción](#descripción)
- [Cómo funciona el algoritmo](#cómo-funciona-el-algoritmo)
- [Ejemplo real de compresión](#ejemplo-real-de-compresión)
- [Formato del archivo `.huf`](#formato-del-archivo-huf)
- [Estructura del proyecto](#estructura-del-proyecto)
- [Referencia de la API](#referencia-de-la-api)
- [Requisitos](#requisitos)
- [Compilación y ejecución](#compilación-y-ejecución)
- [Uso de la aplicación](#uso-de-la-aplicación)
- [Uso programático (sin la interfaz gráfica)](#uso-programático-sin-la-interfaz-gráfica)
- [Tests](#tests)
- [Limitaciones conocidas](#limitaciones-conocidas)
- [Posibles mejoras futuras](#posibles-mejoras-futuras)
- [Tecnologías utilizadas](#tecnologías-utilizadas)
- [Cómo contribuir](#cómo-contribuir)
- [Equipo de trabajo](#equipo-de-trabajo)

## Descripción

El proyecto implementa un compresor de datos sin pérdida (*lossless*) utilizando codificación de Huffman a nivel de byte. El flujo completo consta de dos procesos:

- **Compresión**: analiza el archivo de entrada, construye el árbol de Huffman óptimo según las frecuencias de cada byte, genera el código binario de cada uno y escribe un archivo `<archivo>.huf` que contiene el árbol serializado (encabezado) seguido del contenido codificado en bits.
- **Descompresión**: lee el encabezado del archivo `.huf`, reconstruye el árbol de Huffman original y recorre el contenido bit a bit para restaurar el archivo original byte por byte.

## Cómo funciona el algoritmo

1. **Análisis de frecuencia** (`contarOcurrencias`): se recorre el archivo de entrada byte a byte y se cuenta cuántas veces aparece cada uno de los 256 valores posibles (0-255).
2. **Lista enlazada ordenada** (`crearListaEnlazada`): se genera una lista con un nodo (`HuffmanInfo`) por cada byte que efectivamente aparece en el archivo, ordenada de menor a mayor frecuencia.
3. **Construcción del árbol de Huffman** (`convertirListaEnArbol`): se toman repetidamente los dos nodos de menor frecuencia de la lista, se combinan en un nodo padre cuya frecuencia es la suma de ambos, y se reinserta el padre en la lista manteniendo el orden. El proceso se repite hasta que queda un único nodo: la raíz del árbol. Los caracteres de mayor frecuencia terminan con menor profundidad (código más corto).
4. **Generación de códigos** (`generarCodigosHuffman`): se recorre el árbol (clase `HuffmanTree`, en `huffman.util`) de forma iterativa con una pila, asignando `0` al bajar por la izquierda y `1` al bajar por la derecha, hasta llegar a cada hoja. El código resultante de cada byte se guarda en la tabla `HuffmanTable`.
5. **Escritura del encabezado** (`escribirEncabezado`): se serializa el árbol en el archivo `.huf` (cantidad de hojas, cada byte con su código y el largo total del archivo original).
6. **Escritura del contenido** (`escribirContenido`): se vuelve a recorrer el archivo original y, por cada byte leído, se escribe su código de Huffman bit a bit en el archivo `.huf`, usando un `BitWriter` para empaquetar los bits en bytes reales.
7. **Descompresión** (`recomponerArbol` + `descomprimirArchivo`): se lee el encabezado para reconstruir el árbol exactamente igual a como quedó en la compresión, y luego se recorre el contenido bit a bit: en cada bit se desciende por el árbol (derecha si es `1`, izquierda si es `0`) hasta llegar a una hoja, momento en el que se escribe el byte original en el archivo de salida y se vuelve a la raíz.

## Ejemplo real de compresión

Para ilustrar el comportamiento del algoritmo se comprimió un texto de ejemplo de 1129 bytes con distribución de caracteres típica de un texto en español:

```
Original:   1129 bytes
Comprimido:  743 bytes
Ratio:      65.81 %  (≈ 34 % de reducción)
```

Un fragmento de la tabla de códigos generada para ese archivo (a mayor frecuencia, código más corto):

| Carácter | Ocurrencias | Código Huffman |
|---|---|---|
| `' '` (espacio) | 182 | `001` |
| `o` | 119 | `101` |
| `e` | 99 | `0000` |
| `s` | 79 | `0101` |
| `z` (poco frecuente) | 1 | `0111100000` |

Esto refleja la propiedad central del algoritmo: los caracteres más frecuentes (espacio, vocales) obtienen los códigos más cortos, mientras que los menos frecuentes reciben códigos más largos.

> **Nota sobre archivos pequeños**: con archivos muy chicos (decenas de bytes) el archivo `.huf` puede terminar pesando *más* que el original, porque el encabezado (tabla de códigos) tiene un costo fijo que no siempre se compensa con el ahorro en el contenido. El algoritmo rinde mejor cuanto más grande es el archivo y más desigual es la distribución de frecuencias de sus bytes. Ver [Limitaciones conocidas](#limitaciones-conocidas).

## Formato del archivo `.huf`

El archivo comprimido `<archivo>.huf` tiene la siguiente estructura binaria:

| Campo | Tamaño | Descripción |
|---|---|---|
| Cantidad de hojas | 1 byte | Cantidad de bytes distintos presentes en el archivo original (0 se interpreta como 256) |
| Por cada hoja | variable | Byte original (1 byte) + longitud del código Huffman (1 byte) + código Huffman (bits empaquetados) |
| Largo del archivo original | 4 bytes (`int`) | Cantidad total de bytes que tiene el archivo original, usado para saber cuándo detener la descompresión |
| Contenido codificado | variable | Secuencia de bits con el archivo original codificado según el árbol de Huffman |

## Estructura del proyecto

```
├── pom.xml                                  # Configuración Maven (Java 17, JUnit 5)
└── src
    ├── main/java
    │   ├── huffman/def                      # Contratos (interfaces) y modelos del dominio
    │   │   ├── Compresor.java               # Interfaz del proceso de compresión
    │   │   ├── Descompresor.java            # Interfaz del proceso de descompresión
    │   │   ├── BitReader.java               # Interfaz de lectura de bits desde un InputStream
    │   │   ├── BitWriter.java               # Interfaz de escritura de bits en un OutputStream
    │   │   ├── HuffmanInfo.java             # Nodo del árbol de Huffman (carácter, frecuencia, hijos)
    │   │   └── HuffmanTable.java            # Entrada de la tabla de frecuencias/códigos por byte
    │   ├── huffman/util
    │   │   ├── HuffmanTree.java             # Recorrido iterativo del árbol para generar los códigos
    │   │   ├── HuffmanTreeDemo.java         # Demo standalone (ejemplos "COCORITO" y "Bee Gees")
    │   │   └── Console.java                 # Utilidad de consola gráfica (Swing) para I/O interactivo
    │   └── imple                            # Implementaciones concretas de las interfaces
    │       ├── CompresorImple.java
    │       ├── DescompresorImple.java
    │       ├── BitReaderImple.java
    │       ├── BitWriterImple.java
    │       └── Factory.java                 # Fábrica de instancias (BitReader/BitWriter)
    └── test/java/huffman/def
        ├── GeneralTest.java                 # Punto de entrada interactivo (menú de compresión/descompresión)
        ├── UnitaryTest.java                 # Tests de integración extremo a extremo (comprimir + descomprimir)
        ├── CompresorTest.java               # Test unitario de conteo de ocurrencias
        ├── BitReaderTest.java               # Tests unitarios de lectura de bits
        └── BitWriterTest.java               # Tests unitarios de escritura de bits
```

### Paquetes principales

- **`huffman.def`**: define las interfaces (`Compresor`, `Descompresor`, `BitReader`, `BitWriter`) y los modelos de datos (`HuffmanInfo`, `HuffmanTable`) que representan el árbol y la tabla de frecuencias/códigos.
- **`imple`**: contiene las implementaciones concretas de esas interfaces (`CompresorImple`, `DescompresorImple`, `BitReaderImple`, `BitWriterImple`) y una `Factory` para instanciarlas.
- **`huffman.util`**: utilidades de soporte, entre ellas `HuffmanTree` (algoritmo de recorrido del árbol) y `Console`, una consola gráfica basada en Swing que se usa como interfaz de usuario para elegir archivos e ingresar opciones.

## Referencia de la API

### `huffman.def.Compresor`

| Método | Descripción |
|---|---|
| `HuffmanTable[] contarOcurrencias(String filename)` | Recorre `filename` byte a byte y devuelve un arreglo de 256 posiciones con la cantidad de veces que aparece cada valor. |
| `List<HuffmanInfo> crearListaEnlazada(HuffmanTable[] arr)` | Convierte la tabla de ocurrencias en una lista de nodos ordenada ascendentemente por frecuencia, descartando los bytes que no aparecen. |
| `HuffmanInfo convertirListaEnArbol(List<HuffmanInfo> lista)` | Combina iterativamente los dos nodos de menor frecuencia hasta obtener la raíz del árbol de Huffman. |
| `void generarCodigosHuffman(HuffmanInfo root, HuffmanTable[] arr)` | Recorre el árbol y completa el código binario de cada byte en la tabla `arr`. |
| `long escribirEncabezado(String filename, HuffmanTable[] arr)` | Crea `filename+".huf"` y escribe el encabezado (árbol serializado); devuelve el tamaño en bytes del encabezado. |
| `void escribirContenido(String filename, HuffmanTable[] arr)` | Agrega al final de `filename+".huf"` el contenido del archivo original codificado en bits. |

### `huffman.def.Descompresor`

| Método | Descripción |
|---|---|
| `long recomponerArbol(String filename, HuffmanInfo arbol)` | Lee el encabezado de `filename+".huf"` y reconstruye el árbol de Huffman en `arbol`; devuelve la cantidad de bytes que ocupó el encabezado. |
| `void descomprimirArchivo(HuffmanInfo root, long n, String filename)` | Salta los primeros `n` bytes (encabezado) de `filename+".huf"` y decodifica el contenido escribiendo el resultado en `filename`. |

### `huffman.def.BitReader` / `huffman.def.BitWriter`

| Método | Descripción |
|---|---|
| `using(InputStream/OutputStream)` | Asocia el lector/escritor a un stream concreto. |
| `readBit()` / `writeBit(int bit)` | Lee o escribe un único bit (0 o 1), empaquetando/desempaquetando de a 8 en cada byte real del stream. |
| `flush()` | Completa con ceros el byte parcial pendiente y lo vuelca al stream (o descarta el buffer de lectura al alinear con el siguiente byte). |

### Modelos

- **`HuffmanInfo`**: nodo del árbol de Huffman. Contiene `c` (el byte, o `300` si es un nodo interno), `n` (frecuencia acumulada) y referencias `left`/`right` a sus hijos.
- **`HuffmanTable`**: entrada por byte (0-255) con su cantidad de ocurrencias (`n`) y su código de Huffman asignado (`cod`).

## Requisitos

- **JDK 17** o superior.
- **Maven 3.6+**.

## Compilación y ejecución

Clonar el repositorio y compilar el proyecto con Maven:

```bash
git clone https://github.com/tinogera/algoritmos-de-huffamn-1-a-o.git
cd algoritmos-de-huffamn-1-a-o
mvn compile
```

Ejecutar la aplicación interactiva (requiere entorno gráfico, ya que `Console` utiliza Swing):

```bash
mvn exec:java -Dexec.mainClass="huffman.def.GeneralTest"
```

> Nota: el punto de entrada de la aplicación se encuentra en la clase `GeneralTest` (`src/test/java/huffman/def/GeneralTest.java`). También puede ejecutarse directamente desde un IDE corriendo su método `main`.

## Uso de la aplicación

Al ejecutar `GeneralTest`, se abre una consola gráfica con un menú de opciones:

```
Seleccione una opción:
0. Cerrar Programa
1. Comprimir archivo
2. Descomprimir archivo
```

- **Comprimir archivo**: abre un explorador de archivos para seleccionar el archivo a comprimir y genera `<archivo>.huf` en la misma ubicación.
- **Descomprimir archivo**: abre un explorador de archivos para seleccionar un archivo `.huf` y reconstruye el archivo original (sin la extensión `.huf`).
- **Cerrar Programa**: finaliza la aplicación.

## Uso programático (sin la interfaz gráfica)

Las interfaces `Compresor` y `Descompresor` pueden usarse directamente desde código Java, sin pasar por la consola gráfica de `Console`, lo cual es útil para integrarlas en otros programas o para automatizar pruebas:

```java
import imple.CompresorImple;
import imple.DescompresorImple;
import huffman.def.HuffmanTable;
import huffman.def.HuffmanInfo;
import java.util.List;

// Comprimir "archivo.txt" -> genera "archivo.txt.huf"
CompresorImple compresor = new CompresorImple();
HuffmanTable[] ocurrencias = compresor.contarOcurrencias("archivo.txt");
List<HuffmanInfo> lista = compresor.crearListaEnlazada(ocurrencias);
HuffmanInfo arbol = compresor.convertirListaEnArbol(lista);
compresor.generarCodigosHuffman(arbol, ocurrencias);
compresor.escribirEncabezado("archivo.txt", ocurrencias);
compresor.escribirContenido("archivo.txt", ocurrencias);

// Descomprimir "archivo.txt.huf" -> reconstruye "archivo.txt"
DescompresorImple descompresor = new DescompresorImple();
HuffmanInfo arbolReconstruido = new HuffmanInfo();
long bytesEncabezado = descompresor.recomponerArbol("archivo.txt", arbolReconstruido);
descompresor.descomprimirArchivo(arbolReconstruido, bytesEncabezado, "archivo.txt");
```

> Nótese que `escribirEncabezado`/`escribirContenido` y `recomponerArbol`/`descomprimirArchivo` reciben el nombre del archivo **sin** la extensión `.huf`; ambas clases la agregan o la asumen internamente.

## Tests

El proyecto usa **JUnit 5** para las pruebas automatizadas:

- `CompresorTest`: valida el conteo de ocurrencias de bytes.
- `BitReaderTest` / `BitWriterTest`: validan la lectura y escritura bit a bit sobre streams.
- `UnitaryTest`: prueba de integración que comprime un archivo de ejemplo, verifica que el `.huf` se genere y que la descompresión reconstruya el contenido original.

Ejecutar toda la suite de tests:

```bash
mvn test
```

## Limitaciones conocidas

- **Alfabeto de un solo byte**: la frecuencia se calcula sobre valores de 0 a 255 (bytes crudos), no sobre caracteres Unicode. Un archivo de texto en UTF-8 con acentos o símbolos se comprime igual de forma correcta, pero cada byte se trata como un símbolo independiente, no como parte de un carácter multi-byte.
- **Overhead en archivos chicos**: como el encabezado (tabla de códigos) tiene un tamaño fijo por cada byte distinto presente, en archivos muy pequeños o con muchos símbolos distintos el `.huf` puede resultar más grande que el original (ver [Ejemplo real de compresión](#ejemplo-real-de-compresión)).
- **Sin manejo de errores de entrada**: si el archivo indicado no existe o no se puede leer, las excepciones (`IOException`) se registran con `printStackTrace()` pero no se informan al usuario de forma amigable ni interrumpen el flujo de manera controlada.
- **Interfaz gráfica obligatoria**: `Console` está construida sobre Swing (`JFrame`), por lo que el punto de entrada interactivo (`GeneralTest`) requiere un entorno con soporte gráfico (no funciona en un servidor sin X11/framebuffer sin configuración adicional).
- **Sin soporte de compresión de directorios**: el programa comprime un único archivo por vez; no arma un archivo empaquetado (como `.zip`) a partir de múltiples archivos o carpetas.
- **Nombre del punto de entrada**: la clase con el `main()` de la aplicación (`GeneralTest`) vive en `src/test/java`, no en `src/main/java`, lo cual es una particularidad heredada de la organización original del proyecto.

## Posibles mejoras futuras

- Agregar manejo de errores con mensajes claros para el usuario (archivo inexistente, permisos, disco lleno, etc.).
- Soportar compresión de múltiples archivos o carpetas completas.
- Optimizar el encabezado para reducir el overhead en archivos pequeños (por ejemplo, usando un algoritmo canónico de Huffman que solo necesite las longitudes de código, no el código completo).
- Mover el punto de entrada (`main`) a `src/main/java` y ofrecer, además de la interfaz gráfica, una interfaz de línea de comandos (CLI) para uso en scripts o entornos sin GUI.
- Agregar más pruebas unitarias sobre la construcción del árbol y la codificación/decodificación con archivos binarios (no solo texto).

## Tecnologías utilizadas

- **Java 17**
- **Maven** (gestión de dependencias y build)
- **JUnit 5** (`junit-jupiter`) para testing
- **Swing** (`javax.swing`) para la interfaz de consola gráfica interactiva

## Cómo contribuir

1. Hacer un fork del repositorio y crear una rama descriptiva (`feature/mi-mejora` o `fix/mi-arreglo`).
2. Ejecutar `mvn test` antes de abrir un cambio, para asegurarse de no romper la suite existente.
3. Mantener el estilo de código existente (nombres en español para el dominio del negocio, en inglés para utilidades genéricas) y agregar tests para el código nuevo.
4. Abrir un Pull Request describiendo el cambio y su motivación.

## Equipo de trabajo

- Gerardi, Santino
- Labayen, Franco
