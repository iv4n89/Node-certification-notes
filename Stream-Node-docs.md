## Stream

Stream es una interfaz abstracta para trabajar con datos en streaming en Node.js. El módulo stream provee un API para implementar la interfaz.

Hay muchos objetos de stream provistos por Node.js. Por ejemplo, un request a un servidor http y el proceso process.stdout son instancias de stream.

Los streams pueden ser readable, writable o ambos. Todos los streams son instancias de EventEmitter.

Para acceder al módulo de stream:

```
const stream = require('stream');
```

El módulo de stream es útil para crear nuevos tipos de instancias de streams. Usualmente no es necesario para consumir streams.

## Tipos de streams

Hay 4 tipos fundamentales de streams en Node.js:

- Writable: Streams cuyos datos pueden ser escritos (por ejemplo: fs.createWriteStream()).
- Readable: Streams desde el que se pueden acceder a los datos para leerlos (por ejemplo: fs.createReadStream()).
- Duplex: Streams que son tanto writable como readable (por ejemplo: net.Socket).
- Transform: Duplex streams que pueden modificar o transformar los datos mientras son leídos o escritos (por ejemplo: zlib.createDeflate()).

Adicionalmente, este módulo incluye las funciones de utilidad de stream.pipeline(), stream.finished() y stream.Readable.from().

## Object mode

Todos los streams creados por el API de Node.js opera exclusivamente con strings y Buffers (o Unit8Array). Sin embargo, es posible implemetar el uso de objetos de JavaScript como valor (con la excepción de null, que tiene un caso especial dentro de los streams).
Estos streams se dice que están en 'object mode'.

Se puede activar este modo usando la opción objectMode cuando un stream es creado. Tratar de pasar un stream ya creado a objectMode no es seguro.

## Buffering

Tanto los streams Writable como los Readable guardarán datos en un buffer interno que puede ser recogido usando writable.writableBuffer o readable.readableBuffer, respectivamente.

La cantidad de datos potencialmente buffered depende del highWaterMark option pasado al stream constructor. Para streams normales el highWaterMark option especifica el número total de bytes. Para streams operando en objectMode especifica el número total de objetos.

En los streams readable los datos son buffered cuando la implementación llama a stream.push(chunk). Si el consumer del Stream no llama a stream.read(), los datos estarán esperando en la cola interna hasta que sean consumidos.

Una vez el tamaño total del internal read buffer llega al punto especificado por highWaterMark, el stream temporalmente parará de leer los datos desde el recurso hasta que los datos en buffer puedan ser consumidos (esto es, el stream parará de llamar al método interno readable._read() que es usado para llenar el buffer de lectura).

Un objetivo importante del api de stream, particularmente el método stream.pipe(), es limitar el buffering de datos a niveles aceptables con lo que la diferencia de velocidad entre el recurso y el destino no sobrepase la memoria disponible.

La opción de highWaterMark es un umbral, no un límite: dictamina la cantidad de datos que un stream hace buffer antes de para a preguntar por más datos. En general, no se relaciona con la limitación de memoria. Implemenciaciones específicas de streams pueden ser elegidas para forzar límites más estrictos, pero hacer esto es opcional.

Ya que Duplex y Transform streams son Writable y Readable, cada uno mantiene dos buffers internos por separado usados para leer y escribir, permitiendo a cada lado operar independientemente del otro mientras mantiene un apropiado y eficiente flujo de datos. Por ejemplo, net.Socket instances son streams Duplex cuyo lado Readable permite condumir datos recibidos desde el socket y cuyo lado writable permite escribir datos al socket. Ya que los datos deben ser escritos al socket a una velocidad menor o mayor que los recibidos, cada lado debe operar (y buffer) independientemente del otro.

## API para condumers de streams

Casi todas las apicaciones de Node.js, no importa lo simple que sean, usan streams de alguna manera. El siguiente ejemplo del uso de streams en una aplicación de Node.js es una implementación de un servidor http:

```
const http = require('http');

const server = http.createServer((req, res) => {
  // 'req' es un http.IncomingMessage, que es un stream readable
  // 'res'' es un http.ServerResponse, que es un stream writable

  let body = '';
  // Obtener los datos como strings utf8.
  // Si el encoding no está seteado, Buffer objects serán recibidos.
  req.setEncoding('utf8')';

  // Streams readable emiten eventos de 'datos' una vez un listener es agregado.
  req.on('data', (chunk) => {
    body += chunk;
  });

  // El evento 'end' indica que todo el body ha sido recibido.
  req.on('end', () => {
    try {
      const data = JSON.parse(body);
      // Escribir de vuelta algo intersante al usuario
      res.write(typeof data);
      res.end();
    } catch (err) {
      res.statusCode = 400;
      return res.end(`error: ${err.message}`);
    }
  });
});

server.listen(1337);

// $ curl localhost:1337 -d "{}"
// object
// $ curl localhost:1337 -d "\"foo\""
// string
// $ curl localhost:1337 -d "not json"
// error: Unexpected token o in JSON at position 1
```

