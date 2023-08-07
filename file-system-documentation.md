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

- path `<string>` | `<Buffer>` | `<URL>` | `<FileHandle>` filename o FileHandle
- data `<string>` | `<Buffer>`
- options `<Object>` | `<string>`
  - encoding `<string>` | `<null>` Default utf8
  - mode `<integer>` Default 0o666
  - flag `<string>` Default 'a'
- return `<string>` Completa con undefined si tiene éxito.



Agrega asíncronamente a un fichero, creando el fichero si no existe aún. data puede ser un string o un Buffer.

Si options es un string se toma como el encoding

La opción mode sólo afecta a los nuevos ficheros creados.

El path debe ser especificado como un FileHandle que ha sido abierto como append.


### fsPromises.chmod(path, mode)

- path `<string>` | `<Buffer>` | `<URL>`
- mode `<string>` | `<integer>`
- return `<Promise>` Completa con undefined si tiene éxito


Cambia los permisos de un fichero


### fsPromises.chown(path, uid, gid)

- path `<string>` | `<Buffer>` | `<URL>`
- uid `<integer>`
- gid `<integer>`
- return `<Promise>` Completa con undefined si tiene éxito.


Cambia la propiedad del fichero.


### fsPromises.copyFile(src, dest[, mode])

- src `<string>` | `<Buffer>` | `<URL>` filename del recurso a copiar
- dest `<string>` | `<Buffer>` | `<URL>` filename del destino al que copiar
- mode `<integer>` Modificadores opcionales que especifican el comportamiento de la operación de copia. Es posible crear una máscara bit a bit o dos o más valores (ej. fs.constants.COPYFILE_EXCL | fs.constants.COPYFILE_FICLONE) Default 0
  - fs.constants.COPYFILE_EXCL La operación de copia fallará si dest ya existe.
  - fs.constants.COPYFILE_FICLONE La operación de copia tratará de crear un reflink copy-on-write. Si la plataforma no soporta copy-on-write, entonces el mecanismo de copy fallback es usado.
  - fs.constants.COPYFILE_FICLONE_FORCE La operación de copia tratará de crear un reflink copy-on-write. Si la pla...
- return `<Promise>` undefined si se completa con éxito


Copia asíncronamente src a dest. Por defecto, dest es sobrescrito si ya existe.

No se garantiza la atomicidad de la operación de copia. Si un error ocurre después de que el fichero de destino haya sido abierto, se tratará de eliminar el destino.

```javascript
import { copyFile, constants } from 'node:fs/promises';

try {
  await copyFile('source.txt', 'destination.txt');
  console.log('source.txt was copied to destination.txt');
} catch (err) {
  console.error('The file could not be copied');
}

// Usando COPYFILE_EXCL, la operación fallará si el destination.txt ya existe.
try {
  await copyFile('source.txt', 'destination.txt', constants.COPYFILE_EXCL);
  console.log('source.txt was copied to destination.txt');
} catch {
  console.error('The file could not be copied');
}
```


### fsPromises.cp(src, dest[, options])

### fsPromises.lchmod(path, mode)

- path `string` | `Buffer` | `URL`
- mode `integer`
- return `Promise` Completa con undefined si tiene éxito


Cambia los permisos de un symbolic link

Este método sólo se implementa en MacOS


### fsPromises.lchown(path, uid, gid)

- path `string` | `Buffer` | `URL`
- uid `integer`
- gid `integer`
- return `Promise` Completa con undefined si tiene éxito


Cambia la propiedad de un symbolic link


### fsPromises.lutimes(path, atime, mtime)

- path `string` | `Buffer` | `URL`
- atime `number` | `string` | `Date`
- mtime `number` | `string` | `Date`
- return `Promise` Completa con undefined si tiene éxito


Cambia el acceso y tiempos de modificación de un fichero de la misma manera que fsPromises.utimes(), con la diferencia que si el path se refiere a un symbolic link, entonces el link no es deferenciado, en su lugar los timestamps del symbolic link son cambiados.


### fsPromises.link(existingPath, newPath)

- existingPath `string` | `Buffer` | `URL`
- newPath `string` | `Buffer` | `URL`
- return `Promise` undefined si éxito


Crea un nuevo link desde el existingPath al newPath.


### fsPromises.lstat(path[, option])

- path `string` | `Buffer` | `URL`
- options `Object`
  - bigint `boolean` Si los valores retornados en el objeto `fs.Stats` deben ser bigint. Default false
