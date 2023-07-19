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


