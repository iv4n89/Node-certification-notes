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


