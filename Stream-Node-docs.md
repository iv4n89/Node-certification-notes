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

## Readable streams

Son abstracciones de un recurso cuyos datos serán consumidos

Ejemplos de Readable streams son:

- HTTP responses, en el cliente
- HTTP requests, en el server
- fs read streams
- zlib streams
- crypto streams
- TCP sockets
- child process stdout y stderr
- process.stdin

Todos los streams readable implementan la interfaz definida por la clase stream.Readable.

## Dos modos de lectura

Los readable streams tienen 2 modos de lectura: flowing y paused. Estos modos están separados del objectMode. Un readable stream puede estar en objectMode o no, sin importar si está en modo flowing o modo paused.

- En modo flowing, los datos son leídos desde el SO directamente y provistos a la aplicación tan rápido sea posible usando eventos desde la interfaz EventEmitter.
- En modo paused, el método read() ha de ser llamado esplícitamente para leer los chunks de datos desde el stream.

Todos los streams readables comienzan en paused mode pero pueden ser cambiados a flowing mode de una de estas maneras:

- Agregando un event handler de 'data'.
- Llamando al método stream.resume().
- Llamando al método stream.pipe() para enviar los datos a un stream writable.

El readable puede volver a paused mode de una de estas maneras:

- Si no hay pipe de destino, llamando al método stream.pause().
- Si hay pipe de destino, eliminando todos los pipes. Múltiples pipes de destino pueden ser eliminados llamando al método stream.unpipe().

Es importante recordar que un Readable no generará datos hasta que un mecanismo para consumir o ignorar estos datos sea provisto. Si el mecanismo que consume es deshabilitado o eliminado, el readable tratará de parar la generación de datos.

Por razones de compatibilidad, eliminar el event handler de 'data' no pausará el stream automáticamente. También, si hay pipes de destino, llamar a stream.pause() no garantizará que el stream se mantenga en pausa una vez estos destinos hagan drain o pidan más datos.

Si un readable es cambiado a flowing mode y no hay consumers disponibles para tratar los datos, éstos se perderán. Esto puede ocurrir, por ejemplo, cuando el readable.stream() es llamado sin un listener para el evento 'data', o cuando un event handler de 'data' es eliminado de un stream.

Agregando un event handler de 'readable' automáticamente hará que el stream deje de fluir, y que los datos deban ser consumidos vía readable.read(). Si el event handler de 'readable' es removido, entonces el stream comenzará a fluir nuevamente si hay un event handler para 'data'.

## Tres estados

Los dos modos son una manera de simplificar el manejo interno del estado en un readable.

Específicamente, en algún momento, cada readable está en uno de estos 3 posibles estados:

- readable.readableFlowing === null
- readable.readableFlowing === false
- readable.readableFlowing === true

Cuando readable.readableFlowing es null es que no hay mecanismos para consumir, por lo que el stream no va a generar datos. Mientras está en este estado, agregar un event handler de 'data' o llamar a readable.pipe() o a readable.resume() cambiará readable.readableFlowing a true, causando que el readable comience activamente a emitir eventos según los datos se van generando.

Llamar a readable.pause(), readable.unpipe() o recibir 'backpressure' hará que readable.readableFlowing pase a false, parando temporalmente los eventos y la generación de datos. En este estado, agregar un event handler de 'data' no cambiará a true el estado.

```
const { PassThrough, Writable } = require('stream');
const pass = new PassThrough();
const writable = new Writable();

pass.pipe(writable);
pass.unpipe(writable);
// readableFlowing es ahora false

pass.on('data', (chunk) => { console.log(chunk.toString()); });
pass.write('ok'); // No emitirá 'data'
pass.resume() // Debe ser llamado para que emita 'data'
```

Mientras que readable.readableFlowing sea false, los datos pueden estar acumulándose dentro del buffer interno del stream.

## Elegir un estilo de API

Usar el método readable.pipe() es recomendado por la mayoría de usuarios ya que ha sido implementado para proveer la manera más fácil de consumir datos del stream. Los desarrolladores un mayor control sobre la transferencia y generación de datos pueden usar el EventEmitter y readable.o('readable')/readable.read() o readable.pause()/readable.resume() APIs.

## Class: stream.Readable

### Event: 'close'

El evento ```close``` es emitido cuando un stream y cualquiera de sus recursos ha sido cerrado. El evento indica que no hay más eventos para ser emitidos, por lo que no seguirá trabajándose sobre este stream.

Un Readable stream va a emitir siempre ```'close'``` si ha sido creado con la opción de _emitClose_.

### Event: 'data'

- chunk <Buffer> |  <string> | <any> Un trozo de datos. Para streams que no están operando en objectMode, puede ser un string o un Buffer. Para streams en objectMode, el trozo puede ser cualquier objeto de JavaScript a excepción de null.

El evento ```data``` es emitido siempre que el stream envíe datos a un consumer. Esto puede ocurrir en cualquier momento si el stream está en flowing mode llamando a readable.pipe(), readable.resume() o agregando un listener al evento 'data'. El evento 'data' será emitido también siempre que el readable.read() es llamado y un trozo de datos está disponible para ser retornado.

Agreagar un event listener de 'data' a un stream que no ha sido explícitamente pausado hará que pase a flowing mode. Los datos serán pasados tan pronto sea posible.

El callback del listener recibirá el chunk como un string si se provee un encoding por defecto usando readable.setEncoding(). En caso contrario, el chunk será un buffer.

```
const readable = getReadableStreamSomehow();
readable.on('data', (chunk) => {
  console.log(`Received ${chunk.length} bytes of data.`);
});
```

### Event: 'end'

El evento ```end``` es emitido cuando no hay más datos a consumir desde el stream.

El evento 'end' no será emitido a menos que los datos sean completamente consumidos. Esto puede conseguirse pasando el stream a flowing mode, o llamando stream.read() repetidamente hasta que todos los datos sean consumidos.

