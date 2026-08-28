# Algoritmos de Huffman

Compresor y descompresor de archivos basado en el **algoritmo de Huffman**, desarrollado en Java como Trabajo Práctico Final grupal para la materia *Algoritmos y Estructuras de Datos*.

Inspirado en compresores como WinRAR y 7-Zip, el programa toma cualquier archivo, calcula la frecuencia de aparición de cada byte, construye el árbol de Huffman correspondiente y genera un archivo `.huf` comprimido sin pérdida. El mismo programa permite revertir el proceso y reconstruir el archivo original de forma exacta.

## Tabla de contenidos

- [Descripción](#descripción)
- [Cómo funciona el algoritmo](#cómo-funciona-el-algoritmo)
- [Formato del archivo `.huf`](#formato-del-archivo-huf)
- [Estructura del proyecto](#estructura-del-proyecto)
- [Requisitos](#requisitos)
- [Compilación y ejecución](#compilación-y-ejecución)
- [Uso de la aplicación](#uso-de-la-aplicación)
- [Tests](#tests)
- [Tecnologías utilizadas](#tecnologías-utilizadas)
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

## Tests

El proyecto usa **JUnit 5** para las pruebas automatizadas:

- `CompresorTest`: valida el conteo de ocurrencias de bytes.
- `BitReaderTest` / `BitWriterTest`: validan la lectura y escritura bit a bit sobre streams.
- `UnitaryTest`: prueba de integración que comprime un archivo de ejemplo, verifica que el `.huf` se genere y que la descompresión reconstruya el contenido original.

Ejecutar toda la suite de tests:

```bash
mvn test
```

## Tecnologías utilizadas

- **Java 17**
- **Maven** (gestión de dependencias y build)
- **JUnit 5** (`junit-jupiter`) para testing
- **Swing** (`javax.swing`) para la interfaz de consola gráfica interactiva

## Equipo de trabajo

- Gerardi, Santino
- Labayen, Franco
