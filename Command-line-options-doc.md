# Command-line api

Node.js viene con una variedad de CLI options. Estas opciones exponen debugging, múltiples formas de ejecutar scripts y otras opciones de runtime útiles.

## Synopsis

`node [options] [V8 options] [<program-entry-point>] | -e "script" | -] [--] [arguments]`

`node inspect [<program-entry-point> | -e "script" | <host>:<port>] ... `

`node --v8-options`


## Program entry point

El program entry point es un string specifier-like. Si el string no es un absolute path, se resolverá como uno relativo desde el directorio actual. El path lo resuelve el CommonJS module loader. Si no encuentra el fichero, lanza un error.

Si un fichero es encontrado, su path será pasado al ECMAScript module loader bajo algunas condiciones:

- El programa fue iniciado con un flag de command-line que obliga a cargarlo con el loader de ECMAScript module.
- El fichero tiene una extensión .mjs
- El fichero no tiene una extensión .cjs, y el parent package.json contiene un campo top-level "type" con el valor de "module"


De cualquier otra manera, se cargará usando el loader de CommonJS module.


## ECMAScript modules loader entry point - advertencia

Cuando usamos el loader de modules de ECMAScript para el entry point del programa, node command solo aceptará ficheros con .js, .mjs o .cjs; y .wasm cuando el flag --experimental-wasm-modules está activo.


## Options

Todas las opciones, incluyendo las de V8, permiten que las palabras sean separadas por - o por _. Por ejemplo `--pending-deprecation` es equivalente a `--pending_deprecation`.

Si una opción toma un único valor y éste es pasado más de una vez, sólo se tendrá en cuenta el último.
Las opciones pasadas por el command line tiene precedencia sobre las opciones pasadas por NODE_OPTIONS env vars.


### -

Alias para stdin. El script pasa a leer desde stdin, y el resto de opciones es pasado por este script.


### --

Indica el final de un node option. 


### --abort-on-uncaught-exception

Abortar en lugar de salir causa que un core file sea generado post-mortem para analisis usando debugger (como lldb, gdb o mdb)

Si el flag es pasado, el comportamiento aún puede ser seteado a no abortar con process.setUncaughtExceptionCaptureCallback().


### --build-snapshot

Genera un blob snapshot cuando el proceso cierra y escribe al disco, que puede ser cargado luego con --snapshot-blob

Cuando se genera el snapshot, si --snapshot-blob no es especificado, el blob generado será escrito, por defecto, a snapshot.blob en el directorio actual. De otra manera será escrito al path especificado por --snapshot-blob.

```bash
$ echo "globalThis.foo = 'I am from the snapshot'" > snapshot.js

# Corre snapshot.js para inicializar la aplicación y guarda el estado en snapshot.blob
$ node --snapshot-blob snapshot.blob --build-snapshot snapshot.js

$ echo "console.log(globalThis.foor)" > index.js

# Carga el snapshot generado y empieza la aplicación desde index.js
$ node --snapshot-blob snapshot.blob index.js
I am from snapshot
```


### --completition-bash

Imprime el script de finalización bash para node.js

```bash
$ node --completition-bash > node_basb_completition
$ source node_bash_completition
```


### -C condition, --conditions=condition

Activa el soporte experimental para condiciones custom de exportación de resolución.

Cualquier número de custom string condition names son permitidos.

Las condiciones por defecto de Node.js de "node", "default", "import" y "require" siempre van a estar definidas.

Por ejemplo, para correr un módulo con "development" resoutions:

```bash
$ node -C development app.js
```


### --cpu-prof

Inicia el V8 cpu profiler al iniciar, y escribe el cpu profile al disco antes de salir.

Si --cpu-prof-dir no es especificado, el profile generado se guarda en el directorio actual.

Si --cpu-prof-name no es especificado, el profile generado se nombra como CPU.${yyyymmdd}.${hhmmss}.${pid}.${tid}.${seq}.cpuprofile

```bash
$ node --cpu-prof index.js
$ ls *.cpuprofile
CPU.20190409,202950.15293.0.0.cpuprofile
```