```
const readable = getReadableStreamSomehow();
readable.on('data', (chunk) => {
  console.log(`Received ${chunk.length} bytes of data);
});
readable.on('end', () => {
  console.log('There will be no more data.');
});
```

### Event: 'error'

- <Error>

El evento ```error``` puede ser emitido por un Readable en caulquier momento. Típicamente, esto ocurre si el stream no está disponible para generar datos debido a un fallo interno, o cuando el stream trata de hacer push de un chunk de datos inválido.

El callback del listener recibe un objeto Error.

### Event: 'pause'

El evento ```pause``` es emitido cuando stream.pause() es llamado y readableFlowing no es false.

### Event: 'readable'

El evento ```readable``` es emitido cuando hay datos disponibles para ser leídos desde un stream. En algunos casos, agreagar un listener para 'readable' que algo de datos sea leído en un buffer interno.

```
const readable = getReadableSomehow();
readable.on('readable', function() {
  // Hay algo de datos para leer
  let data;

  while(data = this.read()) {
    console.log(data);
  }
});
```

El evento 'readable' también será emitido una vez que el final de los datos del stream llegue, pero antes el evento 'end' se emite.

Este evento indica que el stream tiene nueva información: si nuevos datos están disponibles o si se ha llegado al final. De esta manera, readable.read() retornará los datos disponibles o null si se ha llegado al final. Por ejemplo: 

```
const fs = require('fs');
const rr = fs.createReadStream('foo.txt');
rr.on('readable', () => {
  console.log(`readable: ${rr.read()}`);
});
rr.on('read', () => {
  console.log('end');
});
```

La salida de este script será:

```
$ node test.js
readable: null
end
```

En general, readable.pipe() y el evento 'data' son más fáciles de entender que el evento 'readable'. Sin embargo, el manejo de 'readable' mejora el rendimiento.

Si ambos 'readable' y 'data' son usados al mismo tiempo, 'readable' toma precedencia en el control del flujo. Ejemplo: 'data' es emitido sólo cuando readable.read() es llamado. La propiedad readableFlowing se vuelve false. Si hay un event listener de 'data' cuando el de 'readable' es eliminado, el stream comenzará a fluir (eventos 'data' serán lanzados sin llamar a .resume()).

### Event: 'resume'

El evento 'resume' es emitido cuando stream.resume() es llamado y readableFlowing no es true.

### readable.destroy([error])

- error <Error> Error que será pasado como payload en el evento 'error'
- returns <this>

Destruye el stream. Opcionalmente emite un evento 'error', y emite 'close' ( a menos que emitClose sea false ). Después de llamarlo, el readable stream soltará cualquier recurso y subsequentes llamadas a push() serán ignoradas. No se debe sobreescribir este método. En su lugar, implementar stream._destroy().

### readable.destroyed

- <boolean>

Es true después de que readable.destroy() haya sido llamado.

### readable.isPaused()

- returns <boolean>

El método readable.isPaused() retorna el estado operativo actual de Readable. Es usado principalmente por readable.pipe() internamente. No suele haber razón para usarlo directamente.

```
const readable = new stream.Readable();

readable.isPaused(); // false
readable.pause();
readable.isPaused(); // true
readable.resume();
readable.isPaused(); // false
```

### readable.pause()

- returns <this>

El método readable.pause() causará que el stream en flowing mode pare de emitir 'data', pasando a paused mode. Cualquier dato que tenga disponible quedará a la espera en el buffer interno.

```
const readable = getReadableSomehow();
readable.on('data', (chunk) => {
  console.log(`Received ${chunk.length} bytes of data.`);
  readable.pause();
  console.log('There will be no additional data for 1 second');
  setTimeOut(() => {
    console.log('Now data will start flowing again');
    readable.resume();
  }, 1000);
});
```

El método readable.pause() no tiene efecto si hay un event listener de 'readable'

### readable.pipe(destination([, options])

- destination <stream.Writable> El destino para escribir datos
- options <Object> Pipe options
  - end <boolean> Acaba el writer cuando acaba el reader. Por defecto: true.
- returns <stream.Writable> El destino, permitiendo encadenar si es un duplex o transform stream.

El método readable.pipe() conecta un stream writable, causando que pase automáticamente a flowing mode y pasando todos los datos al writable conectado. El flujo de datos será manejado automáticamente con lo que el writable destino no será sobrepasado con un readable que vaya más rápido.

```
const fs = require('fs');
const readable = getReadableStreamSomehow();
const writable = fs.createWriteStream('file.txt');
// Todos los datos del readable van a file.txt
readable.pipe(writable);
```

Se pueden conectar varios writable streams a un único readable stream

El readable.pipe() retorna una referencia al stream de destino, haciendo posible encadenar pipes de stream.

```
const fs = require('fs');
const r = fs.createReadStream('file.txt');
const z = zlib.createGzip();
const w = fs.createWriteStream('file.txt.gz');
r.pipe(z).pipe(w);
```

Por defecto, stream.end() es llamado en el writable destino cuando el recuros readable emite 'end', por lo que el destino no puede ser escrito más. Para deshabilitar este comportamiento la opción end puede ser pasada como false, causando que el stream destino quede abierto.

```
reader.pipe(writer, { end: false });
reader.on('end', () => {
  writer.end('Goodbye\n');
});
```

Una advertencia importante es que si el readable stream emite un error durante el proceso, el writable destino no se cierra automáticamente. Si un error ocurre será necesario un cierre manual de cada stream para prevenir memory leaks.

Los process.stderr y process.stdout (writables) nunca son cerrados a menos que Node.js pare, sin importar esta opción.

### readable.read([read])

- size <number> Argumento opcional que especifica la cantidad de datos a leer
- returns <string> | <Buffer> | <null> | <any>

El método readable.read() lanza algo de datos fuera del internal buffer y lo retorna. Si no hay datos disponibles para leer, se retorna null. Por defecto, los datos serán retornados como un buffer a menos que un encoding sea especificado usando readable.setEncoding() o el stream opere en objectMode.

El argumento opcional size especifica el número de bytes a leer. Si la cantidad de bytes a leer no está disponible, null será retornado a menos que el stream haya finalizado, en cuyo caso todos los datos restantes del internal buffer serán retornados.

Si size no es especificado, todos los datos contenidos en el buffer interno serán retornados.

El size debe ser menor a 1 gb.

El método readable.read() sólo se debe llamar en readable streams operando en paused mode. En flowing mode readable.read() es llamado automáticamente hasta que el internal buffer haya sido totalemente dreando.

```
cosnt readable = getReadableStreamSomehow();

// 'readable' puede ser lanzado múltiples veces teniendo datos
readable.on('readable', () => {
  let chunk;
  console.log('Stream is readable (new data received in buffer)');
  // Usar un loop para asegurarse de que se lee todos los datos disponibles
  while(null !== (chunk = readable.read())) {
    console.log(`Read ${chunk.length} bytes of data...`);
  }
});

