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
