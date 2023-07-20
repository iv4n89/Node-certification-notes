## Buffer

Un buffer es un objeto usado para representar una secuencia de bytes de una longitud determinada. Muchas APIs de Node.js soportan Buffers.

La clase Buffer es una clase de Uint8Array y la extiende con métodos que abarcan más casos de uso. Las APIs que soportan Buffers soportan también Uint8Array.

La clase Buffer está dentro del scope global, por lo que no necesita una importación.

```
// Crear un Buffer vacío de tamaño 10
const buf1 = Buffer.aloc(10);

// Crear un Buffer de tamaño 10 relleno con bytes cuyo valor representa '1'.
const buf2 = Buffer.aloc(10, 1);

// Crear un buffer no inicializado de tamaño 10.
// Es más rápido que llamar a Buffer.alloc() pero el buffer retornado puede contener datos antiguos que deban ser sobreescritos usando fill(),
// write() u otra función que rellene el buffer con contenido.
const buf3 = Buffer.allocUnsafe(10);

// Crear un Buffer que contiene los bytes [1, 2, 3]
const buf4 = Buffer.from([1,2,3]);

// Crear un buffer que contiene los bytes [1, 1, 1, 1] - las entradas son todas truncadas usando `(value & 255)` para que entren en el rango 0-255
const buf5 = Buffer.from([257, 257.5, -255, '1']);

// Crear un buffer que contiene bytes codificados en UTF-8 para el string 'tést':
// [0x74, 0xc3, 0a9, 0x73, 0x74]) para hexadecimal
// [116, 195, 169, 115, 116] para decimal
cont buf6 = Buffer.from('tést');

// Crear un Buffer que contiene Latin-1 bytes [0x74, 0xe9, 0x73, 0x74]
const buf7 = Buffer.from('tést', 'latin1');
```

## Buffers y codificación de caracteres

Cuando hacemos conversiones entre Buffers y strings se debe especificar una codificación de caracteres. Si no se especifica, utf8 se usa por defecto.

```
const buf = Buffer.from('hello world', 'utf8');

console.log(buf.toString('hex'));
// imprime 686... (muchos números)
console.log(buf.toString('base64'));
// imprime aGVsBG8... (lo que sea)

console.log(Buffer.from('asdfsdf', 'utf8'));
// imprime <Buffer 66 68 71 77 ...>
console.log(Buffer.from('asdfsad', 'ufg16le'));
// imprime <Buffer 66 00 68 00 71 ...>
```

La codificación de caracteres que Node.js soporta actualmente es:

- 'utf8': Codificación multi-byte de caracteres unicode. Muchas webs y otros formatos de documentos usan utf8. Es la codificación de caracteres por defecto. Cuando decodificacom un buffer en string que no contiene datos exclusivamente en utf8, el caracter unicode U+FFFD <?> representará un error.
- 'utf17le': Codificación multi-byte. Cada caracter será codificado usando 2 o 4 bytes. Node.js sólo soporta la variante littel endian de utf16 (de ahí le)
- 'latin1': Es ISO-8859-1. Sólo soporta caracteres unicode desde U+0000 a U+00FF. Cada caracter es codificado usando un único byte. Todo caracter que se pase será truncado y representado por el caracter en este rango.

Convertir un Buffer a string usando uno de estos se llama decodificar, y convertir un string en buffer es codificar.

Node.js también soporta dos codificaciones binario-texto. Para éstos, las convenciones de nombre son revertidas: Convertir de Buffer a string es típicamente llamada codificar, y de string a Buffer decodificar.

- 'base64': Cuando creamos un Buffer desde un string, esta codificación esta codificación también acepta correctamente 'URL y Filename Safe Alphabet'. Los espacios en blanco son ignorados aquí.
- 'hex': Codifica cada caracter en dos caracteres hexadecimales. El truncado de datos puede ocurrir cuando decodificamos strings que contienen exclusivamente caracteres hexadecimales.

También son soportados algunos encoding antiguos:

