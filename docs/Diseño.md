# Práctica de Sistemas Operativos: Ejecutor de Lotes

Curso: ST0257 — Sistemas Operativos
Estudiante: Yessetk Rodríguez
Sistema operativo: Linux

## 1. Introducción

El presente documento describe el diseño de un sistema que simula la ejecución de procesos por lotes propia de los sistemas operativos de mainframe. Un cliente registra programas y ficheros en una región de almacenamiento, y luego solicita su ejecución. Cada proceso por lotes lee de su entrada estándar, procesa los datos y escribe su salida estándar.

El sistema se compone de cuatro servicios independientes (`ctrllt`, `gesfich`, `gesprog`, `ejecutor`) que se comunican mediante tuberías nombradas y mensajes en formato JSON. Esta primera entrega cubre únicamente el diseño; el cliente y la implementación se presentarán en la segunda entrega.

## 2. Descripción general

```
cliente → ctrllt → gesfich  → aralmac/files/
                 → gesprog  → aralmac/progs/
                 → ejecutor → aralmac/
```

Cada componente es un proceso independiente. La comunicación se realiza mediante tuberías nombradas (FIFO) creadas con `mkfifo`. Los mensajes intercambiados son cadenas JSON delimitadas por salto de línea.

El sistema admite múltiples clientes simultáneos. El componente `ctrllt` actúa como pasarela y, cuando es necesario, los clientes pueden comunicarse directamente con cualquier servicio sin pasar por él.

## 3. Comunicación entre procesos

Linux ofrece tuberías nombradas como mecanismo estándar de IPC. Estas tuberías son half-duplex, por lo que cada canal de comunicación entre dos procesos requiere dos FIFO: una para enviar y otra para recibir.

Las FIFO utilizadas en el sistema son:

- `/tmp/ctrllt_in` y `/tmp/ctrllt_out` para el controlador.
- `/tmp/gesfich_in` y `/tmp/gesfich_out` para el gestor de ficheros.
- `/tmp/gesprog_in` y `/tmp/gesprog_out` para el gestor de programas.
- `/tmp/ejecutor_in` y `/tmp/ejecutor_out` para el ejecutor.
- Cada cliente crea su propia tubería de respuesta con un nombre único, por ejemplo `/tmp/cli_<pid>`.

### 3.1 Formato de los mensajes

Cada mensaje es una cadena JSON terminada con un salto de línea. Los saltos internos se escapan dentro del propio JSON. La estructura general de una petición es:

```json
{
  "id":      "<contador-mensaje>",
  "pid":     <pid-cliente>,
  "service": "gesfich|gesprog|ejecutor|ctrllt",
  "reply":   "/tmp/cli_<pid>",
  "op":      "<operacion>"
}
```

El campo `id` identifica de forma única cada envío y permite correlacionar una petición con su respuesta. El campo `pid` identifica al cliente al que va dirigida la respuesta, lo que permite que varios clientes lean del mismo FIFO de salida sin confusión. El campo `reply` indica la ruta de la tubería donde el cliente espera recibir la respuesta.

Las respuestas conservan los mismos `id` y `pid` y siguen una de estas dos formas:

```json
{"id":"<id>","pid":<pid>,"status":"ok", ...}
{"id":"<id>","pid":<pid>,"status":"error","msg":"<descripción>"}
```

### 3.2 Lectura compartida (polling)

Cuando varios clientes leen del mismo FIFO de respuesta, cada uno aplica el siguiente procedimiento al recibir un mensaje:

1. Leer el mensaje del FIFO.
2. Comparar el `pid` del mensaje con su propio PID.
3. Si coincide, consumir la respuesta.
4. Si no coincide, reescribir el mensaje en el FIFO para que otro cliente lo recoja.

De esta forma se garantiza que ningún mensaje se pierde y que cada cliente solo procesa las respuestas dirigidas a él.

## 4. Componentes del sistema

### 4.1 Cliente

El cliente registra, consulta, actualiza y borra programas y ficheros, y solicita la ejecución, consulta y terminación de procesos por lotes. Su implementación corresponde a la segunda entrega.

Sinopsis:

```
cliente -c <tubería-nombrada> [-a <tubería-nombrada>]
```

