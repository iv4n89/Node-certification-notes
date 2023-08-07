## Path

El módulo node:path provee utilidades para trabajar con ficheros y directorios. Puede ser usado como:

```javascript
const path = require('node:path');
```

## Windows vs POSIX

El comportamiento de path depende del SO. Si estamos en Windows, path asumirá que usamos el formato de windows.

POSIX:
```javascript
path.basename('C:\\temp\\myfile.html');
// Retorna 'C:\\temp\\myfile.html'
```

Windows
```javascript
path.basename('C\\temp\\myfile.html');
// Retorna 'myfile.html'
```

POSIX y Windows:

```javascript
path.win32.basename('C:\\temp\\myfile.html');
// Return myfile.html
```

POSIX y Windows:

```javascript
path.posix.basename('/tmp/myfile.html');
// Return myfile.html
```

### path.basename(path[, suffix])

- path `string`
- suffix `string` Un sufijo opcional a remover
- return `string`


Retorna la última porción de un path, similar a basename de Unix. 

```javascript
path.basename('/foo/bar/baz.html');
// Return baz.html

path.basename('/foo/bar/baz.html', '.html');
// Return baz
```

En windows los tipos son case sensitive.


### path.delimiter

- `string`


Provee el delimitador según el tipo de SO.

- ';' para Windows
- ':' para POSIX


POSIX
```javascript
console.log(process.env.PATH);
// '/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/bin'

process.env.PATH.split(path.delimiter);
// ['/usr/bin', '/bin', '/usr/sbin', '/sbin', '/usr/local/bin']
```

Windows
```javascript
console.log(process.env.PATH)
// 'C:\Windows\system32;C:\Windows;C:\Program Files\node\'

process.env.PATH.split(path.delimiter);
// ['C:\\Windows\\system32', 'C:\\Windows', 'C:\\Program Files\\node\\']
```


### path.dirname(path)

- path `string`
- return `string`


Retorna el nombre del directorio de un path, similar a dirname de Unix.

```javascript
path.dirname('/foo/bar/baz.html');
// /foo/bar
```

TypeError si path no es string


### path.extname(path)

- path `string`
- return `string`


retorna la extensión del path, desde la última ocurrencia de . al final del string. Si no hay un . en la última porción del path, o no hay . más que al inicio, se devuelve un string vacío

```javascript
path.extname('index.html);
// .html

path.extname('index.coffe.md')
// .md

path.extname('index.')
// .

path.extname('index')
//

path.extname('.index')
//

path.extname('.index.md')
// .md
```

TypeError si path no es string


### path.format(pathObject)

- pathObject `Object` Cualquier JS object que tenga las props:
  - dir `string`
  - root `string`
  - base `string`
  - name `string`
  - ext `string`
- return `string`


Retorna el string path de un objeto de path. Opuesto a path.parse.

Recordar que:
- Si pathObject.dir está pathObject.root es ignorado.
- Si pathObject.base está pathObject.ext y pathObject.name son ignorados


Ejemplo en POSIX:

```javascript
// Si dir, root y base son dados `${dir}${path.sep}${base}` será retornado. root se ignora
path.format({
  root: '/ignored',
  dir: '/home/user/dir',
  base: 'file.txt'
});
// /home/user/dir/file.txt

path.format({
  root: '/',
  base: 'file.txt,
  ext: 'ignored'
});
// /file.txt

path.format({
  root: '/',
  name: 'file',
  ext: '.txt'
});
// /file.txt
```

Windows

```javascript
path.format({
  dir: 'C:\\path\\dir,
  base: 'file.txt'
});
// C:\\path\\dir\\file.txt
```


### path.isAbsolute(path)

- path `string`
- return `boolean`



Determina si un path es absoluto

Si su length es cero, retorna false.


POSIX:
```javascript
paht.isAbsolute('/foo/bar'); // true
path.isAbsolute('/bar/..'); // true
path.isAbsolute('qux/'); // false
path.isAbsolute('.'); // false
```