- 'ascii': Sólo para 7-bit ascii. Cuando se codifica un string a buffer, este es el equivalente a 'latin1'. Cuando se decodifica un buffer a string, usar esta codficación adicionalmente quita el mayor bit de cada byte antes de decodificarlo a 'latin1'. Generalmente no hay razón para usar esta codificación.
- 'binary': Alias para 'latin1'.
- 'ucs2': Alias para 'utf16le'.

```
Buffer.from('lag', 'hex');
// Imprime <Buffer 1a>, truncado cuando el primer valor no es hexadecimal.
// ('g') encotrado.

Buffer.from('la7g', 'hex');
// Imprime <Buffer 1a>, datos truncados cuando acaban en un sólo dígito ('7').

Buffer.from('1634', 'hex');
// Imprime <Buffer 16 34>, todo representado.
```

## Buffers y TypedArrays

Las instancias de Buffer son también instancias de Uint8Array y TypedArray. Todos los métodos de TypedArray están disponibles en Buffer. Aún así hay algunos incompatibilidades entre el API de buffer y el de TypedArray.

En particular:

- Mientras que TypedArray#slice() crea una copia de una parte del TypedArray, Buffer#slice() crea una vista del buffer existente sin copiar. Este comportamiento existe sólo por compatibilidad. TypedArray#subarray() puede ser usado para llegar al comportamiento de de Buffer#slice() en ambos, buffer y TypedArray.
- buf.toString() es incompatible con su equivalente TypedArray.
- Un número de métodos (ej. buf.indexOf() soporta argumentos adicionales.

Hay dos formas de crear nuevos TypedArray desde un buffer:

- Pasando un Buffer a un constructor de TypedArray copiará el contenido del buffer, interpretado como un array de integers, y no como secuencia de bytes.

```
const buf = Buffer.from([1, 2, 3, 4]);
const uint32array = new Uint32Array(buf);

console.log(uint32array);

// Imprime Uint32Array(4) [1, 2, 3, 4]
```

-  Al pasar el ArrayBuffer subyacente al Buffer se creará un TypedArray que compartirá su memoria con el Buffer.

```
const buf = Buffer.from('hello', 'utf16le');
const uint16arr = new Uint16Array(
  buf.buffer,
  buf.byteOffset,
  buf.length / Uint16Array.BYTES_PER_ELEMENT);

console.log(uint16array);

// Imprime Uint16Array(5) [104, 101, 108, 108, 111]
```

Es posible crear un nuevo Buffer que comparta la misma memoria que un TypedArray usando el objeto TypedArray.buffer. Buffer.from() se comporta igual que new Uint8Array() en este contexto

```
const arr = new Uint16Array(2);

arr[0] = 5000;
arr[1] = 4000;

// Copia el contenido de 'arr'
const buf1 = Buffer.from(arr);

// Comparte la memoria con 'arr'
const buf2 = Buffer.from(arr.buffer);

console.log(buf1);
// <Buffer 88 a0>

console.log(buf2);
// <Buffer 88 13 a0 0f>

arr[1] = 6000;

console.log(buf1);
// <Buffer 88 a0>

console.log(buf2);
// <Buffer 88 13 70 17>
```

Cuando se crea un buffer usando el .buffer de TypedArray es posible usar sólo una porción del arrayBuffer pasando los parámetros byteOffset y length

```
const arr = new Uint16Array(20);
const buf = Buffer.from(arr.buffer, 0, 16);

console.log(bug.length);
// 16
```

Buffer.from() y TypedArray.from() son diferentes. Específicamente, la variante de TypedArray acepta un segundo argumento que es un mapping function que es invocado en cada elemento del typed array:

- TypedArray.from(source[, mapFn[, thisArg]])

Buffer.from() no soporta este mapping function:

- Buffer.from(array)
- Buffer.from(buffer)
- Buffer.from(arrayBuffer[, byteOffset[, length]])
- Buffer.from(string[, encoding])

## Buffers e iteraciones

Los buffers pueden ser iterados con un for of:

```
const buf = Buffer.from([1, 2, 3]);

for (const b of buf) {
  console.log(b);
}

// Imprime
// 1
// 2
// 3
```

Adicionalmente, buf.values(), buf.keys() y buf.entries() pueden ser usados para crear iteradores.

## Class: Buffer

La clase Buffer es un tipo global para tratar con datos binarios directamente. Puede ser construído de varias formas.

### Static method: Buffer.alloc(size[, fill[, encoding]])

- size <integer> La longitud deseada para el Buffer
- fill <string> | <Buffer> | <Uint8Array> | <integer> Un valor para pre-rellenar el nuevo buffer. Default 0.
- encoding <string> Si fill es un string, este es su encoding. Default 'utf8'

```
const buf = Buffer.alloc(5);

console.log(buf);
// Imprime: <Buffer 00 00 00 00 00>
```

Si el size es más grande que buffer.constants.MAX_LENGTH o menor a 0, se lanza un ERR_INVALID_OPT_VALUE.

Si fill es especificado, el buffer es inicializado con buf.fill(fill).

```
const buf = Buffer.alloc(5, 'a');

console.log(buf);
// <Buffer 61 61 61 61 61>
```

Si fill y encoding son especificados, el buffer es inicializad llamando buf.fill(fill, encoding).

```
const buf = Buffer.alloc(11, 'Gasdlfjsf=', 'base64');

console.log(buf);
// <Buffer 68 65 6c 6c 6f 20 77 6f 72 6c 64>
```

Llamar a Buffer.alloc() pueede ser más lento que Buffer.allocUnsafe() pero se asegura que los nuevos Buffers nunca contendrán datos sensibles de anteriores buffers, incluyendo datos que no hayan sido posicionados en el buffer.

Un TypeError se lanzará si size no es un número.

### Static method: Buffer.allocUnsafe(size)

- size <integer> La longitud deseada.

Crea un nuevo Buffer de <size> bytes. Si el size es mayor que la constante buffer.constants.MAX_LENGTH o menor a 0, un ERR_INVALID_OPT_VALUE es lanzado.

La memoria del Buffer creado no está inicializado. El contenido del nuevo buffer es no conocido y puede contener datos sensibles. Usar Buffer.alloc() en su lugar para inicializar con 0s.

```
const buf = Buffer.allocUnsafe(10);

console.log(buf);
// Imprime <Buffer ...>

buf.fill(0);

console.log(buf);
// Imprime <Buffer 00 00 00 00 00 00 00 00 00 00 00>
```

Un TypeError se lanza si size no es un número

El módule Buffer pre-posiciona un internal buffer del tamaño Buffer.poolSize que es usado por el pool para el posicionamiento de nuevos Buffers creados con Buffer.allocUnsafe(), Buffer.from(array), Buffer.concat() y el deprecado new Buffer(size); sólo si Buffer.poolSize es mayor a 1

El uso de esta memoria pre-posicionada es la clave entre llamar a Buffer.alloc(size, fill) contra Buffer.allocUnsafe(size).fill(fill). Específicamente, Buffer.alloc(size, fill) nunca serán usados internamente por el pool del Buffer, mientras que Buffer.allocUnsafe(size).fill(fill) será usado por el pool del buffer interno si size es menor o igual a la mitad de Buffer.poolSize. La diferencia es mínima, pero puede ser importante cuando la aplicación requiere el rendimiento adicional que proporciona Buffer.allocUnsafe().

### Static method: Buffer.allocUnsafeSlow(size)

- size <integer> El tamaño deseado del nuevo Buffer.

Asigna un nuevo Buffer del tamaño size de bytes. Si size es mayor... (blabla)

No está inicializado y puede contener datos sensibles. Usar buf.fill(0) para inicializar el Buffer con 0s.

Cuando usamos Buffer.allocUnsafe() para asignar un nuevo Buffer, las asignaciones de menos de 4kb son partidas desde un buffer pre-asignado. Esto permite a aplicaciones evitar crear muchos buffers individuales. Esto mejora tanto el rendimiento como el uso de memoria eliminando la necesidad de trackear y limpiar muchos objetos de ArrayBuffer individuales.

Sin embargo, en el caso que un desarrollador necesite retener un pequeño trozo de datos en la memoria desde un pool por un tiempo indeterminado, sería apropiado crear un buffer un-pooled usando Buffer.allocUnsafeSlow() y copiar los bits relevantes.

```
// Se necestia mantener algunos chunks pequeños en memoria
const store = [];

socket.on('readable', () => {
  let data;
  while (null !== (data = readable.read())) {
    // Asignar los datos retenidos
    const sb = Buffer.allocateUnsafeSlow(10);
    // Copiar los datos a la nueva asignación
    data.copy(sb, 0, 0, 10);
    store.push(sb);
  }
});
```

Un TypeError es lanzado si size no es un número

### Static method: Buffer.byteLength(string[, encoding])

- string <string> | <Buffer> | <TypedError> | <DataView> | <ArrayBuffer> | <SharedArrayBuffer> Un valor para calcular la longitud
- encoding <string> Si `string` es un string, este es su encoding. Default utf8
- returns <integer> El número de bytes contenido dentro de string

Retorna la longitud de bytes cuando se codifica con el encoding pasado. No es lo mismo que String.prototype.length.

Para base64 y hex, esta función asume input válido. Para strings que contienen datos no codificados en base64 o hex (ej. espacio en blanco), el retorno puede ser mayor que la longitud del buffer creado desde el string.

```
const str = '\u00db + \u00bc = \u00bd';

console.log(`${str}: ${str.length} characteres, ` + `${Buffer.byteLength(str, 'utf8')} bytes`);
// 9 characteres - 12 characters
```

### Static method: Buffer.compare(buf1, buf2)

- buf1 <Buffer> | <Uint8Array>
- buf2 <Buffer> | <Uint8Array>
- returns <integer> -1, 0 o 1 dependiendo del resultado de la comparación.

Compara buf1 con buf2, típicamente para tareas de ordenación de instancias de Buffer. Esto es equivalente a llamar buf1.compare(buf2).

```
const buf1 = Buffer.from('1234');
const buf2 = Buffer.from('0123');
const arr = [buf1, buf2];

console.log(arr.sort(Buffer.compare));
// Orden [buf2, buf1]
```

### Static method: Buffer.concat(list[, totalLength])

- list <Buffer[]> | <Uint8Array> Lista de Buffer o Uint8Array a concatenar
- totalLength <integer> Longitud total de Buffers en la lista cuando son concatenados
- returns <Buffer>

Retorna un nuevo Buffer resultado de la concatenación de todos los buffers de una lista.

Si la lista no tiene itemos, o el totalLength es 0, entonces un nuevo Buffer de longitud 0 es retornado.

Si totalLength no se provee, es calculado desde los buffers en la lista agregando sus longitudes

Si totalLength es provisto, es coaccionado a un integer no asignado. Si la combinación de longitudes de los buffers en una lista excede el totalLength, el resultado es truncado a totalLength.

```
// Crea un único Buffer desde una lista de tres instancias de Buffer.

const buf1 = Buffer.alloc(10);
const buf2 = Buffer.alloc(14);
const buf3 = Buffer.alloc(18);
const totalLength = buf1.length + buf2.length + bu3.length;

console.log(totalLength);
// 42

const bufA = Buffer.concat([buf1, buf2, buf3], totalLength);

console.log(bufA);
// <Buffer 00 00 00 ...>

console.log(bufA.length)
// 42
```

Buffer.concat() puede usar el pool interno de Buffer como Buffer.allocUnsafe() hace.

### Static method: Buffer.from(array)

- array <integer[]>

Asigna un nuevo Buffer usando un array de bytes en rango 0 - 255. Las entradas fuera de este rango serán truncadas.

```
// Crea un nuevo Buffer que contiene los bytes de utf-8 del string 'buffer'
const buf = Buffer.from([0x62, 0x75, 0x66, 0x66, 0x65, 0x72]);
```

Un TypeError será lanzado si array no es un Array y otro tipo apropiado para variantes de Buffer.from()

Buffer.from(array) y Buffer.from(string) también pueden usar el pool interno de Buffer como Buffer.allocUnsafe().

### Static method: Buffer.from(arrayBuffer[, byteOffeset[, length]]);

- arrayBuffer <ArrayBuffer> | <SharedArrayBuffer> Un ArrayBuffer, SharedArrayBuffer, por ejemplo la propiedad .buffer de un TypedArray
- byteOffset <integer> Inde del primer byte expuesto. Default 0
- length <integer> Número de bytes expuest. Default arrayBuffer.byteLength - byteOffset

Crea una vista del arrayBuffer sin copiar la memoria subyacente. Por ejemplo, cuando pasamos una referencia del .buffer de un TypedArray, el nuevo Buffer creado comparte algo de memoria asignada como el TypedArray.

```
const arr = new Uint16Array(2);

arr[0] = 5000;
arr[1] = 4000;

// Comparte memoria con 'arr'
const buf = Buffer.from(arr.buffer);

console.log(buf)
// <Buffer 88 13 a0 0f>

// Cambiar el Uint16Array modifica también el Buffer
arr[1] = 6000;

console.log(buf)
// <Buffer 88 13 a0 0f>
```

El byteOffset opcional y length especifican un rango de memoria dentro de arrayBuffer que será compartido con el Buffer.

```
const ab = new ArrayBuffer(10);
const buf = Buffer.from(ab, 0, 2);

console.log(buf.length);
// 2
```

Un TypedError será lanzado si arrayBuffer no es un ArrayBuffer o una variante apropiada.

### Static method: Buffer.from(buffer)

- buffer <Buffer> | <Uint8Array> Un Buffer o Uint8Array existente desde el que copiar datos

Copia los datos del buffer a un nuevo Buffer.

```
const buf1 = Buffer.from('buffer');
const buf2 = Buffer.from(buf1);

buf1[0] = 0x61;

console.log(buf1.toString());
// auffer
console.log(buf2.toString());
// buffer
```

Un TypeError será lanzado si buffer (blabla)

### Static method: Buffer.from(object[, offsetOrEncoding[, length]])

- object <Object> Un objeto que soporta Symbol.toPrimitive o valueOf()
- offsetOrEncoding <integer> | <string> Un byte-offset o encoding
- length <integer> Un length

Para aquellos objetos cuya función valueOf() retorna un valor no estrictamente igual a object, retorna Buffer.from(object.valueOf(), offsetOrEncoding, length)

```
const buf = Buffer.from(new String('this is a test'));
// <Buffer 74 68 ...>
```

Para los objetos que soportan Symbol.toPrimitive, retorna `Buffer.from(object[Symbol.toPrimitive]('string')`, offsetOrEncofing).

