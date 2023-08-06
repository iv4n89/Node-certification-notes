## File system

El módulo node:fs habilita la interacción con sistema de ficheros usando el estándard POSIX.

Para usar el api basado en promesas:

```javascript
import * as fs from 'node:fs/promises';
```

Para usar el api de callbacks y sync:

```javascript
import * as fs from 'node:fs';
```

Todas las operaciones de fs tienen formas síncrona, callback y basado en promesas, y son accesibles tanto desde CJS com EMS.

## Ejemplo de promise

Las operaciones basadas en promesas retornan una promesa cuando realizan la operación asignada

```javascript
import { unlink } from 'node:fs/promises';

try {
  await unlink('/tmp/hello');
  console.log('successfully deleted /tmp/hello');
} catch (err) {
  console.error('there was an error:', error.message);
}
```

## Ejemplo de callback

La forma de callback toma un callback que es llamado al completarse como su último argumento e invoca la operación de manera asíncrona. Los argumentos pasados al callback dependen del método, pero el primero argumento siempre está reservado para una exception. Si la operación es completada con éxito, entonces el primer argumento será null o undefined.

```javascript
import { unlink } from 'node:fs';

unlink('/tmp/hello', (err) => {
  if (err) throw err;
  console.log('successfully deleted /tmp/hello');
});
```

La versión basada en callbacks de node:fs es preferible sobre la versión de promises cuando se necesita más rendimiento (tiempo de ejecución y memoria)

## Ejemplo síncrono

El api síncrono bloquea el event loop de node.js y JS hasta que la operación se completa. Las excepciones son lanzadas inmediatamente y pueden ser tomadas usando un try catch, o pueden ser propagadas.

```javascript
import { unlinkSync } from 'node:fs';

try{
  unlink('/tmp/hello');
  console.log('successfully deleted /tmp/hello');
} catch (err) {
  // handle the error
}
```

## Promises API

El API fs/promises provee métodos asíncronos que retornan promesas.

Usa el subyacente threadpool para realizar operaciones sobre el sistema de ficheros fuera del event loop. Estas operaciones no están sincronizadas ni son threadsafe. Se debe tener cuidado cuando se realizan varias operaciones a la vez sobre el mismo fichero ya que puede darse corrupción de ficheros.

## Class: FileHandle

Un <FileHandle> es un wrapper para un file descriptor numérico.

Las instancias son creadas por fsPromises.open().

Todos los FileHandle son EventEmitters

Si un FileHandler no es cerrado usando fileHandle.close(), tratará automáticamente de cerrar el file descriptor y emitir un warning al process, ayudando a prevenir memory leaks. No confiar en este comportamiento ya que puede fallar y el fichero puede no cerrarse. Siempre cerrar el FileHandler. Node.js podría modificar este comportamiento en un futuro.

### Event: 'close'

Emitido cuando el FileHandler es cerrado y no se volverá a usar.

### filehandle.appendFile(data[, options])

- data `<string>` | `<Buffer>` | `<TypedArray>` | `<DataView>` | `<AsyncIterable>` | `<Iterable>` | `<Stream>`
- options `<Object>` | `<string>`
  - encoding `<string>` | `<null>` Default: 'utf8'
- return `<Promise>` Se devuelve undefined hasta que se completa

Alias de filehandle.writeFile()

Cuando se opera con file handlers, el modo no puede ser modificado del que fue seteado con fsPromise.open(), por lo que esto es equivalente a filehandle.writeFile().

### filehandle.chmod(mode)

- mode `<integer>` El bit mask mode del fichero
- return `<Promise>` undefined hasta completado

Modifica los permisos del fichero.

### filehandle.chown(uid, gid)

- uid `<integer>` El user id del nuevo propietario del fichero
- gid `<integer>` El group id del nuevo grupo propietario del fichero
- return `<Promise>` undefined hasta completado

Cambia la propiedad del fichero. Wrapper para chown

### filehandle.close()

- return `<Promise>` undefined hasta completado

Cierra el file handler después de esperar por cualquier operación pendiente de ser completada