La opción `-c` indica la tubería por la que se envían las peticiones, ya sea hacia `ctrllt` o directamente hacia uno de los servicios. La opción `-a` indica la tubería por la que se reciben las respuestas; es necesaria en Linux porque las tuberías nombradas son half-duplex.

El sistema soporta múltiples clientes ejecutándose en simultáneo. Cada uno se identifica por su PID, incluido en cada mensaje, y procesa solo sus propias respuestas siguiendo el patrón de polling descrito en la sección 3.2.

### 4.2 Controlador de lotes (ctrllt)

`ctrllt` es el componente central del sistema. Recibe las peticiones de los clientes, las dirige al servicio apropiado, espera la respuesta y la devuelve al cliente. También se encarga de crear las tuberías que requiere para conectarse con los demás procesos.

Sinopsis:

```
ctrllt -c <ctrllt_in>  [-a <ctrllt_out>]
       -f <gesfich_in> [-b <gesfich_out>]
       -p <gesprog_in> [-c <gesprog_out>]
       -e <ejecutor_in> [-d <ejecutor_out>]
```

Las opciones `-c`, `-f`, `-p` y `-e` indican las tuberías para enviar peticiones a cada servicio. Las opciones `-a`, `-b`, el segundo `-c` y `-d` indican las tuberías para recibir las respuestas correspondientes. Esta segunda forma es obligatoria en Linux por la naturaleza half-duplex de las FIFO.

Su máquina de estados es muy simple: comienza en *Corriendo* y, ante una orden de terminación, pasa a *Terminado*.

Para atender múltiples clientes simultáneos, `ctrllt` mantiene un conjunto fijo de hilos trabajadores y una cola de tareas compartida. El hilo principal lee del FIFO de entrada y encola cada petición; los trabajadores la desencolan y procesan. No se crea un hilo por mensaje. Cada canal hacia un servicio cuenta con su propio mecanismo de exclusión mutua para serializar las peticiones y evitar mezclas en el FIFO.

El ruteo se decide a partir del campo `service` del mensaje. Cuando el servicio destino es `ctrllt` (por ejemplo, una orden de terminación), la operación se atiende localmente.

Si el servicio se encuentra suspendido, los mensajes entrantes se descartan silenciosamente; solo se procesan las órdenes de reanudar y terminar.

### 4.3 Gestor de ficheros (gesfich)

`gesfich` se encarga de crear, leer, actualizar y borrar los ficheros que se almacenan en `aralmac/files/`.

Sinopsis:

```
gesfich -f <tubería-nombrada> [-b <tubería-nombrada>] -x <info-aralmac>
```

Su máquina de estados contempla los estados *Corriendo*, *Suspendido* y *Terminado*. Las transiciones se realizan mediante las operaciones `suspender`, `reasumir` y `terminar`.

Las operaciones soportadas son:

- `crear`: crea un fichero vacío en el área de almacenamiento y devuelve un identificador con el formato `f-XXXX`, donde X es un dígito (por ejemplo, `f-0001`, `f-0002`). Cada fichero es único y dispone de su propio espacio para los datos.
- `leer`: si recibe un identificador devuelve el contenido del fichero correspondiente; si no recibe ningún identificador, devuelve la información de todos los ficheros registrados.
- `actualizar`: recibe un identificador y la ruta de un fichero existente. Si ambos son válidos, copia el contenido del fichero indicado por la ruta dentro del fichero almacenado en `aralmac`.
- `borrar`: recibe un identificador, lo verifica y elimina el fichero del área de almacenamiento.
- `suspender` y `reasumir`: cambian el estado del servicio entre *Corriendo* y *Suspendido*.
- `terminar`: lleva el servicio al estado *Terminado* y cierra su ejecución.

Una característica fundamental del servicio es que persiste su estado en disco. Los ficheros registrados, el contador de identificadores y el contador de errores se conservan en `aralmac/`, de forma que el servicio puede reiniciarse sin pérdida de información. Toda operación de creación, actualización o borrado se refleja inmediatamente en el sistema de archivos.

El servicio mantiene además un contador de operaciones fallidas (identificadores inexistentes, rutas inválidas, fallos de E/S). Cuando este contador alcanza un umbral configurable, por ejemplo veinte errores, se realiza una limpieza de la región `aralmac/files/` y se reinicia el contador. Esta política protege al sistema frente a un estado corrupto persistente.