```
class Foo {
  [Symbol.toPrimitive]() {
    return 'this is a test';
  }
}

const buf = Buffer.from(new Foo(), 'utf8');
// <Buffer 74 68 ...>
```

Un TypeError será lanzado si object no tiene los métodos mencionados

### Static method: Buffer.from(string[, encoding])

- string <string> Un string codificados
- encoding <string> El encoding de string. Default utf8

Crea un nuevo Buffer que contiene string. El encoding identifica el character encoding a usar cuando convertimos string a bytes.

```
const buf1 = Buffer.from('this is a tést');
const buf2 = Buffer.from('394583475983475983493485739458', 'hex');

console.log(buf1.toString());
// this is a tést
console.log(buf2.toString());
// this is a tést
console.log(buf1.toString('latin1'));
// this is a tÃ©st
```

Un TypedError será lanzado si string no es un string

### Static method: Buffer.isBuffer(obj)

- obj <Object>
- returns <boolean>

Retorna true si obj es un Buffer

### Static method: Buffer.isEncoding(encoding)

- encoding <string> Un character encoding para chequear
- returns <boolean>

```
console.log(Buffer.isEncoding('utf-8'));
// true

console.log(Buffer.isEncoding('hex'));
// true

console.log(Buffer.isEncoding('utf/8'));
// false

console.log(Buffer.isEncoding(' '));
// false
```