```javascript
import { open } from 'node:js/promises';

let filehandle;

try {
  filehandle = await open('thefile.txt', 'r');
} finally {
  await filehandle?.close();
}
```

### filehandle.createReadStream([options])

- options `<Object>`
  - encoding `<string>` Default null
  - autoClose `<boolean>` Default true
  - emitClose `<boolean>` Default true
  - start `<integer>`
  - end `<integer>` Default Infinity
  - highWaterMark `<integer>` Default 64 * 1024
- return `<fs.ReadStream>`

Distinto al default highWaterMark para un stream.Readable, el stream retornado por este método tiene por defecto un highWaterMark de 64 kb.

options puede incluir start y end para leer un rango de bytes desde el fichero en lugar del fichero completo. Ambos start y end son inclusivos y contando desde 0, permitiendo valores en un rango [0, Number.MAX_SAFE_INTEGER]. Si start es omitido o undefined, cerateReadStream() lee secuencialmente desde la posición actual del fichero. El encoding puede ser cualquiera de los aceptados por Buffer.

Si el FileHandle apunta a un caracter device que solo soporta lectura en bloques (como keyboards o sound card), las operaciones de lectura no terminarán hasta que los datos estén disponibles. Esto previene al proceso de salir y al stream de cerrarse manualmente.

Por defecto, el stream emitirá un evento close después de que haya sido destruido. Setear emitClose a false cambiará este comportamiento.

```javascript
import { open } from 'node:fs/promises';

const fd = await open('/dev/input/event0');
// Crea un stream desde algún character device.
const stream = fd.createReadStream();
setTimeout(() => {
  stream.close(); // Esto puede no cerrar el stream
  // Artificialmente haciendo un end-of-stream, ya que el recurso subyacente ha indicado end-of-file por sí mismo, permitiendo al stream cerrarse.
  // Esto no cierra operaciones de cierre pendientes, y si hay alguna operación, el proceso puede aún no estar disponible para cerrarse con éxito hasta que acaba.
  stream.push(null);
  stream.read(0);
}, 100);
```

Si autoClose es false, entonces el file descriptor no se cerrará, incluso si hay un error. Es la responsabilidad de la aplicación cerrarlo y asegurarse que no hay file descriptor leak. Si autoClose es true (por defecto es así), en 'error' o 'end' el file descriptor se cerrará automáticamente.

Un ejemplo para leer los últimos 10 bytes de un fichero que es 100 bytes de largo:

```javascript
import { open } from 'node:fs/promises';

const fd = await open('sample.txt');
fs.createReadStream({ start: 90, end: 99 });
```

### filehandle.createWriteStream([options])

- options `<Object>`
  - encoding `<string>` Default 'utf8'
  - autoClose `<boolean>` Default true
  - emitClose `<boolean>` Default true
  - start `<integer>`
- return `<fs.WriteStream>`

options puede incluir un start option que permite escribir los data en alguna posición pasado el principio del fichero, permite valores en un rango de [0, max integer]. Modificar un fichero en lugar de reemplazarlo puede requrerir un open flag para setear a r+ en lugar de r. El encoding puede ser cualquiera de los permitidos por Buffer.

Si autoClose está en true (por defecto) en 'error' o 'finish' el file descriptor será cerrado automáticamente. Si autoClose es false, entonces el file descriptor no se cerrará, incluso si hay un error. Es responsabilidad de la aplicación cerrarlo y asegurarse que no hay file descriptor leak.

Por defecto, el stream emitirá un evento 'close' después de ser destruido. Setear emitClose a false cambia este comportamiento.

### filehandle.datasync()

- return `<Promise>` undefined hasta ser completado.

Fuerza a todas las operaciones I/O encoladas asociadas con un fichero al estado de finalización de I/O sincronizada con el SO.

Diferente a filehanlde.sync, este método no borra metadatos modificados.

### filehandle.fd

- `<number>` El file descriptor numérico manejado por el objeto <FileHandler>

### filehandle.read(buffer, offset, length, position)