// 'end' será lanzado una vez que no haya más datos disponibles
readable.on('read', () => {
  console.log('Reached end of stream.');
});
```

Cada llamada a readable.read() retorna un chunk de datos, o null. Los chunks no son concatenados. Un while loop es necesario para consumir todos loss datos seguidos en un buffer. Cuando tenemos un fichero largo y retorna null habiendo consumido todos los datos buffered, pero aún hay datos por venir que no han pasado por buffer, el evento 'readable' será lanzado nuevamente cuando haya datos en el buffer. Finalmente 'end' se lanzará cuando no haya más datos.

Para leer todo el contenido de readable, es necesario recoger chunks de varios eventos 'readable'

```
const chunks = [];

readable.on('readable', () => {
  let chunk;
  while (null !== (chunk = readable.read())) {
    chunks.push(chunk);
  }
});

readable.on('end', () => {
  const content = chunks.join('');
});
```

Un stream readable en objectMode siempre retornará un único item desde una llamada a readable.call(size), dependiendo del tamaño del argumento.

Si el readable.read() retorna un chunk de datos, un evento 'data' será emitido también.

Llamar a stream.read([size]) después de 'end' retorna null. No lanza error.

### readable.readable

- <boolean>

Es true si es seguro llamar a readable.read()

### readable.readableEncoding

- <null> | <string>

Getter para la propiedad encoding de un stream readable. La propiedad encoding puede ser seteada usando el readable.setEncoding().

### readable.readableEnd

- <boolean>

Es true cuando el evento 'end' es emitido.

### readable.readableFlowing

- <boolean>

Esta propiedad refleja el estado actual de un stream readable.

### readable.readableHighWaterMark

- <number>

Retorna el valor de highWaterMark pasado cuando se construye el readable

### readable.readableLength

- <number>

Propiedad que contiene el número de bytes u objetos en la cola listos para ser leídos. El valor provee instrospección de datos dependiendo del estado de hightWaterMark

### readable.readableObjectMode

- <boolean>

Getter para la propiedad objectMode de un stream readable.

### readable.resume()

- returns <this>

El método readable.resume() causa que una pausa explícita de un readable sea resumida emitiendo 'data', por lo que el stream pasa a flowing mode.

El método readable.resume() puede ser usado para consumir totalmente los datos desde un stream sin procesar los datos.

```
getReadableStreamSomehow()
  .resume()
  .on('end', () => {
    console.log('Reached the end, but did not read anything.');
  });
```

El método readable.resume() no tiene efecto si hay un listener para 'readable'

### readable.setEncoding(encoding)

- encoding <string> El enconding a usar
- returns <this>

El método readable.setEncoding() setea el encoding de los datos desde un readable stream.

Por defecto, no hay un encoding asignado y los datos del stream serán retornados como un buffer. Setear un encoding causa que los datos del stream sean retornados como strings del encoding especificado en lugar de buffers.

```
const readable = getReadableSomehow();
readable.setEncoding('utf8');
readable.on('data', () => {
  assert.equal(typeof chunk, 'string');
  console.log('Got $d characters of string data: ', chunk.lengh);
});
```

### readable.unpipe([destination])

- destination <stream.Writable> Opcional stream específico para el unpipe.
- returns <this>

El método readable.unpipe() saca un stream writable previamente unido al pipe.

Si el destino no es especificado, todos los pipes serán eliminados.

Si el destino es especificado, pero no hay pipe, no hará nada.

```
const fs = require('fs');
const readable = getReadableStreamSomehow();
const writable = fs.createWritableStream('file.txt');
// Todos los datos del readable van al file.txt
// pero sólo por el primer segundo.
readable.pipe(writable);
setTimeout(() => {
  console.log('Stop writing to file.txt');
  readable.unpipe(writable);
  console.log('Manually close the file stream');
  writable.end();
}, 1000);
```

### readable.unshift(chunk[, encoding])

- chunk <Buffer> | <Uint8Array> | <string> | <null> | <any> Chunk de datos para sacar de la cola de lectura. Para streams no operando en objectMode, un chunk puede ser Buffer, string o Uint8Array, o null. Para objectMode, chunk puede ser cualquier objeto de JavaScript.
- encoding <string> Encoding de los string chunks, debe ser un buffer encoding válido, como 'utf8' o 'ascii'.

Si un chunk es null es señal de final del stream (EOF).

readable.unshift() mete un chunk de datos al buffer interno. Esto es útil en ciertas situaciones donde un stream es consumido por código que necesita desconsumir algo de datos que es sacado del recurso, por lo que los datos pueden ser pasados a otros.

Stream.unshift(chunk) no puede ser llamado después de 'end', un error sería emitido.

```
// Pull off a header delimiter by \n\n.
// Use unshift() if we get too much.
// Call the callback with (error, header, stream).
const { StringDecoder } = require('string_decoder');
function parseHeader(stream, callback) {
  stream.on('error', callback);
  stream.on('readable', onReadable);
  const decoder = new StringDecoder('utf8');
  let header = '';
  function onReadable() {
    let chunk;
    while (null !== (chunk = stream.read())) {
      const str = decoder.write(chunk);
      if (str.match(/\n\n/)) {
        const split = str.split(/\n\n/);
        header += split.shift();
        const buf = Buffer.from(remaining, 'utf8');
        stream.removeListener('error', callback);
        // Eliminar el listener de 'readable' antes de hacer un unshift
        stream.removeListener('readable', onReadable);
        if (buf.length)
          stream.unshift(buf);
        // Ahora el cuerpo del mensaje puede ser leído desde el stream
        callback(null, header, stream);
      } else {
        // Aún leyendo el header
        header += str;
      }
    }
  }
}
```

stream.unshift(chunk) no va a terminar el proceso de lectura reseteando el estado del proceso de lectura. Esto causa resultados inesperados si readable.unshift() es llamado durante un read (ejemplo, dentro de una implementación de stream._read() en un custom stream). Siguiendo la llamada a readable.unshift() con un inmediato stream.push('') reseteará el estado apropiadamente, sin embargo es mejor evitar llamar a readable.unshift() mientras se realiza un read.

### readable.wrap(stream)

- stream <Stream> Un stream readable a la antigua
- returns <this>

Antes de Node.js 0.10 streams no implementaban el api completo del módulo de stream como ahora.

Cuando una librería antigua de Node.js emite 'data' y tiene un método stream.pause() que es advertencia solo, el readable.wrap() puede usarse para crear un readable stream que usa el viejo stream como recurso.

Raramente será necesario.

```
const { OldReader } = require('./old-api-module.js');
const { Readable } = require('stream');
const oreader = new OldReader();
const myReader = new Readable().wrap(oreader);

myReader.on('readable', () => {
  myReader.read();
});
```

### readable[Symbol.asyncIterator]()

- return <AsyncIterator> para consumir el stream completo

```
const fs = require('fs');

