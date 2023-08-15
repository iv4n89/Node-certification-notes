## Process

El objeto process ofrece información sobre, y sobre el control de, el proceso actual de Node.js.

```javascript
import process from 'node:process';
```

## Process events

El objeto `process` es una instancia de `EventEmitter`.

### Event: 'beforExit'

Es emitido cuando Node.js vacía su event loop y no tiene más trabajos encolados. Normalmente el proceso de Node.js sale cuando no tiene más trabajo pendiente, pero un listener en el evento `beforeExit` puede hacer llamadas asíncronas y hacer que el proceso de Node.js continúe.

El callback del listener recibe el process.exitCode como argumento.

No se llamará si hay un cierre explícito, como una llamada a process.exit() o un uncaught exception.

No se debe usar como alternativa a `exit` a menos que necesitemos encolar trabajo adicional

```javascript
import process from 'node:process';

process.on('beforeExit', code => {
  console.log('Process beforeExit event with code: ', code);
});

process.on('exit', code => {
  console.log('Process exit event with code: ', code);
});

console.log('This message is displayed first');

// This message is displayed first
// Process beforeExit event with code: 0
// Process exit event with code: 0
```


### Event: 'disconnect'

Si el proceso de Node.js es generado en un canal IPC (child process, cluster), este evento será emitido cuando el canal IPC es cerrado.


### Event: 'exit'

- code `integer`


El evento es emitido cuando el proceso de Node.js está a punto de cerrarse como resultado de:
- Una llamada explícita a process.exit().
- El event loop de node.js ya no tiene trabajo adicional


No hay manera de prevenir la del event loop en este punto, y una vez que todos los listeners de `exit` han finalizado de correr el proceso de Node.js será matado.

El callback recibe process.exitCode o el exitCode pasado al método process.exit()

```javascript
import process from 'node:process';

process.on('exit', code => {
  console.log('About to exit with code: ', code);
});
```

Los listeners debe sólo realizar operaciones síncronas. El proceso de Node.js saldrá inmediatamente después de llamar a los listneres de exit cuasando que cualquier trabajo pendiente en el event loop sea abandonado.
En el ejemplo, el setTimeout nunca ocurrirá:

```javascript
import process from 'node:process';

process.on('exit', code => {
  setTimeout(() => {
    console.log('This will not run');
  }, 0);
});
```


### Event: 'message'

- message `Object`|`boolean`|`number`|`string`|`null` Un objeto JSON parseado o un valor primitivo serializable.
- sendHandle `net.Server`|`net.Socket` Un net.Server o net.Socket, o undefined.


Si el proceso de Node.js está en un canal IPC (child process, cluster), el evento `message` será emitido siempre que un mensaje sea enviado por el proceso padre usando childprocess.send().

Si la opción `serialization` fue seteada a `advanced` cuando se generó el proceso, el argumento `message` puede contener datos que un JSON no sea capaz de representar.


### Event: 'rejectionHandled'

- promise `<Promise>` La promesa manejada


El evento `rejectionHandled` es emitido siempre que una promesa ha sido rechazada y un error handler se ha atajado a él (usando promise.catch(), por ejemplo) un turno más tarde del event loop.

El objeto Promise habría sido previamente emitido en un evento 'unhandledRejection', pero durante el proceso ganó un handler para el rejection.

No hay una noción de alto nivel para una Promise chain cuyo rejection siempre pueda ser manejado. Al ser inherentemente asíncrono, un Promise rejection puede ser manejado en un punto futuro, posiblemente mucho después del turno del event loop en el que ocurre la emisión del evento unhandledRejection.

El código síncrono, el evento 'uncaughtException' es emitido cuando la lista de unhandled exceptions crece.

En código asíncrono, el evento 'unhandledRejection' es emitido cuando la lista de rejections no manejadas crece y el evento 'rejectionHandled' cuando la lista de rejections se reduce.

```javascript
import process from 'node:process';

const unhandledRejections = new Map();
process.on('unhandledRejection', (reason, promise) => {
  unhandledRejections.set(promise, reason);
});
process.on('rejectionHandled', promise => {
  unhandledRejections.delete(promise);
});
```