Mientras el servicio se encuentra en *Suspendido*, los mensajes entrantes distintos de `reasumir` o `terminar` se ignoran sin generar respuesta.

### 4.4 Gestor de programas (gesprog)

`gesprog` administra los programas ejecutables almacenados en `aralmac/progs/`. Sus responsabilidades son guardar, consultar, actualizar y borrar programas.

Sinopsis:

```
gesprog -p <tubería-nombrada> [-c <tubería-nombrada>] -x <info-aralmac>
```

Su máquina de estados es análoga a la de `gesfich`, con la particularidad de que la operación `leer` también está permitida cuando el servicio se encuentra suspendido.

Las operaciones son:

- `guardar`: recibe el ejecutable (binario o guion), la lista de argumentos y la lista de variables de ambiente con la que el programa debe ejecutarse. Lo copia en el área de almacenamiento y devuelve un identificador con el formato `p-XXXX` (por ejemplo, `p-0001`).
- `leer`: si recibe un identificador devuelve la información asociada al programa; en caso contrario, devuelve la lista de todos los programas registrados.
- `actualizar`: recibe un identificador y la ruta de un fichero. Si ambos son válidos, copia el contenido del fichero indicado dentro del programa almacenado en `aralmac`.
- `borrar`: recibe un identificador y elimina el programa junto con su información asociada.
- `suspender` y `reasumir`: alternan el estado del servicio. En *Suspendido* solo permanece habilitada la operación `leer`.
- `terminar`: cierra la ejecución del servicio.

Cada programa registrado guarda en `aralmac/progs/` el ejecutable bajo su identificador y, junto a él, su información asociada: ruta original, argumentos y variables de ambiente.

### 4.5 Ejecutor (ejecutor)

`ejecutor` es responsable de lanzar los procesos por lotes a partir de los programas y ficheros previamente registrados.

Sinopsis:

```
ejecutor -e <tubería-nombrada> [-d <tubería-nombrada>] -x <info-aralmac>
```

Su máquina de estados se ilustra a continuación:

```
inicio → Ejecutar  ⇄  Suspendidos
         Ejecutar  →  Parar  →  (procesos = 0)  →  Terminado
```

En el estado *Ejecutar* el servicio acepta las operaciones de lanzamiento, consulta y terminación de procesos. La transición a *Parar* impide aceptar nuevas ejecuciones, y solo cuando el número de procesos activos llega a cero el servicio pasa a *Terminado*.

Las operaciones son:

- `ejecutar`: recibe la información de un proceso por lotes, formada por el identificador del programa y los identificadores del fichero de entrada y de salida. Tanto la entrada como la salida son obligatorias, ya que cada proceso por lotes tiene exactamente un fichero de entrada y un fichero de salida. Si la información es correcta el servicio devuelve un identificador del proceso, con el formato `e-XXXX`; en caso contrario devuelve un mensaje de error.
- `estado`: si recibe un identificador devuelve el estado actual del proceso correspondiente; si no, devuelve la lista del estado de todos los procesos.
- `matar`: recibe un identificador y fuerza la terminación del proceso por lotes.
- `suspender` y `reasumir`: cambian el estado del servicio entero entre *Ejecutar* y *Suspendidos*. Mientras se encuentra suspendido, las peticiones distintas de `reasumir` se ignoran.
- `parar`: lleva el servicio al estado *Parar*. No acepta nuevas órdenes de ejecución y espera a que terminen los procesos que aún están activos antes de transitar a *Terminado*.

Cada proceso por lotes puede encontrarse en uno de los siguientes estados: *corriendo*, *suspendido*, *terminado* o *matado*.

### 4.6 Área de almacenamiento (aralmac)

`aralmac` no es un proceso, sino el área en la que se guarda toda la información que el sistema necesita conservar. Aunque la especificación admite que sea una base de datos, en este diseño se ha elegido un directorio del sistema de archivos por su sencillez y por no requerir dependencias externas.

La estructura es la siguiente:

```
aralmac/
├── .fich_counter
├── .fich_errors
├── .prog_counter
├── files/
│   ├── f-0001
│   └── f-0002
└── progs/
    ├── p-0001
    ├── p-0001.meta
    └── p-0002
```

Los servicios `gesfich` y `gesprog` persisten su estado en este directorio, lo que les permite sobrevivir reinicios sin pérdida de información.