### --diagnostic-dir=directory

Setea el directorio al cual el output de diagnosis será escrito. Por defecto en el directorio de trabajo.

Affecta al directorio por defecto de:

- --cpu-prof-dir
- --heap-prof-dir
- --redirect-warnings


### --disable-proto=mode

Deshabilita el Object.prototype.__proto__. Si mode es delete, la propiedad será eliminada por completo. Si mode es trhow, acceder a esta propiedad lanza una excepción con el code ERR_PROTO_ACCESS.


### --disallow-code-generation-from-strings

Hace que se lance error al tratar de generar código desde strings, como con eval o new Function

### --dns-result-order=order

Setea el valor por defecto de varbatim en dns.lookup() y dnsPromises.lookup(). El valor puede ser:

- ipv4first: verbatim false
- verbatim: verbatim true


### --enable-fips

Habilita FIPS-compliant crypto al inicio


### --enable-source-maps

Habilita Source map v3 para stack traces

Cuando usamos un transpiler, como typescript, stack traces referencia el código transpilado, no el original. Este flag trata de referenciar el fichero original.

Sobreescribir Error.prepareStackTrace previene que --enable-source-maps modifique el stack trace.

Habilitar este flag puede producir latencia cuando el Error.stack de la aplicación es accedido. 


### --experimental-global-customevent

Expone el global scope CustomEvent Web API


### --experimental-global-webcrypto

Expone el global scope Web Crypto API


### --experimental-import-meta-resolve

Habilita el soporte experimental para import.meta.resolve().


### --experimental-loader=module

Especifica el module de un custom experimental ECMAScript loader. module puede ser un string aceptado como un import specifier.


### --experimental-network-imports

Habilita el soporte experimental para usar import a través de protocolo https


### --no-experimental-fetch

Deshabilita soporte experimental para Fetch API


### --no-experimental-repl-await

Deshabilita el top level await en REPL


### --experimental-shado-realm

Habilita el soporte para ShadowRealm


### --experimental-specifier-resolution=mode

Setea el algoritmo de resolucion para ES module specifiers. Opciones válidas son explicit y node.

Por defecto explicit, que requiere proveer el full path al module. node mode habilita ssoporte para ficheros de extension opcionales y la habilidad de importar un directorio que tiene un index file.


### --experimental-test-coverage

Cuando se uso con node:test module, un reporte de tests es genrado como parte del output del test runner. Si no hay tests, no se genera nada.


### --experimental-vm-modules

Habilita soporte experimental de ES module para node:vm


### --experimental-wasi-unstable-preview1


### --experimental-wasm-module

Habilita soporte para WebAssembly modules


### --force-context-aware

Deshabilita la carga de addons nativos que no son context-aware


### --force-fips

### --frozen-intrinsics


### --force-node-api-uncaught-exceptions-policy

Fuera uncaughtException event en Node-API callbacks asincronos

Para prevenir un crash de un add-on existente, este flag no está activo por defecto. En el futuro se habilitará por defecto para forzar el comportamiento correcto.


### --heapsnapshot-near-heap-limit=max_count


### --heapsnapshot-signal=signal


### --heap-prof


### --heap-prof-dir


### --heap-prof-name


### --icu-data-dir=file

Especifica el ICU data load path


### --input-type=type

Configura node para interpretar input strings como un CommonJS o como un ES module, escritos desde --eval, --print o STDIN.

Valores válidos son 'commonjs' y 'module'. Por defecto 'commonjs'.

REPL no soporta esta opcion.


### --inspect-brk[=[host:]port]

Activa el inspector en host:port y break al inicio del script. Por defecto host:port es 127.0.0.1:9229


### --inspect-port=[host:]port

Setea host:port para ser usado cuando el inspector es activado. Útil cuando activamos el inspector enviando la señal SIGURSR1.

Por defecto 127.0.0.1


### --inspect[=[host:]port]

Activa el inspector en host:port. Por defecto 127.0.0.1:9229.

V8 inspector integration permite tools como chrome dwvtool e IDEs para debugear instancias de node. 