async function print(readable) {
  readable.setEncoding('urf8');
  let data = '';
  for await (const chunk of readable) {
    data += chunk;
  }
  console.log(data);
}

print(fs.createReadStream('file')).catch(console.error);
```

Si el loop finaliza con un break o throw, el stream será destruido. En otros términos, iterar sobre el stream consumirá el stream completo. El stream será leído en trozos iguales al hightWaterMark.

## Duplex y Transform streams

### Class: stream.Duplex

Duplex streams son los que implementan tanto la interfaz Readable como Writable

Ejemplos:

- TCP sockets
- zlib streams
- crypto streams

### Class: stream.Transform

Transform streams son duplex streams cuya salida está relacionada de alguna manera con la entrada. Como los duplex streams, los transform streams implementan ambas interfaces.

Ejemplos:

- zlib streams
- crypto streams

### transform.destroy([error])

- error <Error>
- return <this>

Destruye el stream, opcionalmente emite un evento de 'error'. Después de esta llamada, el stream transform suelta cualquier recurso interno. No se debe sobreescribir este método, en su lugar implementar readable._destroy(). La implementación por defecto de _destroy() para Transform también emite 'close' a menos que emitClose sea false.

## stream.finished(stream[, options], callback)

- stream <Stream> Un readable y/o writable stream
- options <Object>
  - error <boolean> Si es false, llamar a emit('error', err) no es tratado como finalizado. Default: true
  - readable <boolean> Cuando es false, el callback será llamado cuando el stream acaba incluso cuando el stream aún pueda ser leíble. Default: true.
  - writable <boolean> Cuando es false, el callback será llamado cuando el stream acaba incluso cuando el stream aún pueda ser escribible. Default: true.
- callback <Function> Una función de callback que toma un argumento opcional de error.
- returns <Function> Una función de limpieza que remueve todos los listeners registrados.

Una función para ser notificado cuando un stream ya no es legible, escribible o tiene un error un cierre prematuro.

```
const { finished } = require('stream');

const rs = fs.createReadStream('archive.tar');

finished(rs, (err) => {
  if (err) {
    console.error('Stream failed', err);
  } else {
    console.log('Stream is done already');
  }
});

re.resume(); // Drain the stream
```

Es especialmente útil para el manejo de errores en escenarios donde el stream es destruído prematuramente (como un http request abortado), y no emite 'end' o 'finish'.

El api de finished se puede pasar a promesa:

```
const finished = util.promisify(stream.finished);

const rs = fs.createReadStream('archive.tar');