### Class property: Buffer.poolSize

- <integer> Default 8192

Es el tamaño en bytes del pre-asignado buffer interno usado para pooling. Puede ser modificado.

### buf[index]

- index <integer>

El index operator `[index]` puede ser usado para optener y setear la posición index en buf. Los valores se refieren a bytes individuales, entre 0 y 255.

Este operador es heredado de Uint8Array. buf[index] retorna undefined si index es negativo o mayor o igual a buf.length, y buf[index] = value no modifica el buffer si index está fuera de rango.

```
// Copiar un string ascii en Buffer un byte cada vez
// Solo funciona con strings ascii, se debe usar Buffer.from() para realizar esta conversión

const str = 'Node.js';
const buf = Buffer.allocUnsafe(str.length);

for (let i = 0; i < str.length; i++) {
  buf[i] = str.charCodeAt(i);
}

console.log(buf.toString('utf8'));
// Node.js
```

### buf.buffer

- <ArrayBuffer> El objeto ArrayBuffer subyacente desde el que el buffer es creado.

Este ArrayBuffer no está garantizado para corresponder exactamente con el Buffer original

```
const arrayBuffer = new ArrayBuffer(16);
const buffer = Buffer.from(arrayBuffer);

console.log(buffer.buffer === arrayBuffer);
// true
```

