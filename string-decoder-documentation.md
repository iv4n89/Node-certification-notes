## String decoder

El módulo node:string-decoder provee un api de decodificación de Buffers en strings de manera que preserve los caracteres multibyte en utf8 y utf16. Puede ser accedido por:

```
const { StringDecoder } = require('node:string_decoder');
```

El siguiente ejemplo muestra el uso básico de la clase StringDecoder:

```
const { StringDecoder } = require('node:string_decoder');
const decoder = new StringDecoder('utf8');

const cent = Buffer.from([0xC2, 0xA2]);
console.log(decoder.write(cent));

const euro = Buffer.from([0xE2, 0x82, 0xAC]);
console.log(decoder.write(euro));
```

Cuando una instancia de Buffer es escrita a la instancia de StringDecoder, un buffer interno es usado para asegurarse que el string decodificado no contiene ningún caracter multibyte incompleto. Se mantienen en el buffer hasta la siguiente llamada a stringDecoder.write() o hasta que stringDecoder.end() es llamado.

Ejemplo donde se pasan los bytes de € en diferentes partes, hasta que se llama a end.

```
const { StringDecoder } = require('node:string-decoder');
const decoder = new StringDecoder('utf8');

decoder.write(Buffer.from([0xE2]));
decoder.write(Buffer.from([0x82]));
console.log(decoder.end(Buffer.from([0xAC]));
```

## Class: StringDecoder

### new StringDecoder([encoding])

- encoding <string> El encoding a usar. Default: utf8

Crea una nueva instancia de StringDecoder

### stringDecoder.end([buffer])

- buffer <Buffer> | <TypedArray> | <DataView> Un Buffer, TypedArray o DataView que contiene los bytes a decodificar.
- return <string>

Retorna cualquier input remanente guardado en el buffer interno como un string. Los bytes que representan caracteres UTF8 y UTF16 incompletos serán reemplazados con caracteres apropiados al character encoding usado.

Si el argumento buffer es provisto, una llamada final a stringDecoder.write() se realiza antes de retornar el input remanente. Después de ser llamado end(), el objeto StringDecoder puede ser usado con nuevo input

### stringDecoder.write(buffer)

- buffer <Buffer> | <TypedArray> | <DataView> Un Buffer, TypedArray o DataView que contiene los bytes a decodificar.
- return <string>

Retorna un string decodificado, asegurándose que cualquier caracter multibyte en Buffer, TypedArray o DataView es omitido del string retornado y guardado en el buffer interno para la siguiente llamada de write() o end()