- return `Promise` Completa con `fs.Stats` para el symbolic link dado en path


Equivalente a fsPromises.stat() con la diferencia que path se refiere a un symbolic link.


### fsPromises.mkdir(path[, options])

- path `string` | `Buffer` | `URL`
- options `Object` | `integer`
  - recursive `boolean` Default false
  - mode `string` | `integer` No soportado en Windows Default 0o777
- return `Promise` Si recursive es false completa con undefined, o el primer path del directorio creado si recursive es true


Crea asíncronamente un directorio

```javascript
import { mkdir } from 'node:fs/promises';

try {
  const projectFolder = new URL('./test/project/', import.meta.url);
  const createDir = await mkdir(projectFolder, { recursive: true });

  console.log(`Created ${createDir}');
} catch (err) {
  console.error(err.message);
}
```


### fsPromises.mkdtemp(prefix[, options])

- prefix `string`
- options `string` | `Object`
  - encoding `string` Default utf8
- return `Promise` Completa con un string que contiene el path con el nuevo directorio temporal creado


Crea un directorio temporal único. Un nombre de directorio único es generado con 6 caracteres random al final del prefix dado. Evitar usar X en el prefix.

El opcional options puede ser un string, lo que sería el encoding

```javascript
import { mkdtemp } from 'node:fs/promises'
import { join } from 'node:path';
import { tmpdir } from 'node:os';

try {
  await mkdtemp(join(tmpdir(), 'foo-'));
} catch (err) {
  console.error(err);
}
```


### fsPromises.open(path, flags[, mode])

- path `string` | `Buffer` | `URL`
- flags `string` | `number` Default r
- mode `string` | `integer` file mode. Default 0o666 (r y w)
- return `Promise` FileHandle


Abre un FileHandle


### fsPromises.opendir(path[, options])

- path `string` | `Buffer` | `URL`
- options `Object`
  - encoding `string` | `null` Default utf8
  - bufferSize `number` Número de entradas de directorio guardadas internamente cuando se lee desde el directorio. Mayor número es mejor rendimiento pero más memoria usada. Default 32
  - recursive `boolean` Default false
- return `Promise` Completa con un fs.Dir


Abre un directorio asíncronamente para escaneo iterativo.

Crea un fs.Dir, que contiene todas las funciones de lectura y limpieza del directorio.

La opción encoding se setea para el path.

```javascript
import { opendir } from 'node:fs/promises';

try {
  const dir = await opendir('./');
  for await (const dirent of dir) {
    console.log(dirent.name);
  }
} catch (err) {
  console.error(err);
}
```

Se cierra automáticamente.


### fsPromises.readdir(path[, options])

- path `string` | `Buffer` | `URL`
- options `string` | `Object`
  - encoding `string` Default utf8
  - withFileTypes `boolean` Default false
  - recursive `boolean` false
- return `Promise` Completa con un array de nombres de ficheros en el directorio excluyendo '.' y '..'


Lee los contenidos de un directorio

Si options.withFileTypes es true, el array resultante contendrá objetos fs.Dirent

```javascript
import { readdir } from 'node:fs/promises';

try {
  const files = await readdir(path);
  for (const file of files) {
    console.log(file);
  }
} catch (err) {
  console.error(err);
}
```


### fsPromises.readFile(path[, options])

- path `string` | `Buffer` | `URL` | `FileHandle` filename o FileHandle
- options `Object` | `string`
  - enconding `string` | `null` Default null
  - flag `string` Default r
  - signal `AbortSignal` Permite abortar un readFile en progreso
- return `Promise` Contenido del fichero


Asíncronamente lee el contenido completo de un fichero.

Cuando path es un directorio, depende del SO para su comportamiento. En FreeBSD se retorna una representación del directorio, en el resto rejected.

```javascript
import { readFile } from 'no:fs/promises';

try {
  const filePath = new URL('./package.json', import.meta.url);
  const contents = await readFile(filePath, { encoding: 'utf8' });
  console.log(contents);
} catch (err) {
  console.error(err.message);
}
```

Se puede abortar la lectura en proceso con un AbortSignal. Rejected si ocurre con un AbortError:

```javascript
import { readFile } from 'node:fs/promises';