### buf.byteOffset

- <integer> El byteOffset del buffer subyacente al ArrayBuffer

Cuando seteamos byteOffset el Buffer.from(ArrayBuffer, byteOffset, length), o a veces cuando asociamos un buffer más pequeño que Buffer.poolSize, el buffer no comienza desde el offset 0 en el subyacente ArrayBuffer.

Esto puede causar problemas cuando accedemos al subyacente ArrayBuffer directamente usando buf.buffer, ya que otras partes del ArrayBuffer pueden no estar relacionadas con el Buffer.

Un issue común cuando creamos un TypedArray que comparte memoria con un Buffer es que en este caso uno necesita especificar el byteOffset correctamente.

```
// Crear un buffer más pequeño que Buffer.poolSize
const nodeBuffer = new Buffer.from([0, 1, 2, 3, 4, 5, 6, 7, 8, 9]);

// Cuando convertimos el Node.js Buffer a un Int8Array, usar el bytOffset para referirse sólo
// a la parte de 'nodeBuffer.buffer' que contiene la memoria para 'nodeBuffer'
new Int8Array(nodeBuffer.buffer, nodeBuffer.byteOffset, nodeBuffer.length);
```

### buf.compare(target[, targetStart[, targetEnd[, sourceStart[, sourceEnd]]]])

- target <Buffer> | <Uint8Array> Un Buffer o Uint8Array con el que comparar buf.
- targetStart <integer> Offset dentro de target desde el que comenzar la comparación. Default 0
- targetEnd <integer> Offset dentro de target hasta el que hacer la comparación (no inclusive). Default target.length
- sourceStart <integer> Offset dentro de buf desde el que comenzar la comparación. Default 0
- sourceEnd <integer> Offset dentro de buf hasta el que hacer la comparación (no inclusive). Default buf.length
- returns <integer>

Compara buf con target y retorna un número indicando cuál viene antes, después o si son iguales en orden. La comparación se realiza con la secuencia de bytes en cada Buffer.

- 0 si target es igual a buf.
- 1 si target debe ir antes de buf
- -1 si target debe ir después de buf

```
const buf1 = Buffer.from('ABC');
const buf2 = Buffer.from('BCD');
const buf3 = Buffer.from('ABCD');

console.log(buf1.compare(buf1));
// 0
console.log(buf1.compare(buf2));
// -1
console.log(buf1.compare(buf3));
// -1
```