- buffer `<Buffer>` | `<TypedArray>` | `<DataView>` Un buffer que será rellenado con los datos leídos del fichero.
- offset `<integer>` El lugar en el buffer desde el que iniciar a rellenar.
- length `<integer>` El número de bytes a leer
- position `<integer>` | `<null>` El lugar desde el que comenzar a ller en el fichero. Si es null, los dtos se leerán desde el lugar actual en el fichero, y la posición será actualizada. Si position es un integer, el file position actual permanecerá sin cambios.
- return `<Promise>` Hasta completarse retorna un objeto con dos props:
  - bytesRead `<integer>` El número de bytes leídos
  - buffer `<Buffer>` | `<TypedArray>` | `<DataView>` Una referencia pasada en el argument buffer.


Lee datos desde un fichero y los guarda en un buffer dado.

Si el fichero no es modificado concurrentemente, el end-of-file se llega cuando el número de bytes leídos son 0.

### filehandle.read([options])

- options `<Object>`
  - buffer `<Buffer>` | `<TypedArray>` | `<DataView>` Un buffer que será rellenado con los datos leidos. Default Buffer.alloc(16384)
  - offset `<integer>` El lugar en el buffer desde el que empezar a rellenar. Default 0
  - length `<integer>` El número de bytes a leer. Default buffer.byteLength - offset.
  - positon `<integer>` | `<null>` El lugar desde el que comenzar a leer datos desde el fichero. Si es null, los datos se leerán desde la actual posición dentro del fichero, y position será actualizado. Si position es un integer, la posición actual en el fichero permanecerá sin cambios. Default null.
- return `<Promise>` retorna un objeto con dos props hasta completarse la tarea:
  - bytesRead `<integer>` El número de bytes a leer
  - buffer `<Buffer>` | `<TypedArray>` | `<DataView>` Una referencia pasada por el argumento buffer.

Lee datos desde un fichero y los almacena en un buffer dado.

Si el fichero no está siendo modificado a la vez, se llega a end-of-file cuando el número de bytes leídos sea 0

### filehandler.read(buffer[, options])

- buffer `<Buffer>` | `<TypedArray>` | `<DataView>` Un buffer que será rellenado con los datos leídos del fichero.
- options `<Object>`
  - offset `<integer>` La posición en el buffer desde el que se va a comenzar a rellenar. Default 0
  - length `<integer>` El número de bytes a leer. Default buffer.byteLength - offset
  - position `<integer>` El lugar desde el que comenzar a leer datos del fichero. Si es null, los datos serán leídos desde la actual posición dentro del fichero, y la posición será actualizada. Si position es un integer, el position actual permanecerá sin cambios. Default null.
 - return `<Promise>` Hasta completarse retorna un objeto con dos props:
   - bytesRead `<integer>` El número de bytes a leer
   - buffer `<Buffer>` | `<TypedArray>` | `<DataView>` Una referencia al buffer pasado por argumento.
  
Lee datos desde un fichero y los guarda a un buffer dado.

Si el fichero no está siendo modificado a la vez, un end-of-file se obtendrá cuando el número de bytes leídos sea 0.

### filehandle.readableWebStream(options) __Experimental__

- options `<Object>`
  - type `<string>` | `<undefined>` Define si abrir un stream normal o de bytes. Default undefined
- return `<ReadableStream>`

Retorna un readableStream que puede ser usado para leer los datos del fichero.

Un error será lanzado si este método es llamado más de una vez o si es llamado después de que FileHandle es cerrado o esté cerrando.

```javascript
import { open } from 'node:fs/promises';

const file = open('./some/file/to/read');

for await (const chunk of file.readableWebStream()) {
  console.log(chunk);
}

await file.close();
```

Mientras que el ReadableStream lee el fichero hasta acabarlo, no cerrará el FileHandle automáticamente. Se puede llamar manualmente aún al método fileHandle.close().

### filehandle.readFile(options)

- options `<Object>` | `<string>`
  - encoding `<string>` | `<null>` default null
  - signal `<AbortSignal>` Permite abortar un readFile en progreso
