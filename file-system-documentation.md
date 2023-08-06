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

- data <string> | <Buffer> | <TypedArray> | <DataView> | <AsyncIterable> | <Iterable> | <Stream>
- options <Object> | <string>
  - encoding <string> | <null> Default: 'utf8'
- return <Promise> Se devuelve undefined hasta que se completa

Alias de filehandle.writeFile()

Cuando se opera con file handlers, el modo no puede ser modificado del que fue seteado con fsPromise.open(), por lo que esto es equivalente a filehandle.writeFile().

### filehandle.chmod(mode)

- mode <integer> El bit mask mode del fichero
- return <Promise> undefined hasta completado

Modifica los permisos del fichero.

### filehandle.chown(uid, gid)

- uid <integer> El user id del nuevo propietario del fichero
- gid <integer> El group id del nuevo grupo propietario del fichero
- return <Promise> undefined hasta completado

Cambia la propiedad del fichero. Wrapper para chown

### filehandle.close()

- return <Promise> undefined hasta completado

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

- options <Object>
  - encoding <string> Default null
  - autoClose <boolean> Default true
  - emitClose <boolean> Default true
  - start <integer>
  - end <integer> Default Infinity
  - highWaterMark <integer> Default 64 * 1024
- return <fs.ReadStream>

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

- options <Object>
  - encoding <string> Default 'utf8'
  - autoClose <boolean> Default true
  - emitClose <boolean> Default true
  - start <integer>
- return <fs.WriteStream>

options puede incluir un start option que permite escribir los data en alguna posición pasado el principio del fichero, permite valores en un rango de [0, max integer]. Modificar un fichero en lugar de reemplazarlo puede requrerir un open flag para setear a r+ en lugar de r. El encoding puede ser cualquiera de los permitidos por Buffer.

Si autoClose está en true (por defecto) en 'error' o 'finish' el file descriptor será cerrado automáticamente. Si autoClose es false, entonces el file descriptor no se cerrará, incluso si hay un error. Es responsabilidad de la aplicación cerrarlo y asegurarse que no hay file descriptor leak.

Por defecto, el stream emitirá un evento 'close' después de ser destruido. Setear emitClose a false cambia este comportamiento.

### filehandle.datasync()

- return <Promise> undefined hasta ser completado.

Fuerza a todas las operaciones I/O encoladas asociadas con un fichero al estado de finalización de I/O sincronizada con el SO.

Diferente a filehanlde.sync, este método no borra metadatos modificados.

### filehandle.fd

- <number> El file descriptor numérico manejado por el objeto <FileHandler>

### filehandle.read(buffer, offset, length, position)

- buffer <Buffer> | <TypedArray> | <DataView> Un buffer que será rellenado con los datos leídos del fichero.
- offset <integer> El lugar en el buffer desde el que iniciar a rellenar.
- length <integer> El número de bytes a leer
- position <integer> | <null> El lugar desde el que comenzar a ller en el fichero. Si es null, los dtos se leerán desde el lugar actual en el fichero, y la posición será actualizada. Si position es un integer, el file position actual permanecerá sin cambios.
- return <Promise> Hasta completarse retorna un objeto con dos props:
  - bytesRead <integer> El número de bytes leídos
  - buffer <Buffer> | <TypedArray> | <DataView> Una referencia pasada en el argument buffer.
 
Lee datos desde un fichero y los guarda en un buffer dado.

Si el fichero no es modificado concurrentemente, el end-of-file se llega cuando el número de bytes leídos son 0.

### filehandle.read([options])

- options <Object>
  - buffer <Buffer> | <TypedArray> | <DataView> Un buffer que será rellenado con los datos leidos. Default Buffer.alloc(16384)
  - offset <integer> El lugar en el buffer desde el que empezar a rellenar. Default 0
  - length <integer> El número de bytes a leer. Default buffer.byteLength - offset.
  - positon <integer> | <null> El lugar desde el que comenzar a leer datos desde el fichero. Si es null, los datos se leerán desde la actual posición dentro del fichero, y position será actualizado. Si position es un integer, la posición actual en el fichero permanecerá sin cambios. Default null.
- return <Promise> retorna un objeto con dos props hasta completarse la tarea:
  - bytesRead <integer> El número de bytes a leer
  - buffer <Buffer> | <TypedArray> | <DataView> Una referencia pasada por el argumento buffer.

Lee datos desde un fichero y los almacena en un buffer dado.

Si el fichero no está siendo modificado a la vez, se llega a end-of-file cuando el número de bytes leídos sea 0

### filehandler.read(buffer[, options])

- buffer <Buffer> | <TypedArray> | <DataView> Un buffer que será rellenado con los datos leídos del fichero.
- options <Object>
  - offset <integer> La posición en el buffer desde el que se va a comenzar a rellenar. Default 0
  - length <integer> El número de bytes a leer. Default buffer.byteLength - offset
  - position <integer> El lugar desde el que comenzar a leer datos del fichero. Si es null, los datos serán leídos desde la actual posición dentro del fichero, y la posición será actualizada. Si position es un integer, el position actual permanecerá sin cambios. Default null.
 - return <Promise> Hasta completarse retorna un objeto con dos props:
   - bytesRead <integer> El número de bytes a leer
   - buffer <Buffer> | <TypedArray> | <DataView> Una referencia al buffer pasado por argumento.
  
Lee datos desde un fichero y los guarda a un buffer dado.

Si el fichero no está siendo modificado a la vez, un end-of-file se obtendrá cuando el número de bytes leídos sea 0.

### filehandle.readableWebStream(options) __Experimental__

- options <Object>
  - type <string> | <undefined> Define si abrir un stream normal o de bytes. Default undefined
- return <ReadableStream>

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