### Event: 'uncaughtException'

- err `<Error>` La exception no capturada
- origin `<string>` Idica si la exception se originó desde un unhandled rejection o desde un error síncrono. Puede ser 'uncaughtException' o 'unhandledRejection'. Este último es usado cuando una excepción ocurre en una promesa basada en contexto asíncrono y --unhandled-rejections flag está seteado en strict o throw (default) y el rejection no es manejado, o cuando un rejection ocurre cuando el loader de ES module está en la fase de carga.


El evento 'uncaughtException' es emitido cuando una excepción no capturada de JS se propaga hasta el event loop. Por defecto, node.js maneja estas exceptions imprimiendo el stack trace a stderr y saliendo con código 1, sobreescribiendo cualquier seteo anterior de process.exitCode. Añadir un handler al evento uncaughtException sobreescribe este comportamiento. Alternativamente, cambiar el process.exitCode en el unaughtException acabará por cambiar el process.exitCode con el que cierra el proceso. En otro caso, en presencia de este handler saldrá con code 0.

```javascript
import process from 'node:process';

process.on('uncaughtException', (err, origin) => {
  fs.writeSync(
    process.stderr.fd,
    `Caught exception: ${err}\n`,
    `Exception origin: ${origin}`,
  );
});

setTimeout(() => {
  console.log('This will still run');
}, 500);

// Intencionalmente se causa una excepción, pero no se captura.
nonexistingFunc();
console.log('This will not run.');
```

Se puede monitorear eventos 'uncaughtException' sin sobreescribir el comportamiento por defecto de salir del proceso agregando un 'uncaughtExceptionMonitor' listener.

### Avisos: Usar 'uncaughtException' correctamente

'uncaughtException' está pensado como último recurso para manejar exceptions. El evento NO DEBE SER USADO COMO EQUIVALENTE A `On Error Resume Next`. Unhandled exceptions significa inherentemente que la aplicación está en estado indefinido. Permitir continuar la app sin recuperarse correctamente de la excepción puede causar problemas indeseados e inesperados.

Las exceptions lanzadas desde dentro del listener no serán capturadas. En su lugar el proceso saldrá con un código 0 y el stack trace impreso. Esto es para evitar una recursión infinita.

Tratar de continuar con normalidad después de una excepción no capturada sería como sacarle el cable de electricidad a un pc mientras se actualiza. 9 de cada 10 veces no pasará nada, pero a la décima tendremos el sistema corrupto.

El uso correcto de 'uncaughtException' es realizar una limpieza síncrona de los recursos usados (ej. file descriptors, handles, etc) antes de cerrar el proceso. No es seguro continuar normalmente después de un 'uncaughtException'

Para reinciar una aplicación crasheada de una manera más segura, sea emitido o no el evento 'uncaughtException', un monitor externo debería ser empleado en un proceso separado para detectar fallos en la aplicación y recuperarse o reiniciar si es necesario.


### Event: 'unaughtExceptionMonitor'

- err `<Error>` La excepción capturada
- origin `<string>`


El evento 'uncaughtExceptionMonitor' es emitido antes de un evento 'uncaughtException' o un hook instalado via process.setUncaughtExceptionCaptureCallback().

Instalar un 'uncaughtExceptionMonitor' listener no cambia el comportamiento una vez el evento 'uncaughtException' es emitido. El proveso crasheará si no hay un event listener para 'uncaughtException'.

```javascript
import process from 'node:process';

process.on('uncaughtExceptionMonitor', (err, origin) => {
  MyMonitorTool.logSync(err, origin);
});

// exception intencional
nonexistingFunc();
// Aún así crashea
```


### Event: 'unhandledRejection'

- reason `<Error>` | `<any>` El objeto con el que la promesa fue rechazada (normalmente un Error)
- promise `<Promise>` La promise rejected.