- return `<Promise>` Se ejecuta tras la lectura correcta con el contenido del fichero. Si no tiene enconding especificado (usando options.encoding), los datos retornados serán un Buffer. De otra manera, serán string.


Lee asíncronamente el fichero completo.

Si options es un string entonces especifica el encoding.

El FileHandle tiene que soportar lectura.

Si una o más llamadas a fileHandle.read() a un fichero son realizadas y luego se llama a filehandle.readFile(), los datos serán leídos desde la posición actual hasta el final del fichero. No permite la lectura desde el inicio del fichero.


### filehandle.readLines([options])

- options `<Object>`
  - encoding `<string>` Default null
  - autoClose `<boolean>` Default true
  - emitClose `<boolean>` Default true
  - start `<integer>`
  - end `<integer>` Default infinity
  - highWaterMark `<integer>` Default 64 * 1024
- return `<readline.InterfaceConstructor>`


Método de ayuda para crear un interfaz de readLine y strimear sobre el fichero. 

```javascript
import { open } from 'node:fs/promises';

const file = await open('./some/file/to/read');

for await (const line of file.readLines()) {
  console.log(line);
}
```


### filehandle.readv(buffers[, position])

- buffers `<Buffer[]>` | `<TypedArray[]>` | `<DataView[]>`
- position `<integer>` | `<null>` Offset desde el inicio del fichero desde el que los datos deben ser leídos. Si position no es un number, los datos serán leídos desde la posición actual. Default null
- return `<Promise>` Objeto con dos props:
  - bytesRead `<integer>` El número de bytes leídos.
  - buffers `<Buffer[]>` | `<TypedArray[]>` | `<DataView[]>` Propiedad que contiene una referencia a los buffers pasados por argumento.


### filehandle.stat([options])

- options `<Object>`
  - bigint `<boolean>` Indica si el retorno debe ser un bigint Default fasle.
- return `<Promise>` Completo con un `<fs.Stats>` para el fichero


### filehandle.sync()

- return `<Promise>` Completo con undefined en caso de éxito.


Pide que todos los datos para el file descriptor abierto sea borrado en el device. Depende del sistema operativo y el device.


### filehandle.truncate(len)

- len `<integer>` Default 0
- return `<Promise>` Completa con undefined en caso de éxito.


Trunca el fichero.

Si el fichero era más largo de len bytes, sólo los primeros len bytes serán retornados en el fichero.

```javascript
import { open } from 'node:fs/promises';

let filehandle = null;
try {
  filehandle = await open('temp.txt', 'r+');
  await filehandle.truncate(4);
} finally {
  await filehandle?.close();
}
```

Si el fichero es más corto que len, es extendido, y la parte extendida se rellena con '/0';

Si len es negativo se usará 0 en su lugar.


### filehandle.utimes(atime, mtime)

- atime `<number>` | `<string>` | `<Date>`
- mtime `<number>` | `<string>` | `<Date>`
- return `<Promise>`


Cambia los timestamps del file system del objeto referenciado por FileHandle y entonces resuelve las promesas sin argumenos hasta completarse.


### filehandle.write(buffer, offset[, length[, position]])

- buffer `<Buffer>` | `<TypedArray>` | `<DataView>`
- offset `<integer>` La posición inicial del buffer desde la que se comenzarán a escribir los datos
- length `<integer>` El número de bytes desde buffer a escribir. Default buffer.byteLength - offset
- position `<integer>`| `<null>` Offset del fichero desde el que los datos de buffer serán escritos. Si position no es un number, los datos serán escritos desde la posición actual. Default null.
- return `<Promise>`


La promesa se resuelve con un objeto que contiene dos props:

- bytesWritten `<integer>` El número de bytes escritos
- buffer `<Buffer>` | `<TypedArray>` | `<DataView`> Una referencia al buffer escrito


No es seguro usar filehandle.write() múltiples veces en un mismo fichero sin esperar a la promesa ser resuelta o rechazada. Para este otro escenario, usar filehandle.createWriteStream().

En linux, la escritura posicional no funciona cuando el fichero es abierto en modo append. El kernel ignora el argument position y siempre agrega los datos al final del fichero.