No es seguro usar ips públicas como 0.0.0.0.


### --inspect-publish-uid=stderr,http

Especifica maneras del inspector en exponer la url del web socket.

Por defecto inspector websocket url está disponible en stderr y bajo /json/list endpoint en http://host:port/json/list


### --insecure-http-parser

Usa un parser http inseguro que acepta headers http inválidos. Esto debe permitir interoperatibilidad con no-conformant http inplementations. Debe permitirr request smuggling y otros http attacks.


### --jitless

Deshabilita runtime allocation of executable memory. Esto puede ser requerido en algunas plataformas por razones de seguridad. 


### --max-http-header-size=size

Especifica el tamaño máximo, en bytes, de los http headers. Por defecto 16kb.


### --napi-modules

Mantenido por compatibilidad (??)


### --no-addons

Deshabilita los node-addons exportados.


### --no-deprecation

Silencia los warnings por deprecation


### --no-extra-info-on-fatal-exception

Esconde la información extra en excepciones fatales que causan salida.


### --no-force-async-hooks-checks

Deshabilita runtime checks para async_hooks.


### --no-global-search-paths

No busca módulos desde global paths como $HOME/.node_modules y $NODE_PATH


### --no-warnings

Silencia todos los warnings


### --node-memory-debug

Habilita extra debug checks para memory leaks en Node.js internals. Esto usualmente es útil para debugging.


### --openssl-config=file

Carga un fichero de configuración de openssl al inicio. 


### --openssl-shared-config

Habilita la sección de configuración de openssl por defecto, openssl_conf, a ser leído desde un fichero de configuración. El fichero de configuración por defecto es llamado openssl.cnf pero esto puede ser modificado usando la env var OPENSSL_CONF, o usando el command line --openssl-config.


### --openssl-legacy-provider


### --pending-deprecation

Emite warnings de pending deprecation.

### --policy-integrity=sri


### --preserve-symlinks

Hace que el module loader preserve los symbolic links cuando resuelve y guarda en cache modulos.

Por defecto, cuando nodejs carga un módulo desde un path que simbólicamente linked a un lugar diferente del disco, nodejs tratará de usar el real path


### --preserve-symlinks-main

El loader preservará los links simbólicos cuando resuelve y cachea el modulo main


### --prof

Genera V8 profiler output


### --profe-process

Process V8 profiler output generado usando el V8 option --prof


### --redirect-warnings=file

Escribe los warnings al fichero dado en lugar de escribirlos a stderr. El fichero será creado si no existe, y hará append si existe. Si un error ocurre mientras trata de escribir el warning al fichero, el warning será escrito a stderr en su lugar.

El nombre del fichero puede ser un absolute path. Si no lo es, usará el directorio por defecto o el dado por --diagnostic-dir.


### --report-compact

Escribe los reportes en un formato compacto, un json de una sola línea, más fácilmente consumible por lo ssistemas de procesamiento que el default multi-line format diseñado para consumo humano.


### --report-dir=directory, report-directory=directory

Lugar al que los reportes serán generados


### --report-filename=filename

Nombre del fichero de reporte donde será escrito

Si el filename es seteado a stdout o stdeerr, el reporte será escrito a stdout o stderr respectivamente.


### --report-on-fatalerror

Habilita que el reporte sea generado en fatal error que hace que el proceso finalice.


### --report-on-signal

Habilita que el reporte sea generado cuando se recibe el signal especificado o el por defecto.


### --report-signal=signal

Setea o reinicia el signal para generar el reporte. Por defecto SIGUSR2


### --report-uncaught-exception

Habilita generar reporte cuando el proceso sale por una excepción no gestionada


### --secure-heap=n

Inicializa un OpenSSL secure heap de n bytes


### --secure-heap-min=n


### --snapshot-blob=path


### --test

Inicia el command line test runner de node.js. No puede ser combinado con --watch-path, --check, --eval, --interactive o el inspector.


### --test-name-pattern

Regex que configura el test runneer para solo ejecutar aquellos tests cuyos nombres hagan match con el pattern dado.