## 5. Protocolo de mensajes: ejemplos

A continuación se muestran ejemplos representativos de los mensajes intercambiados entre el cliente y los servicios.

Crear un fichero:

```json
{"id":"1","pid":1234,"service":"gesfich","reply":"/tmp/cli_1234","op":"crear"}
{"id":"1","pid":1234,"status":"ok","id_fichero":"f-0001"}
```

Registrar un programa:

```json
{"id":"2","pid":1234,"service":"gesprog","reply":"/tmp/cli_1234","op":"guardar",
 "exec":"/usr/bin/sort","args":"-r","env":"PATH=/usr/bin:/bin"}
{"id":"2","pid":1234,"status":"ok","id_programa":"p-0001"}
```

Lanzar un proceso por lotes:

```json
{"id":"3","pid":1234,"service":"ejecutor","reply":"/tmp/cli_1234","op":"ejecutar",
 "prog":"p-0001","stdin":"f-0001","stdout":"f-0002"}
{"id":"3","pid":1234,"status":"ok","job":"e-0001"}
```

Consultar el estado de un proceso:

```json
{"id":"4","pid":1234,"service":"ejecutor","reply":"/tmp/cli_1234","op":"estado","job":"e-0001"}
{"id":"4","pid":1234,"status":"ok","job":"e-0001","state":"corriendo"}
```

## 6. Estructura del sistema

El sistema está conformado por cinco procesos: `ctrllt`, `gesfich`, `gesprog`, `ejecutor` y `cliente`. Los cuatro primeros se ejecutan como servicios independientes y deben estar activos para que el sistema responda; el cliente, que se entregará en la siguiente fase, es el componente que interactúa con el usuario.

Desde el punto de vista lógico, cada servicio constituye un módulo separado, y se contempla un módulo común que reúne las utilidades compartidas: análisis de mensajes JSON, manejo de tuberías nombradas y mecanismos de sincronización. El área `aralmac` se materializa como un directorio del sistema de archivos.

La puesta en marcha del sistema sigue esta secuencia:

1. Se crean las tuberías nombradas necesarias para cada canal de comunicación.
2. Cada servicio se lanza como un proceso independiente y queda a la escucha en su FIFO de entrada.
3. `ctrllt` arranca su conjunto de hilos trabajadores y queda preparado para multiplexar peticiones de varios clientes.
4. Los clientes crean su propia tubería de respuesta y comienzan a enviar peticiones JSON.
5. La terminación se realiza de forma ordenada enviando la operación `terminar` a cada servicio, que transita a su estado *Terminado* siguiendo su máquina de estados.

## 7. Decisiones de diseño

A lo largo del diseño se tomaron varias decisiones que conviene justificar.

Se eligió usar dos tuberías nombradas por canal porque las FIFO de Linux son half-duplex y ésta es la forma natural de obtener comunicación bidireccional empleando únicamente primitivas estándar del sistema operativo, sin recurrir a librerías externas.

Cada mensaje viaja como una línea JSON terminada en salto de línea. Esta convención permite leer mensajes completos de manera inequívoca, ya que los saltos de línea internos se escapan dentro del propio JSON.

Para soportar múltiples clientes en `ctrllt` se decidió emplear un conjunto fijo de hilos trabajadores con una cola compartida, en lugar de crear un hilo por mensaje. Esta elección evita el coste asociado a la creación y destrucción continua de hilos y permite acotar la concurrencia. Es una variante clásica del esquema productor-consumidor.

Los identificadores `id` y `pid` que viajan en cada mensaje resuelven dos problemas distintos: el primero correlaciona una petición con su respuesta, y el segundo permite que múltiples clientes lean del mismo FIFO de salida descartando los mensajes que no les corresponden. Este es el patrón de polling descrito en la sección 3.2.

Se admite la comunicación directa del cliente con cualquiera de los servicios sin pasar por `ctrllt`. Aunque la pasarela centraliza el flujo habitual, tener acceso directo facilita la depuración y reduce la latencia en operaciones puntuales.

Cuando un servicio se encuentra en estado suspendido, los mensajes entrantes se ignoran. Solo se procesan las órdenes que reactivan el servicio o lo terminan. Esta política produce un comportamiento simple y predecible.