Los streams Writable (como res en el ejemplo) exponen métodos como write() y end() que son usados para wscribir datos al stream.

Los streams Readable usan el API de EventEmitter para notificar cuando datos están disponibles para ser leídos. Estos datos pueden ser leídos desde el stream de diferentes formas.

Ambos Writable y Readable streams usan el EventEmitter de varias maneras para comunicar el estado actual del stream.

Duplex y Transform streams son tanto Writable como Readable.

Las aplicaciones que tanto escriben como consumen datos de un stream no requieren que se implemente la interfaz de Stream directamente y por lo general no hay razón para llamar a require('stream').

## Writable streams

Los streams writable son una abstracción del destino al que los datos serán escritos.

Ejemplos de streams writable son:

- Http requests, en el cliente
- Http responses, en el server
- fs write streams
- zlib streams
- crypto streams
- TCP sockets
- child process stdin
- process.stdout, process.stderr

Algunos de estos ejemplos son streams duplex que implementan la interfaz Writable.

Todos los streams writable implementan la interfaz definida por la clase stream.Writable.

Mientras que una instancia específica de writable puede diferir de varias maneras, todos los streams writable siguen el mismo patrón fundamental:

```
const myStream = getWritableStreamSomehow();
myStream.write('some data');
myStream.write('some more data');
myStream.end('done writing data');
```

## Clase: stream.Writable

### Event: 'close'

El evento 'close' es emitido cuando un stream y cualquiera de sus recursos (un descriptor de fichero, por ejemplo) ha sido cerrado. El evento indica que no hay más eventos para ser emitido, y no ocurrirán más actividades.

Un stream writable siempre emite el 'close' event si ha sido creado con el emitClose option.

### Event: 'drain'

Si una llamada a stream.write(chunk) retorna false, el evento 'drain' será emitido cuando sea apropiado resumir la escritura de datos al stream.

```
// Escribir datos al writable stream un millón de veces.

function writeOneMillionTimes(writer, data, econding, callback) {
  let i = 1000000;
  write();
  function write() {
    let ok = true;
    do {
      i--;
      if (i === 0) {
        // Last time
        writer.write(data, encoding, callback);
      } else {
        // Revisar si se puede continuar o esperar
        // No pasar el callback, ya que aún no hemos acabado
        ok = writer.write(data, encoding);
      }
    } while (i > 0 && ok);
    if (i > 0) {
      // Parar
      // Escribir algo más mientras ocurre el evento 'drain'
      writer.once('drain', write);
    }
  }
}
```

### Event: 'error'

El evento 'error' es emitido cuando ocurre un error mientras se está escribiendo datos o pasando por pipe. El callback listener toma un argumento de Error cuando es llamado.

El stream no se cierra cuando un evento de 'error' es emitido a menos que la opción de autoDestroy sea true cuando se crea el stream.

### Event: 'finish'

El evento 'finish' es emitido después de que el método stream.end() sea llamado y todos los datos emitidos.

```
const writer = getWritableStreamSomehow();
for (let i = 0; i < 100; i++) {
  writer.write(`hello, #${i}!\n`);
}
writer.on('finish', () => {
  console.log('All writes are now complete.');
});
writer.end('This is the end\n');
```

### Event: 'pipe'

El evento 'pipe' es emitido cuando el método stream.pipe() es llamado es un stream readable, agregando este writable a sus destinos.

```
const writer = getWritableStreamSomehow();
const reader = getReadableStreamSomehow();
writer.on('pipe', (src) => {
  console.log('Something is piping into the writer.');
  assert.equal(src, reader);
});
reader.pipe(writer);
```

### Event: 'unpipe'

El evento 'unpipe' es emitido cuando el método stream.unpipe() es llamado en un Readable stream, eliminando este Writable de sus destinos.

Esto también es emitido cuando un writable stream emite un error cuando un readable stream hace pipe en él.

```
const writer = getWritableStreamSomehow();
const reader = getWritableStreamSomehow();