Windows
```javascript
path.isAbsolute('//server'); // true
path.isAbsulute('\\\\server'); // true
path.isAbsolute('C:/foo/..'); // true
path.isAbsolute('C:\\foo\\..'); // true
path.isAbsolute('bar\\baz'); // false
path.isAbsolute('bar/baz'); // false
path.isAbsolute('.'); // false
```

TypeError si path no es string


### path.join([...paths])

- paths `string` Una secuencia de segmentos de path
- return `string`


Junta todos los segmentos de path dados usando los separadores y delimitadores del SO, después lo normaliza.

Los segmentos de 0 length son ignorados

```javascript
path.join('/foo', 'bar', 'baz/asdf', 'quux', '..');
// /foo/bar/baz/qux

path.join('foo', {}, 'bar');
// TypeError
```


### path.normilize(path)

- path `string`
- return `string`


Normaliza el path dado, resolviendo segmentos de . y ..

Cuando se ponen varios segmentos con caracteres / seguidos, se normaliza también.

Un path de 0 length retorna ., lo que es el dir actual.

POSIX:
```javascript
path.normalize('/foo/bar//baz/asdf/quux/..');
// return '/foo/bar/baz/asdf'
```

Windows
```javascript
path.normalize('C:\\temp\\\foo\\bar\\..\\');
// C:\\temp\\foo

path.normalize('C:////temp\\\\/\\/\\/foo/bar');
// C:\\temp\\foo\\bar
```

TypeError si path no es string


### path.parse(path)

- path `string`
- return `Object`
  - dir `string`
  - root `string`
  - base `string`
  - name `string`
  - ext `string`


POSIX
```javascript
path.parse('/home/user/dir/file.txt');
// { root: '/',
//   dir: '/home/user/dir',
//   base: 'file.txt',
//   ext: '.txt',
//   name: 'file' }
```

```
┌─────────────────────┬────────────┐
│          dir        │    base    │
├──────┬              ├──────┬─────┤
│ root │              │ name │ ext │
"  /    home/user/dir / file  .txt "
└──────┴──────────────┴──────┴─────┘
(All spaces in the "" line should be ignored. They are purely for formatting.)
```


### path.posix

- `Object`


Propiedad que provee acceso a la implementación específica de path para POSIX

Accesible desde require('node:path').posix o require('node:path/posix')


### path.relative(from, to)

- from 'string'
- to 'string'
- return 'string'


Retorna un relativo desde from a to basado en el directorio de trabajo actual. Si from y to se resuelven en el mismo directorio se devuelve un 0 length string.

Si se pasa un 0 length a from o to se usará el directorio actual.

POSIX
```javascript
path.relative('/data/orandea/test/aaa', '/data/oreandea/impl/bbb');
// '../../impl/bbb'
```


### path.resolve([...paths])

- ...paths `string` Una secuencia de segmentos de strings
- return `string`


Resuelve una secuencia de paths o segementos a un absolute path

Son procesados de izquierda a derecha, preponiendo cada segmento hasta obtener un absolute path. 

Si después de procesar todos los segmentos no se obtiene un absolute path, se usa el directorio actual.

El resultado es normalizado

Los segmentos de 0 length son ignorados.

Si no se pasan segmentos retorna un absolute path al directorio de trabajo actual

```javascript
path.resolve('/foo/bar', './baz');
// /foo/bar/baz

path.resolve('/foo/bar', '/tmp/file/');
// /tmp/file

path.resolve('wwwroot', 'static_files/png/', '../gif/image.gif');
// Directorio de trabajo actual es /home/myself/node
// retorna /home/myself/node/wwwroot/static_files/gif/image.gif
```


### path.sep

- `string`


Separador específico de cada SO

`\` en Windows
`/` en POSIX

POSIX
```javascript
'foo/bar/baz'.split(path.sep)
// ['foo', 'bar', 'baz']
```

Windows
```javascript
'foo\\bar\\baz'.split(path.sep)
// ['foo', 'bar', 'baz']
```


### path.toNamespacedPath(path)

- path `string`
- return `string`


Solo windows. Retorna un equivalente con prefijo de namespace. Si path no es un string retornará sin modificaciones.


### path.win32

- `Object`


Implementación específica para Windows.

Accesible por require('node:path').win32 o require('node:path/win32')