### --test-reporter

Un test reporter a usar cuando corre los tests


### --test-resporter-destination

El destino del reporte de tests


### --test-only

Configura el test runner para solo ejecutar top level tests que tienen la opnción only seteada


### --throw-deprecation

Lanza error para deprecations


### --title=title

Setea process.title en el startup


### --tls-cipher-list=list

Especifica un tls cipher list por defecto alternativo. Requiere node.js a ser construido con crypto support (por defecto)


### --tls-keylog=file

Log tls key material a un fichero.  


### --tls-max-v1.2

Setea tls.DEFAULT_MAX_VERSION a 'TLSv1.2'


### --tls-max-v1.3


### --tls-min-v1.0


### --tls-min-v1.1


### --tls-min-v1.2


### --tls-min-v1.3


### --trace-deprecation


### --trace-event-categories


### --trace-event-file-pattern


### --trace-events-enabled


### --trace-exit


### --trace-sigint


### --trace-sync-io


### --trace-tls


### --trace-uncaught


### --trace-warnings


### --trace-heap-objects


### --unhandled-rejections=mode

- throw: Emite unhandledRejection
- strict: Eleva a uncaught exception
- warn: Siempre emite un warning
- warn-with-error-code: Emite unhandledRejection
- none: Silencia todos los warnings


### --use-bundled-ca, --use-openssl-ca


### --use-largepages=mode

- off: No mapping wil be attempted. Default
- on: If supported by the OS, mapping will be attempted. Failure to map will be ignored and a message will be printed to standard error.
- silent: If supported by the OS, mapping will be attempted. Failrue to map will be ingored and will not be reported


### --v8-options


### --watch

Inicia node en watch mode. Cuando está en modo watch, los cambios en los ficheros que se están haciendo watch causan que el proceso se reinicie. Por defecto se hace watch al entry point y cualquier import. 

```bash
$ node --watch index.js
```

### --watch-path


### --watch-preserve-output

Deshabilita que se borre la consola al reiniciar


### --zero-fill-buffers

Automáticamente rellena los buffers y slowBuffers creados con 0


### -c, --check

Syntax check sin ejecutar


### -e, --eval "script"

Evalua un argumento como js.


### -h, --help


### -i, --interactive

Abre el REPL event si stdin no parece ser un terminal


### -p, --print "script"


### -r, --require module

Preload the specified module at startup

Solo commonjs.


### -v, --version


## Env vars

### FORCE_COLOR=[1,2,3]

Habilita ANSI colorized output. Valores:

- 1, true o '', 16-color support
- 2 256-color
- 3 16 milion-color


### NODE_DEBUG=module[,...]

','-separated list of core modules that should print debug information


### NODE_DEBUG_NATIVE=module[,...]


### NODE_DISABLE_COLORS=1

No colores en REPL


### NODE_EXTRA_CA_CERTS=file


### NODE_ICU_DATA=file


### NODE_NO_WARNINGS=1


### NODE_OPTIONS=options...

ej.

```.env
NODE_OPTIONS='--require "./my-path/file.js"'
```

```.env
NODE_OPTIONS='--inspect=localhost:4444' node --inspect=localhost:5555
```

```.env
NODE_OPTIONS='--require "./a.js"' node --require "./b.js"
# equivalente a
node --require "./a.js" --require "./b.js"
```


### NODE_PATH=path[:...]


### NODE_PENDING_DEPRECATION=1


### NODE_PENDING_PIPE_INSTANCEES=instances


### NODE_PRESERVE_SYMLINKS=1


### NODE_REDIRECT_WARNINGS=file


### NODE_REPL_EXTERNAL_MODULE=file


### NODE_SKIP_PLATFORM_CHECK=value


### NODE_TEST_CONTEXT=value


### NODE_TLS_REJECT_UNAUTHORIZED=value


### NODE_V8_COVERAGE=dir


### NO_COLOR=<any>


### OPENSSL_CONF=file


### SSL_CERT_DIR=dir


### SSL_CERT_FILE=file


### TZ


### UV_THREADPOOL_SIZE=size