El evento 'unhandledRejection' es emitido cuando un Promise es rejected y no hay un error handler que capture el error durante el turno del event loop. Cuando se programa con promesas, las excepciones son encapsuladas como 'rejected promises'. Las rejections pueden ser capturadas y manejadas usando promise.catch() y son propagadas a través de la cadena de Promise. El evento 'unhandledRejection' es útil para detectar y mantener trackeadas las promesas que son rejected cuyas rejections no han sido manejadas.

```javascript
import process from 'node:process';

process.on('unhandledRejection', (reason, promise) => {
  console.log('Unhandled Rejection at:', promise', 'reason:', reason);
  // Aplicación de un logging específico, lanzar un error, otras lógicas
});

somePromise.the(res => {
  return reportToUser(JSON.pasre(res)); // typo
}); // Sin catch o then
```

Esto va a lanzar también el evento 'unhandledRejection':

```javascript
import process from 'node:process';

function SomeResource() {
  // Inicialmente seteado el status a una promesa rejected
  this.loaded = Promise.reject(new Error('Resource not yet loaded!'));
}

const resource =  new SomeResource();
```

### Event: 'warning'

- warning `<Error>` Claves del warning son:
  - name `<string>` El nombre del warning. Default `'Warning'`
  - message `<string>` Descripción del warning provisto por el sistema
  - stack `<string>` Dónde el warning fue lanzado


El evento `warning` es emitido siempre que Node.js emite un process warning.

Un process warning es similar a un error en que describe condiciones excepcionales que necesitan atención del usuario. Sin embargo, los warnings no son parte del flujo normal de Node.js y JS. Node.js puede lanzar warnings si detecta malas prácticas que pueden llevar a un rendimiento bajo, bugs, o vulnerabilidades de seguridad.

```javascript
import process from 'node:process';

process.on('warning', warning => {
  console.warn(warning.name); // nombre del warning
  console.warn(warning.message); // mensaje del warning
  console.warn(warning.stack); // stack trace
});
```

Por defecto, Node.js va a imprimir los warnings a stderr. La opción `--no-warnings` puede ser usada para suprimir el output por defecto pero el evento `warning` será emitido igualmente por process.

```zhe
$ node
> events.defaultMaxListeners = 1;
> process.on('foo', () => {});
> process.on('foo', () => {});
> (node:38638) MaxListenersExceededWarning: Possible EventEmitter memory leak detected. 2 foo listeners added. Use emitter.setMaxListeners() to increase limit
```

En contraste, el siguiente ejemplo apaga el default warning output y agrega un custom handler al evento 'warning'.

```zhe
$ node --no-warnings
> p + process.on('warning', (warning) => console.warn('Do not do that!'));
> events.defaultMaxListeners = 1;
> process.on('foo', () => {});
> process.on('foo', () => {});
> Do not do that!
```

La opción `--trace-warnings` del command-line puede ser usado para tener el output por defecto de los warnings incluido en el full stack trace del warning.

Lanzando node con `--throw-deprecation` flag causará que los custom deprecation warnings sean tomados como excepciones.

Usando el flag `--trace-deprecation` se imprimirá todos los custom deprecation en el stderr con el stack trace

Usando el flag `--no-deprecation` no se tomarán en cuenta los custom deprecation.

El flage `*-deprecation` sólo afecta a los warnings que usan el nombre `DeprecationWarning`.


### Event: 'worker'

- `worker` `<Worker>` El worker que fue creado


El evento worker es emitido después de que un nuevo worker ha sido creado.


### Emitir custom warnings