`gesfich` persiste su estado en disco como requisito explícito. Si el servicio se reinicia, los ficheros registrados, el contador de identificadores y el contador de errores se recuperan sin pérdida. Asociado a esta persistencia, se introduce un mecanismo de limpieza por umbral: tras una cantidad configurable de operaciones fallidas (por ejemplo veinte) se purga la región de ficheros, lo que sirve como defensa frente a estados corruptos.

Cada proceso por lotes tiene exactamente un fichero de entrada y uno de salida, ambos obligatorios. Esta restricción es coherente con el modelo clásico de procesamiento por lotes de los sistemas de mainframe, en el que un proceso lee de su entrada estándar y escribe en su salida estándar.

Finalmente, `aralmac` se implementa como un directorio del sistema de archivos. La especificación admite también una base de datos, pero la opción de directorio es más simple y suficiente para los requisitos de la práctica.

## 8. Plan de implementación y organización del código fuente

La presente entrega cubre exclusivamente el diseño. La segunda entrega traerá consigo la implementación completa en lenguaje C sobre Linux, junto con su sistema de compilación, sus guiones de arranque y un conjunto mínimo de pruebas. A continuación se describe la organización propuesta para el repositorio, así como las razones que motivan dicha distribución.

### 8.1 Estructura general del repositorio

La raíz del proyecto contendrá los siguientes elementos: un fichero `README.md` con instrucciones de compilación y ejecución, un fichero `Makefile` que orquesta la construcción de todos los artefactos, un directorio `docs/` con la documentación, un directorio `src/` con el código fuente, un directorio `include/` con las cabeceras públicas, un directorio `scripts/` con los guiones de arranque y parada, un directorio `tests/` con las pruebas y un directorio `build/` que se generará durante la compilación y que no formará parte del control de versiones.

El árbol de implementación previsto es el siguiente:

```
Practica_SO/
├── Makefile
├── README.md
├── .gitignore
├── docs/
│   ├── Diseño.md
│   └── diagramas/
├── include/
│   ├── common/
│   │   ├── ipc.h
│   │   ├── json.h
│   │   ├── log.h
│   │   ├── msg.h
│   │   └── util.h
│   ├── ctrllt.h
│   ├── gesfich.h
│   ├── gesprog.h
│   └── ejecutor.h
├── src/
│   ├── common/
│   │   ├── ipc.c
│   │   ├── json.c
│   │   ├── log.c
│   │   ├── msg.c
│   │   └── util.c
│   ├── cliente/
│   │   ├── main.c
│   │   ├── repl.c
│   │   └── comandos.c
│   ├── ctrllt/
│   │   ├── main.c
│   │   ├── router.c
│   │   ├── pool.c
│   │   └── cola.c
│   ├── gesfich/
│   │   ├── main.c
│   │   ├── store.c
│   │   ├── ops.c
│   │   └── persistencia.c
│   ├── gesprog/
│   │   ├── main.c
│   │   ├── store.c
│   │   ├── ops.c
│   │   └── meta.c
│   └── ejecutor/
│       ├── main.c
│       ├── lanzador.c
│       ├── tabla_procesos.c
│       └── ciclo_vida.c
├── scripts/
│   ├── arrancar.sh
│   ├── detener.sh
│   ├── limpiar_aralmac.sh
│   └── crear_fifos.sh
├── tests/
│   ├── unit/
│   │   ├── test_json.c
│   │   ├── test_msg.c
│   │   └── test_ipc.c
│   └── integracion/
│       ├── flujo_basico.sh
│       └── multiples_clientes.sh
├── build/
└── aralmac/
    ├── files/
    └── progs/
```

### 8.2 El módulo común

El directorio `src/common/` reunirá la lógica que comparten todos los servicios. En particular, contendrá el análisis y la generación de mensajes JSON, las primitivas para crear, abrir y leer tuberías nombradas, las utilidades de bitácora y los envoltorios para los mecanismos de sincronización empleados por el conjunto de hilos trabajadores. Las cabeceras correspondientes residirán en `include/common/`, lo que permite que cualquier servicio incluya únicamente las interfaces públicas y desconozca los detalles internos.

La separación entre `src/` e `include/` no es arbitraria. Mantener las cabeceras en un directorio independiente facilita expresar al compilador, mediante una única opción `-I include/`, dónde encontrar las interfaces, y reduce el riesgo de incluir por accidente ficheros internos de otro módulo.