async function run() {
  await finished(rs);
  console.log('Stream is done already);
}

run().catch(console.error);
rs.resume(); // Drain the stream
```

stream.finished() deja varios event listeners ('error', 'end', 'finish' y 'close') después del callback. La razón de esto es que ante un evento de 'error' no cause un crash inesperado. Se puede realizar limpieza con la función retornada.

```
const cleanup = finished(rs, (error) => {
  cleanup();
  //...
});
```

## stream.pipeline(...streams, callback)

- ...streams <Stream> Dos o más streams para hacer pipe
- callback <Function> Llamada cuando el pipeline es finalizado
  - err <Error>

Un método modular para hacer pipe entre streams gestionando errores y limpiando y proveyendo un callback cuando el pipeline se completa.

```
const { pipeline } = require('stream');
const fs = require('fs');
const zlib = require('zlib');

// Usar el api de pipeline para hacer pipe con una serie de streams
// y ser notificado cuando el pipeline finaliza

// Un pipelin para hacer gzip a un fichero potencialmente grande eficientemente:

pipeline(
  fs.createReadStream('archive.tar'),
  zlib.createGzip(),
  fs.createWriteStream('archive.tar.gz'),
  err => {
    if (err) {
      console.error('Pipeline failed', err);
    } else {
      console.log('Pipeline succeeded.');
  	}
  }
);
```

El api de pipeline se puede volver promesa

```
const pipeline = util.promisify(stream.pipeline);

async function run() {
  await pipeline(
    fs.createReadStream('archive.tar'),
    zlib.createGzip(),
    fs.createWriteStream('archive.tar.gz')
  );
  console.log('Pipeline succeeded');
}

run().catch(console.error);
```

stream.pipeline() llamará a strea.destroy(err) en cualquier excepción del stream:
- Readable que ha emitido 'end' o 'close'
- Writable que ha emitido 'finish' o 'close'

stream.pipeline() deja algunos event listeners después del callback. En el caso de reusar los streams tras un error, esto puede causar errores en los event listeners.

## stream.Readable.from(iterable, [options])

- iterable <Iterable> Objeto que implementa el protocolo iterable Symbol.asyncIterable o Symbol.iterator. Emite un evento 'error' si un valor null es pasado.
- options <Object> Opciones provistas para un nuevo stream.Readable([options]). Por defecto, Readable.from() setea options.objectMode a true, a menos que explícitamente se ponga a false.
- returns <streams.Readable>

Un método de utilidad para crear streams readable desde iterables

```
const { Readable } = require('stream');

async function * generate() {
  yield 'hello';
  yield 'streams';
}

const readable = Readable.from(generate());

readable.on('data', chunk => {
  console.log(chunk);
});
```

Llamar a Readable.from(string) o Readable.from(buffer) no hará que strings o buffer sean iterados para hacer match con la semántica de otros streams por razones de rendimiento.

## API para stream implementers

El módulo de stream API ha sido diseñado para ahcer posible su implementación fácil usando la herencia de prototipo de JavaScript.

Primero, un stream developer debe declarar una nueva clase de JavaScript que extienda uno de los streams básicos (writable, readable, duplex o transform), asegurándose de llamar al constructor padre apropiado.

```
const { Writable } = require('stream');

class MyWritable extends Writable {
  constructor({ highWaterMark, ...options }) {
    super({
      highWaterMark,
      autoDestroy: true,
      emitClose: true
    });
    // ...
  }
}
```

Cuando extendemos streams, tener presente que las opciones que el usuario se pueden y deben proveer antes de llegar al constructor base. Por ejemplo, si la implementación asume la responsabilidad del autoDestroy y emitClose, no permitir a los usuarios sobreescribirlos.

Las nuevas clases de streams entonces implementan uno o más métodos específicos, dependiendo del tipo de stream a ser creado, como se detalla:

<table>
  <thead>
    <tr>
      <th><b>Use-Case</b></th>
      <th><b>Class</b></th>
      <th><b>Method(s) to implement</b></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Sólo lectura</td>
      <td>Readable</td>
      <td>_read()</td>
    </tr>
    <tr>
      <td>Sólo escritura</td>
      <td>Writable</td>
      <td>_write(), _writev(), _final()</td>
    </tr>
    <tr>
      <td>Lectura y escritura</td>
      <td>Duplex</td>
      <td>_read(), _write(), _writev(), _final()</td>
    </tr>
    <tr>
      <td>Operar y escribir datos, entonces leer el resultado</td>
      <td>Transform</td>
      <td>_transform(), _flush(), _final()</td>
    </tr>
  </tbody>
</table>

La implementación de un stream no debe nunca llamar a métodos públicos que son para el uso de consumidores. Hacer esto puede llevar a efectos secundarios mientras se consume un stream.

Evitar sobreescrir métodos públicos como write(), end(), uncork(), read() y destroy(), o emitir internamente eventos como 'error', 'data', 'end', 'finish' y 'close' desde .emit(). Hacer esto puede romper el actual o futuros streams dependiendo del comportamiento y/o compatibilidad con otros streams, streams utils y expectativas de usuarios

## Construcción simplificada

Para casos simples es posible construir un stream sin dependencias o herencia. Esto puede ser realizado directamente creando instancias de stream.Writable, stream.Readable, stream.Duplex o stream.Transform y pasando el método apropiado como opciones del constructor.

```
const { Writable } = require('stream');

const myWritable = new Writable({
  write(chunk, encoding, callback) {
    // ...
  }
});
```

## Implementar un writable stream

La clase stream.writable debe ser extendida para implementar un writable stream.

Custom writable streams deben llamar al constructor stream.Writable([options]) e implementar el writable._write() y/o writable._writev().

### new stream.Writable([options])

- options <Object>
  - highWaterMark <number> Nivel de buffer cuando stream.write() retorna false. Default 165384 (16kb) o 16 para objectMode.
  - decodeStrings <boolean> Codificar o no los strings pasados por stream.write() a buffers (con el encoding facilitado) antes de pasarlo a stream._write(). Otros tipos de datos no son convertidos (ejemplo, los buffers no son decodificados en strings). Setearlo a false prevendrá que los strings sean convertidos. Default true.
  - defaultEncoding <string> El encoding por defecto usado cuando no se pasa un encoding como argumento a stream.write(). Default 'utf8'
  - objectMode <boolean> Indica si stream.write(cualquierObjeto) será válido. Cuando se setea, es posible escribir objetos si son soportados por la implementación. Default false.
  - emitClose <boolean> Si el stream debe emitir 'close' después de haber sido destruido. Default true.
  - write <Function> Implementación del método stream._write().
  - writev <Function> Implementación del método stream._writev().
  - destroy <Function> Implementación del método stream._destroy().
  - final <Function> Implementación del método steram._final().
  - autoDestroy <boolean> Si el stream debe llamar automáticamente a .destroy() tras el final. Default false.

  ```
  const { Writable } = require('stream');

  class myWritable extends Writable {
    constructor (options) {
      // Llama al constructor de stream.Writable().
      super(options);
      // ...
    }
  }
  ```

  O, usando pre-ES6 constructors:

  ```
  const { Writable } = require('stream');
  const uril = require('util');

  function MyWritable(options) {
    if (!(this instanceof MyWritable)) {
      return new MyWritable(options);
    }
    Writable.call(this, options);
  }

  util.inherits(MyWritable, Writable);
  ```

  O usando el constructor simplificado:

  ```
  const { Writable } = require('stream');

  const myWritable = new Writable({
    write(chunk, encoding, callback) {
      // ...
    },
    writev(chunks, callback) {
      // ...
    }
  });
  ```

  ### writable._write(chunk, encoding, callback)

  - chunk <Buffer> | <string> | <any> El Buffer a ser escrito, convertido a desde string pasado a stream.write(). Si la opción decodeStrings es false o el stream opera en objectMode, el chunk no será convertido y será pasado a stream.write().
  - encoding <string> Si el chunk es un string, entonces encoding es el character encoding de este string. Si el chunk es un Buffer o está operando en objectMode, encoding será ignorado.
  - callback <Function> Llamar a esta función (opcionalmente con un argumento de error) cuando se complete el procesado de un chunk.

Todas las implementaciones de writable debenn proveer un writable._write() y/o writable._writev() para enviar datos al recurso.

Transform streams proveen su propia implementación del writable._write().

Esta función NO DEBE ser llamada por aplicaciones de código directamente. Se debe implementar por clases hijas y ser llamado internamente.

El método callback debe ser llamado como señal de que una escritura se completó con éxito o falló. El primer argumento pasado al callback debe ser el objeto de Error si la llamada falla o null si la escritura es correcta.

Todas las llamadas a writable.write() que ocurren entre que writable._write() es llamado y el callback es llamado causará que los datos escritos sean almacenados. Cuando el callback es invocado, el stream debe emitir un evento 'drain'. Si la implementación del stream es capaz de procesar múltiples chunks de datos de una vez, el writable._writev() debe ser implementado.

Si decodeStrings se setea a false en el constructor, el chunk permanecerá como el mismo objeto que es pasado a .write(), y debe ser un string. Esto es para soportar implementaciones que tienen un manejo optimizado de ciertos string data encoding. En este caso, el argumento encoding indicará el character encoding del string. De otra manera, el argumento encoding puede ser ignorado.

El método writable._write() es precedido con una barra baja ya que interno a la clase que lo define, y nunca debe ser llamado por programas de usuarios.

### writable._writev(chunks, callback)

- chunks <Object[]> Los chunks a ser escritos. Cada chunk tiene el formato: { chunk: ..., encoding: ... }.
- callback <Function> Será invocado al finalizar el procesamiento de los chunks

Esta función NO DEBE ser llamada por la aplicación directamente. Debe ser implementada por clases hijas y llamada internamente.

El método writable._writev() debe ser implementado además de o alternativamente a writable._write() en implementaciones capaces de procesar varios chunks a la vez. 

El método writable._writev() es precedido con una barra baja ya que es un método interno de la clase y no debe ser llamado directamente por el usuario.

### writable._destroy(err, callback)

- err <Error> Un posible error
- callback <Function> Una función de callback que toma un argumento opcional de error

El método _destroy() es llamado por writable.destroy(). Puede ser sobreescrito por una clase hija y no debe ser llamada directamente

### writable._final(callback)

- callback <Function> Llamar a esta función (opcionalmente con un argumento de error) cuando se finaliza la escritura de los datos restantes.

El método _final() no debe ser llamado directamente. Debe ser implementado por la clases hijas.

Esta función opcional será llamada antes del cierre del stream, dejando el evento 'finish' hasta después de la llamada al callback. Es útil para cerrar recursos o escribir datos en buffer antes del cierre del stream

## Errores durante la escritura

Pueden ocurrir errores durante el procesado de writabe._write(), writable._writev() y writable._final() que pueden propagarse invocando el callback y pasando el error como primer argumento. Lanzando un Error desde dentro de estos métodos o manualmente emitir 'error' provoca un comportamiento indefinido.

Si un readable conecta con un writable cuando writable emite error, el readable se desconectará.

```
const { Writable } = require('stream');

const myWritable = new Writable({
  write(chunk, encoding, callback) {
    if (chunk.toString().indexOf('a') => 0) {
      callback(new Error('chunk is invalid'));
    } else {
      callback();
    }
  }
});
```

## Un ejemplo de writable stream

```
const { Writable } = require('stream');

class MyWritable extends Writable {
  _write(chunk, encoding, callback) {
    if (chunk.toString().indexOf('a') >= 0) {
      callback(new Error('chunk is invalid');
    } else {
      callback();
    }
  }
}
```

## Decodificando buffers en un writable stream

Decodificar buffers es una tarea común, por ejemplo, cuando usamos transformers cuyo input es un string. 

```
const { Writable } = require('stream');
const { StringDecoder } = require('string_decoder');

class StringWritable extends Writable {
  cosntructor(options) {
    super(options);
    this._decoder = new StringDecoder(options && options.defaultEncoding);
    this.data = '';
  }
  _write(chunk, encoding, callback) {
    if (encoding === 'buffer') {
      chunk = this._decoder.write(chunk);
    }
    this.data += chunk;
    callback();
  }
  _final(callback) {
    this.data += this._decoder.end();
    callback();
  }
}

const euro = [[0xE2, 0x82], [0xAC]].map(Buffer.from);
const w = new StringWritable();

w.write('currency: ');
w.write(euro[0]);
w.end(euro[1]);

console.log(w.data); // concurrency: €
```

## Implementar un readable stream

- options <Object>
  - highWaterMark <number> El máximo de bytes a guardar en el internal buffer antes de dejar de leer del recurso. Default: 16384 (16kb) o 16 en objectMode.
  - encoding <string> Si se especifica, los buffers serán decodificados a strings usando el encoding especificado. Default null.
  - objectMode <boolean> Está o no en objectMode. stream.read(n) retornará un único valor en lugar de Buffer de tamaño n. Default false.
  - emitClose <boolean> Debe o no lanzar el 'close' después de ser destruido. Default true.
  - read <Function> Implementación de stream._read()
  - destroy <Function> Implementación de stream._destroy()
  - autoDestroy <boolean> El stream debe llamar automáticamente a .destroy() después de finalizar. Por defecto false.

```
const { Readable } = require('stream');

class MyReadable extends Readable {
  constructor (options) {
    // Llama al constructor de Readable(options)
    super(options)
    // ...
  }
}
```

O usando pre-ES6 constructors

```
const { Readable } = require('stream');
const util = require('util');

function MyReadable(options) {
  if (!(this instanceof MyReadable)) {
    return new MyReadable(options);
  }
  Readable.call(this, options);
}

util.inherits(MyReadable, Readable);
```

O usando el constructor simplificado

```
const { Readable } = require('stream');

const myReadable = new Readable({
  read(size) {
    // ...
  }
});
```

### readable._read(size)

- size <number> Número de bytes a leer asíncronamente

Esta función NO DEBE ser llamada directamente.

Todas las implementaciones de stream deben tener una implementación de readable._read() para obtener los datos desde el recurso.

Cuando readable._read() es llamado, si los datos están disponibles en el recurso, la implementación debe comenzar a enviar datos a la cola de lectura usando this.push(dataChunk). _read() puede continuar leyendo desde el recurso y enviando datos hasta que readable.push() retorne false. Sólo cuando _read() es llamado de nuevo después de finalizar debe resumir el envío de datos adicionales en la cola.

Una vez que readable._read() ha sido llamado, no será llamado de nuevo hasta que hayan más datos pasados por readable.push(). Datos vacíos como buffers vacíos o strings vacíos no llamarán a readable._read().

El argumento size es advirsorio.l Para implementaciones de read() donde una operación retorna datos puede usarse el argumento size para determinar cuántos datos se pedirán. Otras implementaciones pueden igrnorar este argumento y simplemente proveer datos cuando estén disponibles. No hay necesidad de esperar hasta que los bytes de size estén disponibles a stream.push(chunk).

El método readable._read() es precedido con un _ ya que es interno.

### readable._destroy(err, callback)

- err <Error> Un posible error
- callback <Function> Una función callback que toma un argumento opcional de error

Es llamado por readable.destroy(). Puede ser sobreescrito en la clase hija, no debe ser llamado directamente.

### readable.push(chunk[, encoding])

- chunk <Buffer> | <Uint8Array> | <string> | <null> | <any> Trozo de datos a enviar a la cola de lectura. Para streams que no operan en objectMode, chunk debe ser Buffer, string o Uint8Array. Si está en objectMode, será cualquier valor de JavaScript.
- encoding <string> Encoding de los chunks string. Debe ser un encoding válido de Buffer, como 'utf8' o 'ascii'.
- returns <boolean> true si un chunk adicional será empujado, false si no.

Cuando un chunk es un Buffer, Uint8Array o string, el chunk de datos será agregado a la cola interna de usuarios del stream para ser consumido. Pasando chunk como null será señal de final del stream (EOF), tras lo que no se podrá escribir más datos.

Cuando un readable opera en paused mode, los datos agregados con readable.push() puede ser leido llamando a readable.read() cuando el evento 'readable' es emitido.

Cuando Readable opera en flowing mode, los datos agregados con readable.push() serán entregados emitiendo el evento 'data'.

readable.push() es diseñado para ser tan flexible como pueda. Por ejemplo, cuando se encapsula un recurso de bajo nivel que provee algún mecanismo de pause/resume, y un data callback, el low-level resource puede ser encapsulado en una instancia custom de readable.

```
// '_source' es un objeto con readStop() y readStart()
// y un 'ondata' miembro que es llamado cuando tiene datos
// y 'onend' miembro que es llamado cuando los datos se acaban

class SourceWrapper extends Readable {
  constructor(options) {
    super(options);

    this._source = getLowLevelSourceObject();
    // Cada vez que hay datos, lo envia al internal buffer
    this._source.ondata = (chunk) => {
      // Si push() retorna false, para de leer el recurso
      if (!this.push(chunk)) {
        this._source.readStop();
      };

    this._source.onend = () => {
      this.push(null);
    };
  }

  // _read() será llamado cuando el stream quiere enviar más datos
  // El argumento size será ignorado en este caso
  _read(size) {
    this._source.readStart();
  }
}
```

readable.push() es usado para empujar contenido al internal buffer. Puede ser realizado por readable._read().

Para streams no operando en objectMode, si el parámetro chunk de readable.push() es undefined será tratado como un string vacío o buffer.

## Errores en la lectura

Errores pueden ocurrir durante el proceso de readable._read() que pueden ser propagados por readable.destroy(err). Enviando un Error desde dentro de readable._read() o manualmente emitir 'error' resulta en un comportamiento indefinido.

```
const { Readable } = require('stream');

const myReadable = new Readable({
  read(size) {
    const err = checkSomeErrorCondition();
    if (err) {
      this.destroy(err);
    } else {
      // Hacer algo
    }
  }
});
```

## Un ejemplo de stream contandor

```
const { Readable } = require('stream');

class Counter extends Readable {
  constructor(opt) {
    super(opt);
    this._max = 1000000;
    this._index = 1;
  }

  _read() {
    const i = this._index++;
    if (i > this._max) {
      this.push(null);
    } else {
      const str = String(i);
      const buf = Buffer.from(str, 'ascii');
      this.push(buf);
    }
  }
}
```

## Implementar un stream Duplex

Un stream duplex implementa tanto Readable como Writable, como en TCP sockets.

Ya que JavaScript no soporta la herencia múltiple, stream.Duplex ha de ser extendida para abarcar ambos.

El prototipo de duplex hereda de Readable y parasita Writable, pero instanceof funciona bien con ambas clases debido a que sobreescribe Symbol.hasInstance en Writable.

Custom duplex puede llamar a new stream.Duplex([options]) constructor e implementar ambos readable._read() y writable._write().

### new stream.Duplex(options)

- options Pasado tanto para writable como para readable
  - allowHalfOpen <boolean> Si es false el stream automáticamente cerrará el writable cuando el readable termine. Default true.
  - readable <boolean> Setea si el stream debe ser readable. Default true
  - writable <boolean> Setea si el stream debe ser writable. Default true
  - readableObjectMode <boolean> Setea objectMode para readable. No tiene efecto si objectMode es true. Default false.
  - writableObjectMode <boolean> Setea objectMode para writable. No tiene efecto si objectMode es true. Default false.
  - readableHighWaterMark <number> Setea highWaterMark para readable. No tiene efecto si highWaterMark es provisto.
  - writableHighWaterMark <number> Setea highWaterMark para writable. No tiene efecto si highWaterMark es provisto.

```
const { Duplex } = require('stream');

class MuDuplex extends Duplex {
  constructor(options) {
    super(options);
    // ...
  }
}
```

O pre-ES6 constructors

```
const { Duplex } = require('stream');
cosnt util = require('util');

function MyDuplex(options) {
  if (!(this instanceof MyDuplex)) {
    return new MyDuplex(options);
  }
  Duplex.call(this, options);
}
util.inherits(MyDuplex, Duplex);
```

O constructor simplificado

```
const { Duplex } = require('stream');

const myDuplex = new Duplex({
  read(size) {
    // ...
  }
  write(chunk, encoding, callback) {
    // ...
  }
});
```

## Un ejemplo de stream duplex

```
const { Duplex } = require('stream');
const kSource = Symbol('source');

class MyDuplex extends Duplex {
  constructor(source, options) {
    super(options);
    this[kSource] = source;
  }

  _write(chunk, encoding, callback) {
    if (Buffer.isBuffer(chunk)) {
      chunk = chunk.toString();
    }
    this[kSource].writeSomeData(chunk);
    callback();
  }

  _read(size) {
    this[kSource].fetchSomeData(size, (data, encoding) => {
      this.push(Buffer.from(data, encoding));
    });
  }
}
```

Lo más importante en un duplex es que los lados readable y writable operen independientemente aunque coexistan en un único objeto.

## Object mode en stream duplex

En duplex streams el objectMode puede ser exclusivo de readable o writable usando readableObjectMode o writableObjectMode.

```
const { Transform } = require('stream');

// Todos los streams transform son también duplex
const myTransform = new Transform({
  writableObjectMode: true,
  transform(chunk, encoding, callback) {
    // Coaccionar el trozo si es necesario
    chunk |= 0;

    // Transformar el chunk en algo más
    const data = chunk.toString(16);

    // Empujar los datos a la cola de lectura
    callback(null, '0'.repeat(data.length % 2) + data);
  }
});

myTransform.setEncoding('ascii');
myTransform.on('data', (chunk) => console.log(chunk));

myTransform.write(1); // 01
myTransform.write(10); //0a
myTransform.write(100); //64
```

## Implementar un stream transform

Un stream transform es un duplex donde el output es una transformación del input. Ejemplos son zlib o crypto streams.

No hay requisitos de que el output deba ser del mismo tamaño que el input, el mismo número de chunks o llegar al mismo tiempo. Por ejemplo, Hash siempre va a tener un chunk de salida que sale cuando el input acaba.

La clase stream.Transform extiende para implementar Transform stream.

Hereda el prototype de duplex e implementa su versión de _write() y _read(). Custom transform implementations deben implementar _transform() y pueden implementar _flush().

Si la salida de Readable no es consumida puede causar que writable se pause.

### new stream.Transform([options])

- options <Object> Pasado tanto para writable como para readable.
  - transform <Function> Implementación de stream._transform()
  - flush <Function> Implementación de stream._flush()

```
const { Transform } = require('stream');

class MyTransform extends Transform {
  constructor (options) {
    super(options);
    // ...
  }
}
```

O pre-ES6

```
const { Transform } = require('stream');
const util = require('util');

function MyTransform(options) {
  if (!(this instanceof MyTransform)) {
    return new MyTransform(options);
  }
  Transform.call(this, options);
}
util.inherits(MyTransform, Transform);
```

O constructor simplificado

```
const { Transform } = require('stream');

const MyTransform = new Transform({
  transform(chunk, encoding, callback) {
    // ...
  }
});
```

### Events 'finish' y 'end'

Los eventos 'finish' y 'end' son de stream.Writable y stream.Readable respectivamente. El evento 'finish' es emitido después de que stream.end() sea llamado y todos los chunks han sido procesados por _transform(). El evento 'end' es emitido después de que todos los datos hayan sido emitidos, que ocurre cuando el callback en _flush() ha sido llamado.

### transform._flush(callback)

- callback <Function> Una función callback a ser llamada cuando los datos restantes se han vacíado.

Esta función NO DEBE ser llamada directamente. La llama internamente métodos de readable.

En algunos casos, una operación de transform puede necesitar emitir un bit de datos adicional al final del stream. Por ejemplo, una compresión de zlib almacenará una cantidad estado interno para comprimir el output. Cuando el stream acaba, estos datos adicionales deben ser vaciados para completar la compresión.

Custom implementaciones de transform pueden implementar _flush(). Será llamado cuando no hay más datos a consumir, pero antes del evento 'end'.

Dentro de _flush(), transform.push() puede ser llamado 0 o más veces. El callback debe ser llamado cuando la operación de vaciado se ha completado.

### transform._transform(chunk, encoding, callback)

- chunk <Buffer> | <string> | <any> El Buffer a ser transformado, convertido desde el string pasado a stream.write(). Si la opción decodeStrings es false o el stream opera en objectMode, el chunk no será convertido y será pasado igualmente a stream.write().
- encoding <string> Si el chunk es un string, entonces esto es su encoding. Si el chunk es un buffer, esto debe ser el valor especial 'buffer'. Se ignora en este caso.
- callback <Function> Una función de callback llamada después del procesado del chunk.

Esta función NO DEBE ser llamada directamente. Será llamada internamente por métodos de Readable.

Se debe proveer implementación de _transform() para aceptar input y producir output. Maneja los bytes siendo escritos, computa un output y pasa ese output hacia la parte readable usando transform.push().

transform.push() puede ser llamado 0 o más veces para generar outputs de un único chunk, dependiendo de cuánto ha de ser el output resultante del chunk.

Es posible que no se genere output de chunks en el input.

El callback debe ser llamado sólo cuando el chunk actual sea completamente consumido. El primer argumento del callback debe ser un Error si un error ocurre, o null si fue bien. Si un segundo argumento es pasado, será enviado a transform.push(). En otraas palabras, las dos líneas siguientes son equivalentes:

```
transform.prototype._transform = function(data, encoding, callback) {
  this.push(data);
  callback();
}

transform.prototype._transform = function(data, encoding, callback) {
  callback(null, data);
}
```

_transform() se llama en paralelo. Streams impementan mecanismos de cola y para recibir el siguiente chunk deben ser llamados, síncrona o asíncronamente.

## Class: stream.PassThrough

stream.PassThrough es una implementación trivial de Transform que simplemente pasa el input hacia el output. Su principal uso es en ejemplos o testing, pero hay algunos casos en los que PassThrough puede ser útil, como construir nuevos tipos de streams.

## Notas adicionales

### Compatibilidad de streams con generadores asíncronos e iteradores asíncronos

Dentro del soporte de generadores e iteradores en JavaScript, generadores asíncronos son un first-class language-level stream construct.

### Consumir readable streams con async iterators

```
(async function() {
  for await (const chunk of readable) {
    console.log(chunk);
  }
})();
```

### Crear readable streams con generadores asíncronos

```
const { Readable } = require('stream');

async function * generate() {
  yield 'a';
  yield 'b';
  yield 'b';
  yield 'c';
}

const readable = Readable.from(generate());

readable.on('data', (chunk) => {
  console.log(chunk);
});
```

### Canalizar un stream writable desde iteradores asíncronos

```
const { once } = require('events');
const finished = util.promisify(stream.finished);

const writable = fs.createWriteStream('./file');

function drain(writable) {
  if (writable.destroyed) {
    return Promise.rejected(new Error('premature close'));
  }

  return Promise.race([
    once(writable, 'drain'),
    once(wriatable, 'close')
      .then(() => Promise.reject(new Error('premature close')))
  ]);
}

async function pump(iterable, writable) {
  for await (const chunk of iterable) {
    if (!writable.write(chunk)) {
      await drain(writable);
    }
  }
  writable.end();
}

(async function() {
  await Promise.all([
    pump(iterable, writable),
    finished(writable)
  ]);
})();
```

Alternativamente un readable stream puede ser encapsulado con Readable.from() y piped via pipe().

```
const finished = util.promisify(stream.finished);

const writable = fs.createWriteStream('./file');

(async function() {
  const readable = Readable.from(iterable);
  readable.pipe(writable);
  await finished(writable);
})();
```

O usando stream.pipeline()

```
const pipeline = util.promisify(stream.pipeline);

const writable = fs.createWritableStream('./file');

(async function() {
  const readable = Readable.from(iterable);
  await pipeline(readable, writable);
})();
```

## Compatibilidad con versiones antiguas de Node.js

Tras Node.js 0.10 la iterfaz de readable se simplificó, pero también pasó a ser menos poderosa y útil.

- En lugar de esperar llamadas de read(), los eventos 'data' serán emitidos inmediatamente. Las aplicaciones que necesiten mejorar su rendimiento podrían necesitar algo de trabajo para decidir cómo tratar los datos en los que requieren ser guardados los datos leídos en buffers con lo que los datos no se pierdan.
- pause() era advirtorio, ahora es garantizado. Esto significa que era necesario estar preparado para recibir 'data' incluso cuando  el stream estaba en pause.

## readable.read(0)

Hay algunos casos en los que es necesario hacer un refresh de los mecanismos del stream, sin consumir datos. Se puede llamar a readable.read(0), que siempre retornará null.

Si el buffer interno de lectura está por debajo del highWaterMark, y el stream no está leyendo, llamar a stream.read(0) realizará un _read() de bajo nivel.

Mientras la mayoría de aplicaciones nunca lo van a necesitar, hay situaciones dentro de node.js donde esto se hace, particularmente de manera intera en la clase.

## readable.push("")

No es recomendable su uso

Esto llama a readable.push() y provoca que el proceso de lectura se produzca. Pero ir vacío no genera datos para ser consumidos.

## hightWaterMark discrepancia tras llamar a readable.setEncoding()

Usar setEncoding() cambiará el comportamiento de cómo highWaterMark opera en no objectMode

Típicamente, el tamaño del buffer es medido con highWaterMark en bytes. Sin embargo,tras llamar a setEncoding() la función de comparación comenzará a medir el tamaño del buffer en caracteres.

No es un problema común en casos de latin1 o acii. Pero se debe tener en cuenta cuando se trabaja con strings ya que puede contener caracteres multi-byte.