try {
  const controller = new AbortController();
  const { signal } = controller;
  const promise = readFile(fileName, { signal });

  // Abortar el request antes de que la promesa se complete
  controller.abort();

  await promise;
} catch (err) {
  console.error(err);
}
```


### fPromises.readlink(path[, options])

- path `string` | `Buffer` | `URL`
- options `string` | `Object`
  - encoding `string` Default utf8
- return `Promise` linkString


Lee el contenido de un symbolic link referido al path. 


### fsPromises.realpath(path[, options])

- path `string` | `Buffer` | `URL`
- options `string` | `Object`
  - encoding `string` Default utf8
- return `Promise` Path resuelto


Determina el lugar de path usando la misma semántica de fs.realpath.native().

Sólo paths que puedan ser convertidas a utf8


### fsPromises.rename(oldPath, newPath)

- oldPath `string` | `Buffer` | `URL`
- newPath `string ` | `Buffer` | `URL`
- return `Promise` undefined


Renombra oldPath a newPath


### fsPromises.rmdir(path[, options])

- path `string` | `Buffer` | `URL`
- options `Object`
  - maxRetries `integer` Si un EBUSY, EMFILE, ENFILE, ENOTFILE o EPERM error es econtrado, Node.js vuelve a probar tras esperar retryDelay ms. Número de reintentos. Ignorado si recursive es true. Default 0
  - recursive `boolean` Si es true, borra recursivamente. En este modo reintenta al fallar. Default false. Deprecated
  - retryDelay `integer` La cantidad de tiempo en ms para esperar entre reintentos. Default 100.
- return `Promise` undefined


Borra el directorio identificado por path

Usar fsPromises.rmdir() en un fichero (no un dir) resulta en un ENOENT error en windows y un ENOTDIR en el resto.

Para obtener un comportamiento similar rm -rf en Unix usar fsPromises.rm() con las opciones `{ recursive: true, force: true }`


### fsPromises.rm(path[, options])

- path `string` | `Buffer` | `URL`
- options `Object`
  - force `boolean` Si es true las excepciones serán ignoradas si el path no existe. Default true
  - maxRetries `integer` Default 0
  - recursive `boolean` Default false
  - retryDelay `integer` Default 100
- return `Promise` undefined


### fsPromises.stat(path[, options])

- path `string` | `Buffer` | `URL`
- options `Object`
  - bigint `boolean` Default false
- return `Promise` fs.Stats


### fsPromises.statfs(path[, options])

- path `string` | `Buffer` | `URL`
- options `Object`
  - bigint `boolean` Default false
- return `Promise` fs.StatFs.



### fsPromises.symlink(target, path[, type])

- target `string` | `Buffer` | `URL`
- path `string` | `Buffer` | `URL`
- type `string` Default 'file'
- return `Promise` undefined


Crea un symbolic link

El argumento type es sólo usado en Windows, puede ser dir, file o junction.


### fsPromises.truncate(path[, len])

- path `string` | `Buffer` | `URL`
- len `integer` Default 0
- return `Promise` undefined


Trunca (acorta o extiende) el contenido de path en len bytes


### fsPromises.unlink(path)

- path `string` | `Buffer` | `URL`
- return `Promise` undefined


Si path es un symbolic link es removido sin afectar a directorios o ficheros. Si no es un symbolic link, el fichero se borra.


### fsPromises.utimes(path, atime, mtime)

- path `string` | `Buffer` | `URL`
- atime `number` | `string` | `Date`
- mtime `number` | `string` | `Date`
- return `Promise` undefined


Cambia los timestamps de un objeto referenciado en path

Los argumentos atime y mtime siguen las reglas:
- Los valores pueden ser números que representen fechas en Unix, Date o string numérico como `123456789.0`.
- Si el valor no puede ser convertido en número o es NaN, Infinity o -Infinity, se lanza un Error


### fsPromises.watch(filename[, options])

- filename `string` | `Buffer` | `URL`
- options `Object` | `string`
  - persistent `boolean` Indica si el proceso debe continuar corriendo mientras que los ficheros son seguidos. Default true
  - recursive `boolean` Indica si se debe mirar también subdirectorios. Default false.
  - encoding `string` Default utf8
  - signal `AbortSignal`
- return `AsyncIterable`
  - eventType `string` Tipo de cambio
  - filename `string` | `Buffer` El nombre del fichero cambiado


```javascript
const { watch } = require('node:fs/promises');

const ac = new AbortController();
const { signal } = ac;
setTimeout(() => ac.abort(), 10000);