### 8.3 Un directorio por servicio

Cada uno de los cuatro servicios principales dispondrá de su propio subdirectorio dentro de `src/`. Esta partición refleja la independencia de los procesos descrita en la sección 4 y produce binarios separados, uno por servicio, lo que simplifica la depuración y permite arrancar y detener cada componente por separado.

Dentro de cada subdirectorio se separa el punto de entrada del servicio (`main.c`) de la lógica de negocio (operaciones, almacenamiento, máquina de estados). De esta forma el `main` queda reducido al análisis de los argumentos de línea de órdenes, la apertura de las tuberías y la entrega del control al bucle principal del servicio.

El cliente, aunque pertenece a la segunda entrega, se ubicará igualmente en `src/cliente/` y reutilizará el mismo módulo común para evitar duplicación.

### 8.4 Guiones de arranque y herramientas auxiliares

El directorio `scripts/` contendrá los guiones de soporte. El guion `crear_fifos.sh` será responsable de invocar `mkfifo` para todas las tuberías nombradas que requiere el sistema, con permisos adecuados. El guion `arrancar.sh` lanzará en segundo plano cada uno de los servicios en el orden correcto y registrará sus identificadores de proceso para poder detenerlos posteriormente. El guion `detener.sh` enviará la operación `terminar` a cada servicio respetando la secuencia ordenada que se describe en la sección 6, esto es, primero el ejecutor, después los gestores y finalmente el controlador. El guion `limpiar_aralmac.sh` permitirá restaurar el área de almacenamiento a un estado vacío, lo que resulta útil al desarrollar y al ejecutar pruebas.

### 8.5 Sistema de compilación

El `Makefile` raíz coordinará la compilación de los binarios. Cada servicio se construirá enlazando sus propios ficheros objeto con los del módulo común. El directorio `build/` recibirá tanto los objetos intermedios como los ejecutables finales, organizados en subdirectorios paralelos a `src/`. El uso de `make` resulta natural en este contexto porque permite recompilar únicamente los ficheros modificados y, gracias a las reglas con dependencias automáticas, mantener actualizadas las relaciones entre fuentes y cabeceras sin intervención manual.

Las opciones de compilación incluirán las advertencias estrictas (`-Wall -Wextra -Wpedantic`) y la información de depuración (`-g`). Una variante de compilación con optimización (`-O2`) quedará disponible mediante una variable del `Makefile`, lo que facilita comparar comportamiento entre ambos modos.

### 8.6 Pruebas

El directorio `tests/` se subdividirá en `unit/` e `integracion/`. Las pruebas unitarias verificarán el comportamiento de los componentes del módulo común sin necesidad de levantar ningún servicio, en particular el analizador de mensajes JSON y los envoltorios de tuberías. Las pruebas de integración serán guiones de consola que arrancan el sistema completo, ejercitan un flujo característico (por ejemplo, registrar un fichero, registrar un programa y lanzar un proceso por lotes) y verifican que las respuestas tienen la forma esperada. La inclusión de un escenario con varios clientes simultáneos pondrá a prueba el patrón de polling descrito en la sección 3.2.

### 8.7 Versionado y exclusiones

El fichero `.gitignore` excluirá del control de versiones los artefactos de compilación (`build/`), las tuberías nombradas eventualmente creadas en el directorio del proyecto y el contenido del área `aralmac/`, ya que esta última es estado de ejecución y no fuente. Se conservará, sin embargo, la estructura mínima de directorios mediante ficheros vacíos `.gitkeep`, de modo que la jerarquía esperada esté presente desde el primer clon del repositorio.

### 8.8 Cronograma sugerido

La construcción de la segunda entrega se prevé en cinco fases. La primera se concentrará en el módulo común, ya que sin un análisis fiable de mensajes y sin envoltorios de tuberías ningún servicio puede operar. La segunda fase abordará `gesfich`, por ser el servicio más sencillo y, a la vez, el primero que ejercita el ciclo completo de petición y respuesta. La tercera fase replicará el patrón en `gesprog`, que comparte gran parte de la lógica de almacenamiento. La cuarta fase implementará el `ejecutor`, que requiere creación de procesos hijos, redirección de descriptores de fichero y una tabla de procesos viva. La quinta fase introducirá `ctrllt` con su conjunto de hilos trabajadores y su cola compartida, para finalmente integrar el cliente y ejecutar las pruebas de integración.