writer.on('unpipe', (src) => {
  console.log('Something has stopped piping into the writer.');
  assert.equal(src, reader);
});
reader.pipe(writer);
reader.unpipe(writer);
```

### writable.cork()

El método ```writable.cork()``` fuerza que todos los datos escritos sean buffered en memoria. Estos datos serán eliminados una vez los métodos stream.uncork() o stream.end() sean llamados.

La intención principal de writable.cork() es acomodarse a la situación cuando muchos chunks son escritos por el stream en una rápida sucesión. En lugar de eviarlos todos al detino, writable.cork() lo almacena hasta que writable.uncork() sea llamado, que es cuando son pasados a writable._writev(), si existe. Esto evita una situación bloqueante donde los datos son almacenados mientras espera por el primer pequeño trozo a ser procesado. Sin embargo, usar writable.cork() sin implementar writable._writev() puede tener efectos adversos.

### writable.destroy([error])

- error <Error> Opcional, un error para emitir con el evento 'error'
- returns <this>

Destruye el stream. Opcionalmente emite un evento 'error' y emite un evento 'close' ( a menos que emitClose sea false ). Después de esta llamada, el stream writable ha terminado y posteriores llamadas a write() o end() resultan en un error ERR_STREAM_DESTROYED. Esto es una manera destructiva e inmediata de acabar con un stream. Llamadas previas a write() pueden no hacer drain, y pueden lanzar el error ERR_STREAM_DESTROYED. Usar end() en lugar de destroy si los datos han de ser enviados antes de cerrar, o esperar al evento 'drain' antes de destruir el stream. No se debe sobreescribir este método, usar en su lugar writable._destroy().

### writable.destroyed

- <boolean>

Es true después de que ```writable.destroy()``` haya sido llamado.

### writable.end([chunk[, encoding]][, callback])

- chunk <string> | <Buffer> | <Uint8Array> | <any> Opcional datos a escribir. Para streams no operando en objectMode, chunk será string, Buffer o Uint8Array. Para objectMode, chunk será un objeto de JavaScript diferente a null.
- encoding <string> Encoding si el chunk es un string
- callback <Function> Opcional callback para cuando el stream finaliza
- returns <this>

La llamada a stream.end() significa que no hay más datos que escribir al writable. Los argumentos opcionales chunk y encoding permiten una última escritura de datos inmediatamente antes de cerrar el stream. Si se provee, el opcional callback function se trata como un listener para el evento 'finish'.

Llamando a stream.write() después de llamar a stream.end() lanzará un error.

```
// Escribir 'hello' y entonces acabar con 'world!'
const fs = require('fs');
const file = fs.createWriteStream('example.txt');
file.write('hello, ');
file.end('world!');
// No se permite escribir más
```

### writable.setDefaultEncoding(encoding)

- encoding <string> El nuevo encoding por defecto
- returns <this>

El método writable.setDefaultEncoding() setea el encoding por defecto para un stream writable.

writable.uncork()

El método writable.uncork() elimina todos los datos almacenados desde que stream.cork() fue llamado.

Cuando se usa writable.cork() y writable.uncork() para manejar el buffering de escritura a un stream, se recomienda llamar a writable.uncork() precedido de process.nextTick. Hacer esto permite la agrupación por lotes de todas las llamadas a writable.write() que ocurre dentro de una fase del event loop de Node.js.

```
stream.cork();
stream.write('some ');
stream.write('data ');
process.nextTick(() => stream.uncork());
```

Si el método writable.cork() es llamado muchas veces en un stream, el mismo número de llamadas a writable.uncork() debe realizarse para eliminar los datos en buffer.

```
stream.cork();
stream.write('some ');
stream.cork();
stream.write('data ');
stream.nextTick(() => {
  stream.uncork();
  stream.uncork();
});
```

### writable.writable

- <boolean>

Es true si es seguro llamar a writable.write()

### writable.writableEnded

- <boolean>

Es true después de que writable.end() haya sido llamado. Esta propiedad no indica si los datos han han sido vaciados, para esto usar writable.writableFinished en su lugar.

### writable.writableCorked

- <integer>

Número de veces que se necesita llamar a writable.uncork() para dejar vacío el stream.

### writable.writableFinished

- <boolean>

Es true inmediatamente antes de que el evento 'finish' sea emitido.

### writable.writableHighWaterMark

- <number>

Retorna el valor de highWaterMark pasado en el costructor de este writable.
Es el tamaño del buffer interno del writable stream. Por defecto, 16384 (16 kb)

### writable.writableLength

- <number>

Esta propiedad contiene el número de bytes (u objetos) en la cola a ser escritos. El valor provee instrospección de datos dependiendo del estado del highWaterMark

### writable.writableObjectMode

- <boolean>

Getter para la propiedad objectMode de un stream writable

### writable.write(chunk[, encoding]][, callback])

- chunk <string> | <Buffer> | <Uint8Array> | <any> Opcional datos a escribir. Para streams que no operan en objectMode, chunk será un string, Buffer o Uint8Array. Para los que operan en objectMode, chunk puede ser cualquier objeto de JavaScript excepto null.
- encoding <string> El encoding. Si chunk es un string, por defecto es 'utf8'
- callback <Function> Callback para cuando el chunk de datos es vaciado.
- returns <boolean> false si el stream espera por el evento 'drain' a ser emitido antes de continuar escribiendo datos, de otra manera true.

El método writable.write() escribe datos al stream, y llama el callback una vez los datos han sido totalmente manejados. Si ocurren errores, el callback puede o no ser llamado con el error como primer argumento. Para controlar los errores de escritura, añadir un listener al evento 'error'.

El retorno es true si el buffer interno es menor que el highWaterMark configurado cuando el stream fue creado después de admitir el chunk. Si false es retornado, posteriores llamadas a write pararán hasta que el evento 'drain' sea emitido.