Se verá en [process.emitWarning()](#emitwarning)


### Node.js warning names

No hay reglas estrictas para los tipos de warnings emitidos por Node.js (identificados por su propiedad name). Nuegos tipos pueden ser añadidos en cualquier momento. Algunos comunes son:

- `DeprecationWarning`: Indica el uso de algo deprecado de Node.js. Este warning debe incluir una propiedad `code` que identifique el código deprecado.
- `ExperimentalWarning`: Indica el uso de algo experimental de Node.js. Se advierte que puede cambiar en el futuro o eliminarse.
- `MaxListenersExceededWarning`: Indica que demasiados listeners para un evento han sido registrados a un EventEmitter o un EventTarget. Esto es frecuentemente un indicador de memory leak.
- `TimeOutOverflowWarning`: Indica que un valor numérico que no puede entrar en un integer de 32bits ha sido enviado desde un setTimeout o setInterval.
- `UnsupportedWarning`: Indica uso de opciones no soportadas que serán ignoradas o tratadas como un error. Un ejemplo es el uso de http response status cuando se usa el api de compatibilidad de HTTP/2.


### Signal events

Los signal events serán emitidos cuando el process de Node.js recibe un signal. 

Los signals no están disponibles en los threads de workers.

El signal handler recibirá el signal name como el primer argumento ('SIGINT', 'SIGTERM', ...)

El nombre de cada evento será el nombre en mayúsculas del nombre común del signal (ej. 'SIGINT' para SIGINT signals).

```javascript
import process from 'node:process';

// Comenzar a leer desde stdin para que el process no salga.
process.stdin.resume();

process.on('SIGINT', () => {
  console.log('Received SIGINT. Press Control-D to exit.');
});

// Usar una única función para manejar múltiples signals
function handle(signal) {
  console.log(`Received ${signal}`);
}

process.on('SIGINT', handle);
process.on('SIGTERM', handle);
```

- `SIGUSR1` es reservado por Node.js para comenzar el debugger. Es posible instalar un listener pero hacerlo puede interferir con el debugger.
- `SIGTERM` y `SIGINT` tienen handlers por defecto en no-Windows SO que resetean el terminal de node después de salir con código `128 + signal number`. Si uno de estos signals tiene un listener instalado, su comportamiento por defecto será removido (Node.js no saldrá).
- `SIGPIPE` Es ignorado por defecto. Puede tener un listener instalado.
- `SIGHUP` es generado en Windows cuando la consola de Windows es cerrada, y en otras plataformas bajo condiciones similares.
- `SIGTERM` No es soportado en Windows, pero puede ser escuchado.
- `SIGINT` es soportado por todas las plataformas y puede usualmente ser generado con `Ctrl` + `C`.
- `SIGBREAK` es enviado en Windows cuando se pulsa `Ctrl` + `Break`. En plataformas no Windows, puede ser escuchado, pero no se puede generar.
- `SIGWINCH` es enviado cuando la consola es cambiada de tamaño. En Windows sólo ocurre cuando se escribe en consola cuando el cursor ha sido movido, o cuando se usa un tty readable en raw mode.
- `SIGKILL` no puede tener un listener instalado, si lo tiene mata a Node.js.
- `SIGSTOP` no puede tener listeners instalados.
- `SIGBUS`, `SIGFPE`, `SIGSEGV` y `SIGILL`, cuando no son enviados artificialmente con kill(2), inherentemente deja al proceso en un estado en el que no es seguro llamar a los listeners de JS. Hacer esto puede Node.js sin responder.
- `0` puede ser enviado para salir del proceso, no tiene efectos si sale del proceso, pero lanza una excepción si el proceso no sale.


En Windows no se soporta signals por lo que no tiene equivalencia a matar por signal, pero se puede emular con process.kill() y subprocess.kill():

- Enviando `SIGINT`, `SIGTERM` y `SIGKILL` causará la muerte del proceso, luego el sub proceso reportará que el proceso ha sido acabado por un signal.
- Enviar un signal `0` puede usar en cualquier plataforma para matar el proceso.


### process.abort()

Causa que el proceso de Node.js salga inmediatamente y genere un core file.

No disponible en threads de workers.


### process.allowedNodeEnvironmentFlags

- `<Set>`


Conjuto de solo lectura de flags disponibles dentro de NODE_OPTIONS environment var.

Extiende Set, pero sobreescribe `has` para reconocer muchas representaciones posibles de flags. Retorna true en los siguientes casos:

- Los flags pueden omitir el `-` o `--`. Ej. `inspect-brk`, en lugar de `--inspect-brk`.
- Los flags pasados a V8 pueden reemplazar un guión bajo por uno normal, o al contrario (ej. `--perf_baic_prof`, `--perf_basic_prof`)
- Los flags pueden contener uno o más caracteres `=`, después del primero los demás son ignorados (ej. `--stack-trace-limit=100`.
- Los flags deben estar disponibles dentro de NODE_OPTIONS.


### process.arch

- `<string>`


La arquitectura de la CPU para la cual el binario de Node.js fue compilado. Valores posibles son: `arm`, `arm64`, `ia32`, `mips`, `mipsel`, `ppc`, `ppc64`, `s390`, `s390x` y `x64`.

```javascript
import { arch } from 'node:process';

console.log(`This processor architecture is ${arch}`);
```


### process.argv

- `<string[]>`


Retorna un array que contiene los argumentos pasados por el command-line cuando el proceso de Node.js fue ejecutado. El primer argumento será process.execPath. El segundo argumento será el path de JS para ser ejecutado. Los demás serán argumentos adicionales.

```javascript
import { argv } from 'node:process';

argv.forEach((val, index) => {
  console.log(`${index}: ${val}`);
});
```

Lanzando el comando ```$node process-args.js one two=tree four``` tendremos:

```
0: /usr/local/bin/node
1: /Users/mjr/work/node/process.args.js
2: one
3: two=three
4: four
```


### process.argv0

- `<string>`


Guarda una copia de solo lectura del original argv[0] pasado a Node.js al inicio.

```bash
$ bash -c 'exec -a customArgv ./node'
> process.argv[0]
'/Volumes/code/external/node/out/Release/node'
> process.argv0
'customArgv0'
```


### process.channel

- `<Object>`


Si el proceso de Node.js fue generado dentro de un IPC Channel (child process), el `process.channel` referencia el IPC Channel. Si no existe un IPC Channel, será undefined.

### process.channel.ref()

Hace que el IPC Channel mantenga el event loop del proceso corriendo si `.unrief()` fue llamado.

Típicamente manejado a través del número de listeners de `disconnect` y `message` en el objeto `process`. Sin embargo, puede ser usado para pedir un comportamiento por defecto.


### process.channel.unref()

Hace que el channel no mantenga el event loop activo y permite finalizarlo incluso mientras que el channel está activo aún.

Típicamente manejado a través del número de listeners de `disconnect` y `message` en el objeto `process`. Sin embargo, puede ser usado para pedir un comportamiento por defecto.


### process.chdir(directory)

- directory `<string>`


Cambia el directorio de trabajo actual de un proceso de Node.js o lanza una exception si no es capaz de hacerlo 

```javascript
import { chdir, cwd } from 'node:process';

console.log(`Starting directory: ${cwd()}`);
try {
  chdir('/tmp');
  console.log(`New Directory: ${cwd()}`);
} catch (err) {
  console.error(`chdir: ${err});
}
```

No disponible en threads de workers.


### process.config

- `<Object>`


Retorna un `Object` que contiene la representación de las opciones usadas para compilar el ejecutable de Node.js actual. Es lo mismo que el fichero `config.gypi` producido cuando se corre el script `./configure`. 

Un ejemplo de posible output sería:

```json
{
  target_defaults:
    { cflags: [],
      default_configuration: `Release',
      defines: [],
      include_dirs: [],
      libraries: [] },
    variables:
      {
        host_arch: 'x64',
        napi_build+version: 5,
        node_install_npm: 'true',
        node_prefix: '',
        node_shared_cares: 'false',
        node_shared_http_parser: 'false',
        node_shared_libuv: 'false',
        node_shared_zlib: 'false',
        node_use_dtrace: 'false',
        node_use_openssl: 'true',
        node_shared_openssl: 'false',
        strict_aliasing: 'true',
        target_arch: 'x64',
        v8_use_snapshot: 1
      }
}
```

No es sólo lectura y existen módulos que extienden, modifican o reemplazan por completo el valor de process.config.

Modificar process.config, o cualquier propiedad hija, ha sido deprecado. process.config se hará read-only en una release futura.


### process.connected

- `<boolean>`


Si el proceso de Node.js ha sido generado en un IPC Channel, process.connected retornará true mientras que el IPC Channel esté conectado, y false si se llama a process.disconnect().

Una vez process.connected sea false, no será posible enviar más mensajes a través de process.send().


### process.constrainedMemory()
```Experimental```

Obtiene la cantidad de memoria ram que el SO le da al proceso para que realize la operación, o undefined si no se conoce.


### process.cpuUsage([previousValue])

- previousValue `<Object>` Un valor previo retornado por process.cpuUsage()
- return `<Object>`
  - user `<integer>`
  - system `<integer>`


Retorna el usuario y tiempo de uso de CPU del proceso actual, en un objeto con propiedades user y system, cuyos valores son microsegundos. 

Se puede pasar por argumento el resultado previo para que sean restados.

```javascript
import { cpuUsage } from 'node:process';

const startUsage = cpuUsage();
// { user: 38579, system: 6986 }

// 500ms de uso de cpu
const now = Date.now();
while (Date.now() - now < 500);

console.log(cpuUsage(startUsage));
// { user: 514883, system: 11226 }
```


### process.cwd()

- return `<string>`


Retorna el directorio de trabajo actual del proceso de Node.js.

```javascript
import { cwd } from 'node:process';

console.log(`Current directory: ${cwd()}`);
```


### process.debugPort

- `<number>`

Puerto usado por NOde.js cuando el debugger está activo.

```javascript
import process from 'node:process';

process.debugPort = 5858;
```


### process.disconnect()

Si el proceso de node.js ha sido generado dentro de un IPC Channel, este método cerrará el channel desde el proceso padre, permitiendo al proceso hijo salir de manera correcta una vez no hayan más conexiones activas.

Tiene el mismo efecto que llamar a ChildProcess.disconnect() desde el proceso padre.

Si el proceso de Node.js no fue generado en un IPC Channel, retornará undefined.


### process.dlopen(module, filename[, flags])

- module `<Object>`
- filename `<string>`
- flags `<os.constants.dlopen>` Default: `os.constants.dlopen.RTLD_LAZY`


Permite cargar dinámicamente objetos compartidos. Es usado principalmente por require() para cargar Addons de C++, y no debe ser usado directamente, excepto en casos especiales. require() debe preferirse antes que process.dlopen() a menos que haya razones específicas como usar un custom flag o cargar desde ES modules.

Los flags son integers que especifican un comportamiento específico.

Importante al llamarlo que la instancia de module sea pasada. Las funciones exportadas por C++ serán entonces accesibles desde module.exports.

```javascript
import { dlopen } from 'node:process';
import { constants } from 'node:os';
import { fileURLToPath } from 'node:url';

const module = { exports: {} };
dlopen(module, fileURLToPath(new URL('local.node', import.meta.url)),
      constants.dlopen.RTLD_NOW);
module.exports.foo();
```


### process.emitWarning(warning[, options])

- warning `<string> | <Error>` El warning a emitir
- options `<Object>`
  - type `<string>` Cuando warning es un string, type es el nombre a usar para el type del warning a ser emitido. Default `warning`
  - code `<string>` Un identificador único para la instancia siendo emitida.
  - ctor `<Function>` Cuando warning es un string, ctor es una función opcional usada para limitar el stack trace generado. Default: `process.emitWarning`.
  - detail `<string>` Texto adicional a incluir con el error


Este método puede ser usado para emitir process warnings customs o específicos de la aplicación. Pueden ser escuchados agregando un listener al evento 'warning'

```javascript
import { emitWarning } from 'node:process';

// Emite un warning con un código adicional como detalle
emitWarning('Something happened!', {
  code: 'MY_WARNING',
  detail: 'This is some additional information',
});

// Emite:
// (node:56338) [MY_WARNING] Warning: Something happened!
/// This is some additional information
```

```javascript
import process from 'node:process';

process.on('warning', (warning) => {
  console.warn(warning.name);    // 'Warning'
  console.warn(warning.message); // 'Something happened!'
  console.warn(warning.code);    // 'MY_WARNING'
  console.warn(warning.stack);   // Stack trace
  console.warn(warning.detail);  // 'This is some additional information'
});
```

Si warning es un Error, el argumento options es ignorado.


### process.emitWarning(warning[, type[, code]][, ctor])