(async () => {
  try {
    const watcher = watch(__filename, { signal });
    for await (const event of watcher)
      console.log(event);
  } catch (err) {
    if (err.name === 'AbortError')
      return;
    throw err;
  }
})();
```


### fsPromises.writeFile(file, data[, options])

- file `string` | `Buffer` | `URL` | `FileHandle` filename o FileHandle
- data `string` | `Buffer` | `TypedArray` | `DataView` | `AsyncIterable` | `Iterable` | `Stream`
- options `Object` | `string`
  - encoding `string` | `null` Default utf8
  - mode `string` Default 0o666
  - flag `string` Default `w`
  - signal `AbortController`
- return `Promise` undefined


Asíncronamente escribe datos a un fichero, reemplazando el fichero si existe previamente.

encoding es ignorado si data es un Buffer.

Si options es un string especifica el encoding

mode sólo afecta a los nuevos ficheros creados.

No es seguro llamar varias veces a fsPromises.writeFile sin esperar a las promesas

```javascript
import { writeFile } from 'node:fs/promises';
import { Buffer } from 'node:buffer';

try {
  const controller = new AbortController();
  const { signal } = controller;
  const data = new Uint8Array(Buffer.from('Hello'));
  cosnt promise = writeFile('message.txt', data, { signal });

  controller.abort();

  await promise;
} catch (err) {
  console.error(err);
}
```

### fsPromises.contants

```typescript
Object
```

Retorna un objeto que contiene contantes de uso común para operaciones de sistema. El objeto es el mismo que fs.contants.


## Callback API

El api de callbacks realiza todas las operaciones asíncronamente, sin bloquear el event loop, y entonces invoca un callback cuando ha completado y tiene error.

El callback api usa el threadpool de Node.js para realizar operaciones con ficheros fuera del thread del event loop. Estas operaciones no están sincronizadas ni son threadSafe. Cuidado con corrupción de datos.


### fs.access(path[, mode], callback)

```typescript
type args = {
  path: string | Buffer | URL;
  mode: integer = fs.constants.F_OK;
  callback: Function;
}
```

Testea los permisos del usuario para un fichero o directorio especificado por path.

```javascript
import { access, constants } from 'node:fs';

const file = 'package.json';

access(file, constants.F_OK, (err) => {
  console.log(`${file} ${err ? 'does not exist' : 'exist'} `);
});

