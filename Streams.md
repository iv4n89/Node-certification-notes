# Streams

## Qué son los streams

Son _colecciones_ de datos que pueden ser muy grandes, por lo que ser leídos o escritos de una sola vez consumiría mucha memoria. En su lugar, va creando chunks (trozos) que va leyendo o escribiendo.

Todos los streams son instancias de _EventEmitter_. Emiten eventos que pueden ser usados para leer o escribir datos.

## Tipos de streams

Hay 4 tipos fundamentales de streams:

- Readable: Es la abstracción de datos que pueden ser consumidos. Un ejemplo de método es fs.createReadStream
- Writable: Abstracción para un destino al que pueden ser escritos datos. Un ejemplo de método es fs.createWriteStream.
- Duplex: Es tanto readable como writable.
- Transform: Es un duplex que puede modificar los datos. Un ejemplo es zlib.createGzip, que comprime datos usando gzip.

## El método pipe

```
readableSrc.pipe(writableDest);
```

Con la línea de arriba se está creando una tubería que une el contenido de un stream readable con la posibilidad de escribir en otro stream writable.
Se pueden juntar varios, como en el siguiente ejemplo, que el contenido de un stream readable pasa por 2 transform streams antes de ser escrito en un writable stream:

```
readableSrc
  .pipe(transformStream1)
  .pipe(transformStream2)
  .pipe(finalWritableDest);
```

El método pipe devuelve el stream de destino, por lo que puede encadenarse las instrucciones como arriba.
Esto sería equivalente a:

```
a.pipe(b).pipe(c).pipe(d);

a.pipe(b);
b.pipe(c);
c.pipe(d);

## En linux sería equivalente a:
$ a | b | c | d
```

Este método pipe es la manera más sencilla de consumir streams. Se recomienda usar o _pipe_ o _eventos_, pero no mezclarlos.
Normalmente si usamos el método _pipe_ no necesitamos usar _eventos_, pero si necesitamos consumir los streams de una manera más custom los _eventos_ pueden ser una buena manera de seguir.

## Eventos de streams

Por detrás el método _pipe_ realiza algunas funciones por nosotros, como manejo de errores, end-of-file, o las causas que hace que un stream sea más lento que otro.

También podemos consumir streams directamente a través de _eventos_. Este código es equivalente al anterior, pero usando _eventos_:

```
## readable.pipe(writable);

readable.on('data' (chunk) => {
  writable.write(chunk);
});

readable.on('end', () => {
  writable.end();
});
```

Lista de eventos y funciones importantes en los readable y writable streams:

### Readable streams:
#### Eventos:
- data
- end
- error
- close
- readable

#### Funciones
- pipe(), unpipe()
- read(), unshift(), resume()
- pause(), isPaused()
- setEncoding()

### Writable streams:
#### Eventos:
- drain
- finish
- error
- close
- pipe/unpipe

#### Funciones:
- write()
- end()
- cork(), uncork()
- setDefaultEncoding()

Los eventos y funciones están relacionados de alguna manera porque usualmente se usan a la vez.

Los más importantes de los _readable streams_ son:
- data: Emitido siempre que el stream pasa un chunk de datos al consumidor
- end: Emitido cuando no hay más datos a ser consumidos por el stream.

Los eventos más importantes en un _writable stream_ son:
- drain: Señal de que el writable stream puede recibir más datos.
- finish: Emitido cuando todos los datos se han vaciado en el sistema subyacente.

Los eventos y funciones se pueden combinar para hacer un custom y optimizado uso de streams. Para consumir un readable stream podemos usar el método pipe/unpipe, 
o los métodos read/unshift/reasume.
Para consumir un writable stream podemos hacerlo destino de un pipe/unpipe o escribirlo con el método write y un método end cuando hayamos finalizado.

## Modos pausa y fluir de los readable streams

Los readable streams tienen dos modos principales que afectan a la manera en que son consumidos:

- Modo pausado (paused) - Se consume usando el método read() del stream.
- Modo fluir (flowing) - Se consume escuchando los eventos ya que la información fluye constantemente.

También pueden ser llamados modos pull o push.

Todos los readable streams comienzan en modo paused por defecto, pero pueden ser movidos al modo flowing o paused cuando sea necesario. A veces el cambio es automático.

Cuando un readable stream está en modo paused podemos usar el método read() para leer desde el stream bajo demanda. Sin embargo, para un readable stream en modo flowing, los datos fluyen constantemente y necesitamos escuchar los eventos para consumirlos.

En modo flowing los datos pueden perderse si no hay un consumer escuchando. Para este tipo de readable stream necesitamos un data event handler.
Si agregamos un data event handler a un readable stream en modo paused, automáticamente se convierte en modo flowing. Si eliminamos este data event handler, vuelve a ser paused.

Podemos convertirlos manualmente usando los métodos resume() y pause().

![1_HI-mtispQ13qm8ib5yey3g](https://github.com/iv4n89/Node-certification-notes/assets/69988988/2f4535c4-a394-4686-9914-6e3328147d26)

Cuando consumimos un stream con el método pipe no debemos preocuparnos por esto, ya que éste lo hace por nosotros.

## Implementando Streams

Cuando hablamos de streams en Nodejs hay dos tareas principales:

- Implementar streams.
- Consumir streams.

Los implementadores de streams usualmente son los que requieren el módulo stream.

## Implementando un writable stream

Para implementar un writable stream necesitamos usar el costructor de Writable desde el módulo stream.

```
const { Writable } = require('stream');
```

Podemos implementar un writable stream de muchas maneras. Por ejemplo, extendiendo el constructor de Writable:

```
class myWritableStream extends Writable {
}
```

Hay una manera más simple. Podemos crear un objeto desde el constructor de Writable y pasarle un número de opciones. La única opción requerida es un método write que expone un chunk de datos a ser escritos:

```
const { Writable } = require('stream');

const outStream = new Writable({
  write(chunk, encoding, callback) {
    console.log(chunk.toString());
    callback();
  }
});

process.stdin.pipe(outstream);
```

Este método write toma 3 argumentos:

- El chunk es usualmente un buffer a menos que configuremos el stream de manera diferente.
- Encoding es necesario en este caso, pero normalmente podemos ignorarlo.
- Callback es una función que necesitamos llamar después de acabar de procesar el chunk de datos. Es lo que hace ver que la escritura tuvo éxito o no. En caso de fallar, se llamaría al callback con un objeto de error.

En outStream simplemente llamamos a console.log con el chunk como un string y llamamos al callback después de esto sin errores para indicar que salió bien. Esta es una manera muy simple de probar un echo stream.

Para consumir el stream sólo usamos process.stdin, que es un readable stream, por lo que podemos usar pipe junto a nuestro outStream.

Esto es equivalente a process.stdout. Podemos hacer pipe en stdin junto a stdout y obtendremos exactamente el mismo echo con una sola línea: 

```
process.stdin.pipe(process.stdout);
```

## Implementar un Readable Stream


