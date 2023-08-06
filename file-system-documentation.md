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