Esta organización persigue dos objetivos. Por un lado, obtener cuanto antes una pieza funcional con la que medir avance real, sin esperar a que todo el sistema esté terminado. Por otro, mantener el código de cada servicio aislado en su propio directorio, de modo que un cambio interno en uno no obligue a recompilar ni a revisar a los demás más allá de las cabeceras públicas del módulo común.

## 9. Decisiones de comunicación entre procesos

Se eligieron tuberías nombradas (FIFO) porque están disponibles en Linux, no requieren bibliotecas externas y permiten tratar la comunicación como lectura y escritura de archivos. Para esta práctica son más simples que otras opciones como sockets UNIX, colas POSIX o memoria compartida, que introducen más pasos de configuración.

Como en Linux las FIFO son half-duplex, cada canal lógico usa dos tuberías: una para enviar y otra para recibir. Esta decisión explica por qué las sinopsis incluyen pares de rutas de entrada y salida.

Los mensajes se envían como líneas JSON; al mantenerlos de tamaño razonable se evita que se mezclen en la FIFO y se simplifica el manejo de concurrencia.

## 10. Concurrencia y sincronización

El sistema permite varios clientes en paralelo. En `ctrllt` se usa un conjunto fijo de hilos trabajadores y una cola compartida: el hilo principal recibe, encola y los trabajadores procesan. Esto evita crear un hilo por petición y mantiene el control de recursos.

Para que las respuestas de cada servicio no se crucen, cada canal de salida se protege con un mecanismo de exclusión mutua. Así una petición y su respuesta viajan en pareja.

Cuando varios clientes comparten una tubería de salida, cada uno valida el `pid` del mensaje antes de consumirlo. Si no coincide, lo reinyecta para que el destinatario correcto lo lea. Esto mantiene el aislamiento entre clientes sin crear una tubería por respuesta.

## 11. Modelo de procesos en el ejecutor

El `ejecutor` recibe el identificador del programa y de los archivos de entrada y salida. Con esos datos localiza el ejecutable y su metadato (argumentos y variables de ambiente), abre los archivos correspondientes y lanza el proceso por lotes con las redirecciones necesarias. El servicio guarda el PID y lo asocia a un identificador `e-XXXX` en una tabla interna para poder consultar estado o terminarlo luego.

El estado de cada proceso se actualiza cuando termina y también cuando el usuario solicita `estado` o `matar`. Con esta tabla, el `ejecutor` puede listar trabajos activos y mantener el control del ciclo de vida sin exponer detalles de bajo nivel al cliente.

## 12. Manejo de señales y cierre ordenado

Los servicios deben terminar de forma limpia. Por eso se atienden señales de terminación (por ejemplo `SIGINT` y `SIGTERM`) como si fueran una operación `terminar`, cerrando recursos y tuberías de manera controlada.

También se ignora la señal de tubería rota (`SIGPIPE`) para que un cliente desconectado no detenga al servicio; en su lugar se reporta un error de escritura.

## 13. Persistencia y recuperación

`gesfich` y `gesprog` persisten su estado en `aralmac/`. Cada operación que modifica el almacenamiento se refleja de inmediato en disco, de modo que un reinicio no pierda información. Para evitar inconsistencias, las actualizaciones se hacen sobre un archivo temporal y luego se reemplaza el original.

El contador de errores de `gesfich` se mantiene en el mismo almacenamiento. Si alcanza el umbral configurado, se limpia la región de archivos y se reinician los contadores; es una medida defensiva ante estados corruptos.

## 14. Manejo de errores y modelo de fallos

Se distinguen errores de formato (JSON mal formado u operación desconocida), errores semánticos (identificadores inválidos, rutas inexistentes) y fallos de infraestructura (servicios caídos o tuberías rotas). En todos los casos se responde con `status:"error"` y un mensaje claro.

El protocolo sigue una semántica de "a lo sumo una vez": el sistema no reintenta automáticamente, para evitar duplicados en operaciones no idempotentes. Si el cliente decide reintentar, puede usar el `id` del mensaje para correlacionar.


## 15. Cierre

Hasta aquí el diseño. Lo que sigue es escribir el código sobre esta estructura y poner a prueba las decisiones tomadas, en particular las relativas a concurrencia y a recuperación tras fallo.