access(file, constants.R_OK, err => console.log(`${file} ${err ? 'is not readable' : 'is readable'));

access(file, constants.W_OK, err => console.log(`${file} ${err ? 'is not writable' : 'is writable'));

access(file, contants.R_OK | constants.W_OK, err => console.log(`${file} ${err ? 'is not' : 'is' } readable and writable`);
```

No usar antes de fs.open, fs.readFile o fs.writeFile.


### fs.appendFile(path, data[, options], callback)

```typescript
type args = {
  path: string | Buffer | URL | number; // filename o file descriptor
  data: string | Buffer;
  options: {
    encoding: string = 'utf8';
    mode: string = '0o666';
    flag: string = 'a';
  } | string;
  callback: Function;
}
```

Asíncronamente agrega datos a un fichero, creando el fichero si no existe. Data puede ser string o Buffer.

```javascript
import { appendFile } from 'node:fs';

appendFile('message.txt', 'data to append', (err) => {
  if (err) throw err;
  console.log('The data to append was appended to file');
});
```

Si optinons es un string se toma como el encoding

```javascript
import { appendFile } from 'node:fs';

appendFile('message.txt', 'data to append', 'utf8', callback);
```

path puede ser especificado como un file descriptor numérico que ha sido abierto para append (usando fs.open() o fs.openSync()). El file descriptor no se cerrará automáticamente

```javascript
import { open, close, appendFile } from 'node:fs';

function closeFd(fd) {
  close(fd, (err) => {
    if (err) throw err;
  });
}

open('message.txt', 'a', (err, fd) => {
  if (err) throw err;
  try {
    appendFile(fd, 'data to append', 'utf8', err => {
      closeFd(fd);
      if (err) throw err;
    });
  } catch (err) {
    closeFd(fd);
    throw err;
  }
});
```


### fs.chmod(path, mode, callback)

```typescript
type args = {
  path: string | Buffer | URL;
  mode: string | integer;
  callback: Function;
}
```

Asíncronamente cambia los permisos para un fichero. El callback solo recibe un Error.

```javascript
import { chmod } from 'node:fs';

chmod('my_file.txt', 0o777, err => {
  if (err) throw err;
  conosle.log('The permissions have been changed');
});
```


### File modes

|      Constant        | Octal |      Description        |
| -------------------- | ----- | ----------------------- |
| fs.constants.S_IRSUR | 0o400 | Read by owner           |
| fs.constants.S_IWUSR | 0o200 | write by owner          |
| fs.constants.S_IXUSR | 0o100 | execute/search by owner |
| fs.constants.S_IRGRP | 0o40  | read by group           |
| fs.constants.S_IWGRP | 0o20  | write by group          |
| fs.constants.S_IXGRP | 0o10  | execute/search by group |
| fs.constants.S_IROTH | 0o4   | read by others          |
| fs.constants.S_IWOTH | 0o2   | write by others         |
| fs.constants.S_IXOTH | 0o1   | execute/search by others|


Un modo más sencillo de usar mode es usar unsa secuencia de 3 octales (ej 765). El primero se refiere al propietario, el segundo al grupo y el tercero a otros.

| Number | Description             |
| ------ | ----------------------- |
|   7    | read, write and execute |
|   6    | read and write          |
|   5    | read and execute        |
|   4    | read only               |
|   3    | write and execute       |
|   2    | write only              |
|   1    | execute only            |
|   0    | no permissions          |


### fs.chown(path, uid, gid, callback)

```typescript
type args = {
  path: string | Buffer | URL;
  uid: integer;
  gid: integer;
  callback: Function;
}
```

Asíncronamente cambia el propietario de un fichero.


### fs.close(fd[, callback])

```typescript
type args = {
  fs: integer;
  callback: Function;
}
```

Cierra un file descriptor.

Llamar a fs.close() en un file descriptor que esté actualmente en uso puede llevar a un comportamiento no deseado.


### fs.copyFile(src, dest[, mode], callback)

```typescript
type args = {
  src: string | Buffer | URL; // filename del recurso a copiar
  dest: string | Buffer | URL; // filename destino
  mode: integer; // modificador
  callback: Function;
}
```

Asíncronamente copia src to dest. Por defecto dest es sobreescrito.

mode es un integer opcional que especifica el comportamiento de la operación de copia. 

```javascript
import { copyFile, constants } from 'node:fs';

function callback(err) {
  if (err) throw err;
  console.log('hecho');
}

// destination.txt será creado o sobresscrito por defecto
copyFile('source.txt', 'destination.txt', callback);

// Usando COPYFILE_EXCL, la operación fallará si destination.txt ya existe.
copyFile('source.txt', 'destination.txt', constants.COPYFILE_EXCL, callback);
```


### fs.cp(src, dest[, options], callback)
`experimental`

```typescript
type args = {
  src: string | URL; // path del recurso
  dest: string | URL; // path del destino a copiar
  options: {
    deference: boolean = false; // deference symlinks
    errorOnExit: boolean = false; // Suelta error al salir si force es false
    filter: (src: string, dest: string) => boolean | Promise<boolean>;
    force: boolean; // Sobreescribe el fichero o dir.
    mode: integer = 0;
    preserveTimestamps: boolean = false; // Si es true los timestamps de src se mantienen.
    recursive: boolean = false;
    verbatimSymlinks: boolean = false;
  callback: Function;
}
```


### fs.createReadStream(path[, options])

```typescript
type args = {
  path: string | Buffer | URL;
  options?: string | {
    flags?: string = 'r';
    encoding?: string;
    fs?: integer | FileHandle;
    mode?: integer = 0o666;
    autoClose?: boolean = true;
    emitClose?: boolean = true;
    start?: integer;
    end?: integer = Infinity;
    highWaterMark?: integer = 64 * 1024;
    fs?: Object | null;
    signal?: AbortSignal | null;
  }
} => fs.ReadStream
```

```javascript
import { createReadStream } from 'node:fs';

const stream = createReadStream('/dev/input/event0');
setTimeout(() => {
  stream.close();
  stream.push(null);
  stream.read(0);
}, 100);
```

```javascript
import { createReadStream } from 'node:fs';

createReadStream('sample.txt', { start: 90, end: 99 });
```


### fs.createWriteStream(path[, options])


... resto igual que promises


## Synchronous API

... Igual al resto

## Objetos comunes

### Class: fs.Dir

Representa un directorio.

```javascript
import { opendir } from 'node:fs/promises';

try {
  const dir = await opendir('./');
  for await (const dirent of dir) {
    console.log(dirent.name);
  }
} catch (err) {
  console.error(err);
}
```