### filehandle.write(buffer[, options])

- buffer `<Buffer>` | `<TypedArray>` | `<DataView>`
- optinos `<Object>`
  - offset `<integer>` Default 0
  - length `<integer>` Default buffer.byteLength - offset
  - position `<integer>` Default null
- return `<Promise>`


Escribe buffer al fichero

Similar a filehandle.write, pero esta versión toma un options opcional. Si options no es especificado, se tomarán los valores por defecto.


### filehandle.write(string[, position[, encoding]])

- string `<string>`
- position `<integer>` | `<null>` Offset desde el inicio del fichero donde los datos de string serán escritos. Si position no es un number los datos serán escritos desde la posición actual. Default null.
- encoding `<string>` El encoding esperado Default utf8
- return `<Promise>`


Escribe string a un fichero. Si string no es un string, la promesa será rechazada con un error.

La promesa es resuelta con un objeto que contiene las props:

- bytesWritten `<integer>` El número de bytes escritos
- buffer `<string>` Una referencia al string escrito


No es seguro llamar varias veces a .write sin esperar a que las promesas sean completadas.

En linux no funciona la escritura posicional.


### filehandle.writeFile(data, options)

- data `<string>` | `<Buffer>` | `<TypedArray>` | `<DataView>` | `<AsyncIterable>` | `<Iterable>` | `<Stream>`
- options `<Object>` | `<string>`
  - encoding `<string>` | `<null>` El encoding esperado Default utf8
- return `<Promise>`


Escribe asíncronamente datos a un fichero, reemplazando el fichero si ya existe. data puede ser un string, un buffer, un AsyncIterable o un Iterable. La promesa se resuelve sin argumentos (void)

Si options es un string se toma como el encoding

El FileHanlde debe soportar escritura

No es seguro usar filehandle.writeFile() múltiples veces sin esperar que las promesas se resuelvan.

Si una o más llamadas de filehandle.write() son realizadas a un filehandle y luego se llama a filehandle.writeFile(), los datos serán escritos desde la posición actual hasta el final del fichero. No siempre escribe desde el inicio del fichero.


### filehandle.writev(buffers[, option])

- buffers `<Buffer[]>` | `<TypedArray[]>` | `<DataView[]>`
- position `<integer>` | `<null>` El offset desde el inicio del fichero donde los datos del buffer serán escritos. Si position no es un número, los datos se escribirán en la posición actual. Default null.
- return `<Promise>`


Escribe un array ArrayBufferView al fichero.

La promesa se resuelve con un objeto que contiene dos props:
- bytesWritten `<integer>` El número de bytes escritos
- buffers `<Buffer[]>` | `<TypedArray[]>` | `<DataView[]>` Una referencia a buffers pasado por argumento.


No es seguro usar writev() múltiples veces si esperar que las promesas se resuelvan.

En linux la escritura posicional no funciona.


### fsPromises.access(path[, mode])

- path `<string>` | `<Buffer>` | `<URL>`
- mode `<integer>` Default fs.constants.F_OK
- return `<Promise>` undefined cuando completa


Testea los permisos del usuario para un un fichero o directorio especificado por path. El argumento mode es un integer opcional que especifica los checks de accesibilidad a ser realizados. mode debe ser el valor fs.constants.F_OK o una máscara que contiene bit a bit o cualquiera de las constantes R_OK, W_OK o X_OK.

Si acaba correcto, la promesa se resuelve sin valor. Si algo falla es rejected con un Error. 

```javascript
import { access, constants } from 'node:fs/promises';

try {
  await access('/etc/passwd', contants.R_OK | contants.W_OK);
  console.log('can access');
} catch {
  console.error('cannot access');
}
```

Usar fsPromises.access() para checkear la accesibilidad a un fichero antes de usar fsPromises.open() no está recomendado. Hacer esto introduce una race condition ya que otros procesos pueden modificar el estado del fichero entre las dos llamadas. En su lugar, se debe hacer open/read/write al fichero directamente y lidiar con el error si el fichero no es accesible.


### fsPromises.appendFile(path, data[, options])

