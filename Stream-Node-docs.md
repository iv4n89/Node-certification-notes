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
