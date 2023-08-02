## Modules: CommonJS modules

CommonJS modules son la manera original de JavaScript para Node.js de usar paquetes. Node también soporta ECMAScript modules usados por navegadores y otros runtimes de JS

En Node.js cada fichero es tratado como un módulo separado. Por ejemplo, un fichero llamado `foo.js`:

```
const circle = require('./circle.js');
console.log(`The area of a circle of radius 4 is %{circle.area(4)}`);
```

En la primera línea, `foo.js` carga un módulo `circle.js` que está en el mismo directorio que `foo.js`.

Aquí está el contenido de `circle.js`:

```
const { PI } = Math;

exports.area = (r) => PI * r ** 2;

exports.circumference = (r) => 2 * PI * r;
```

El módulo `circle.js` ha exportado las funciones `area()` y `circumference()`. Las funciones y objetos son añadidos al root de un módulo especificando propiedades adicionales en el objeto especial `exports`.

Las variables locales del módulo serán privadas, ya que el módulo está encapsulado en una función por Node.js. En este ejemplo, la variable `PI` es privada para `circle.js`.

La propiedad `module.exports` puede ser asignada a un nuevo valor (como una función o un objeto).

`bar.js` hace uso del módulo `square`, que exporta la clase `Square`:

```
const Square = require('./square.js');
const mySquare = new Square(2);
console.log(`The area of mySquare is ${mySquare.area()}`);
```

El módulo `square` es definido en `square.js`:

```
// Asignar a exports no modifica el módulo, se debe usar module.exports
module.exports = class Square {
  constructor(width) {
    this.width = width;
  }
  area() {
    return this.width ** 2;
  }
};
```

El sistema de CommonJS module es implementado en el module core module.

## Enabling

Node.js tiene dos sistemas de módulos: CommonJS y ECMAScript modules.

Por defecto, Node.js tratará lo siguiente como CommonJS modules:

- Ficheros con extensión `.cjs`.
- Ficheros con extensión `.js` cuando el padre `package.json` más cercano contiene un campo <span style="color: green;">"type"</span> con el valor de "commonjs".
- Ficheros con extensión `.js` cuando el padre `package.json` más cercano no contiene un campo "type". Se debe incluir el campo "type", incluso en paquetes cuyos componentes son todos CommonJS. Ser expícito con `type` hará las cosas más fáciles para las build tools y loaders para determinar cómo los ficheros del paquete deben ser interpretados.
- Ficheros con extensión que no son `.mjs`, `.cjs`, `.json`, `.node`, o `.js` (cuando el padre `package.json` contiene el campo "type" con valor `"module"`, estos ficheros serán reconocidos como CommonJS modules sólo si son incluídos con require()).

Llamar a `require()` siempore usa el loader de CommonJS module. Llamar a `import()` siempre usa el ECMAScript module loader.

## Accediendo al main module

Cuando un fichero es ejecutado directamente desde Node.js, require.main es seteado a este módulo. Esto significa que es posible determinar si un fichero está siendo ejecutado directamente testeando `require.main === module`.

Para el fichero `foo.js`, esto será `true` si se corre via `node foo.js`, pero `false` si se ejecuta por `require('./foo')`.

Cuando el entry point no es un CommonJS module, `require.main` es `undefined`, y el main module no puede ser usado.

## Package manager tips

La semántica de Node.js `require()` fue diseñada para ser sufientemente general como para ser soportar estructuras de ficheros razonables. Programas de gestión de paquetes como `dpkg`, `rpm` o `npm` lo encontrarán posible para construir paquetes nativos desde módulos de Node.js sin modificación.

Digamos que queremos tener la carpeta `/usr/lib/node/<paquete>/<version>, que tiene el contenido específico de una versión del paquete.

Los paquetes pueden depender de otros. Para instalar el paquete `foo`, puede ser necesario instalar una versión específica de `bar`. El paquete `bar` puede tener dependencias, y en algunos casos, éstos pueden colisionar o crear dependencias cíclicas.

Ya que NOde.js busca el `realpath` de cualquier módulo que carga (resuelve symlinks) y entonces revisa sus dependencias en node_modules, esta situación puede ser resuelta de la siguiente manera:

- `/usr/lib/node/foo/1.2.3/`: Contiene el paquete `foo`, version 1.2.3.
- `/usr/lib/node/bar/4.3.2/`: Contiene el paquete `bar` del cual `foo` depende.
- `/usr/lib/node/foo/1.2.3/node_modules/bar`: Symbolic link a `/usr/lib/node/bar/4.3.2/`.
- `/usr/lib/node/bar/4.3.2/node_modules/*`: Symbolic link a los paquetes de los que `bar` depende.

Con esto, aunque hayan dependencias cíclicas o conflictos, todos los módulos encontrarán las versiones de las dependencias que necesitan.

Cuando el código en el paquete `foo` llama a `require('bar')`, obtendrá el contenido de la versión que tiene symbolic link en `/usr/lib/node/foo/1.2.3/node_modules/bar`. Entonces, cuando el código de `bar` llama a `require('quux)`, obtendrá la versión que ha sido symblinked en `/usr/lib/node/bar/4.3.2/node_modules/quux`.

Para hacer el proceso de búsqueda de módulos aún más óptimo, en lugar de ponder los paquetes directamente en `/usr/lib/node`, podemos ponerlos en `/usr/lib/node_modules/<name>/<version>`. Entonces node.js no estará tratando de encontrar dependencias en `/usr/node_modules` o `/node_modules`.

Para hacer disponibles los módulos en Node.js REPL, puede ser útil también añadir a `/usr/lib/node_modules` a la variable de entorno `$MODE_PATH`. Debido a que la búsqueda usando `node_modules` es relativa y basada en el path real de los ficheros llamando a `require()`, los paquetes pueden estar en cualquier lugar.

## La extensión .mjs

Debido a la naturaleza síncronade `require()`, no es posible usarlo para cargar módulos ECMAScript. Tratar de hacerlo lanzará un error ERR_REQUIRE_ESM. Usar import() en su lugar.

La extensión `.mjs` está reservada para módulos de ECMAScript que no pueden ser cargados via `require()`.

## Todo junto

Para obtener el filename exacto que será cargado cuando `require()` es llamado, usar la función `require.resolve()`.

A continuación, el pseudocódigo del algoritmo de `require()` a alto nivel:

```
require(x) desde módulo en path Y
1. Si x es un core module:
  a. retorna el core module.
  b. STOP
2. Si x comienza por '/'
  a. setea Y para ser el file system root
3. Si x comienza por './', '/' o '../':
  a. LOAD_AS_FILE(Y + x)
  b. LOAD_AS_DIRECTORY(Y + x)
  c. Lanza 'not found'
4. Si x comienza por '#'
  a. LOAD_PACKAGE_IMPORTS(x, dirname(Y))
5. LOAD_PACKAGE_SELF(x, dirname(Y))
6. LOAD_NODE_MODULES(x, dirname(Y))
7. Lanza 'not found'

LOAD_AS_FILE(x)
1. Si x es un fichero, carga x con su extensión. STOP
2. Si x.js es un fichero, carga x.js como JS text. STOP
3. Si x.json es un fichero, carga x.json como un objeto de JS. STOP
4. Si x.node es un fichero, carga x.node como un addon binario. STOP

LOAD_INDEX(x)
1. Si x/index.js es un fichero, carga x/index.js como JS. STOP
2. Si x/index.json es un fichero, carga x/index.json como un objeto de JS. STOP
3. Si x/index.node es un fichero, carga x/index.node como un addon binario. STOP

LOAD_AS_DIRECTORY(x)
1. Si x/package.json es un fichero
  a. Parsea x/package.json y busca su campo 'main'
  b. Si 'main' es falsy, pasar a 2
  c. let M = X + (json main field)
  d. LOAD_AS_FILE(M)
  e. LOAD_INDEX(M)
  f. LOAD_INDEX(X) DEPRECATED
  g. Lanza 'not found'
2. LOAD_INDEX(x)

LOAD_NODE_MODULES(x, START)
1. let DIRS = NODE_MODULES_PATHS(START)
2. for DIR of DIRS:
  a. LOAD_PACKAGE_EXPORTS(x, DIR)
  b. LOAD_AS_FILE(DIR/x)
  c. LOAD_AS_DIRECTORY(DIR/x)

NODE_MODULES_PATHS(START)
1. let PARTS = path split(START)
2. let I = count of PARTS - 1
3. let DIRS = []
4. while I >= 0,
  a. Si PARTS[I] === 'node_modules' CONTINUE
  b. DIR = path join(PARTS[0 .. I] + 'node_modules')
  c. DIRS = DIR + DIRS
  d. let I = I - 1
5. return DIRS + GLOBAL_FOLDERS

LOAD_PACKAGE_IMPORTS(x, DIR)
1. Encuentra el package scope más cercano a DIR
2. Si no encuentra scrop, return
3. Si es SCOPE/package.json 'imports' es null o undefined, return
4. let MATCH = PACKAGE_IMPORTS_RESOLVE(x, pathToFileURL(SCOPE), ['node', 'require']) definido en el ESM resolver.
5. RESOLVE_ESM_MATCH(MATCH)

LOAD_PACKAGE_EXPORTS(x, DIR)
1. Tratar de interpretar x como una combinación de NAME y SUBPATH donde el nombre debe tener un @scope/ prefix y el subpath comenzar con /
2. Si x no tiene este patrón o DIR/NAME/package.json no es un fichero, return.
3. Parsear DIR/NAME/package.json y buscar el campo 'exports'
4. Si 'exports' es null o undefined, return
5. let MATCH = PACKAGE_EXPORTS_RESOLVE(pathToFileURL(DIR/NAME), '.' + SUBPATH, 'package.json' 'exports', ['node', 'require'])
6. RESOLVE_ESM_MATCH(MATCH)

LOAD_PACKAGE_SELF(x, DIR)
1. Buscar el package scope más cercano SCOPE a DIR
2. Si el scope no es encontrado, return.
3. Si el SCOPE/package.json 'exports' es null o indefined, return.
4. Si el SCOPE/package.json 'name' no es el primer argumento de x, return.
5. let MATCH = PACKAGE_EXPORTS_RESOLVE(pathToFileURL(SCOPE), '.' + x.slice('name'.length), 'package.json' 'exports', ['node', 'require'])
6. RESOLVE_ESM_MATCH(MATCH)

RESOLVE_ESM_MATCH(MATCH)
1. let RESOLVED_MATCH - fileURLToPath(MATCH)
2. Si el fichero en RESOLVED_PATH existe, cargar RESOLVED_PATH con su extensión. STOP
3. Lanza 'not found'
```

## Caching

Los módulos son capturados después de la primera vez que sean cargados. Esto significa que (entre otras cosas) en todas las llamadas a `require('foo')` se obtendrá exactamente el mismo objeto retornado, si es resuelto al mismo fichero.

`require.cache` no es modificado, muchas llamadas a `require('foo')` no causará que el módulo sea ejecutado varias veces. Esto es una feature importante. Con esto, objetos 'parcialmente acabados' pueden ser retornados, pudiendo cargar dependencias transitivas incluso si esto causa una dependencia cíclica.

Si se quiere ejecutar varias veces un módulo, exportar una función y llamarla.

## Advertencias sobre module caching

Los módulos son capturados basándose en su filename. Ya que los módulos pueden ser resueltos a un filename diferente basado en la posición del módulo llamado (cargando desde node_modules), no está garantizado que require('foo') siempre retorne el mismo objeto, puede resolver a ficheros diferentes.

Adicionalmente, en sistemas de fichero case-sensitive, diferentes filename resueltos pueden apuntar al mismo fichero, pero el cache será aún tratado como módulos diferentes y recargará el fichero múltiples veces.

## Core modules

Node.js tiene muchos módulos compilados en binario.

Los core modules son definidos dentro del source de Node.js y están en la carpeta /lib.

Core modules pueden identificarse usando el prefijo node:, en cuyo caso hace bypass al cache de require. Por ejemplo, require('node:http') diempre retornará el builtin http module, incluso si hay un entrada en require.cache con el mismo nombre.

Algunos core modules son siempre preferencialmente cargados si su identificador es pasado a require(). Por ejemplo, require('http') siempre carga el builtin http module, incluso si hay un fichero del mismo nombre.

## Ciclos

Cuando hay llamadas circulares a require(), un módulo puede no haber finalizado su ejecución cuando es retornado.

Considera esta situación:

`a.js`:

```
console.log('a starting');
exports.done = false;
const b = require('./b.js');
console.log('in a, b.done + %j', b.done);
exports.done = true;
console.log('a done');
```

`b.js`:

```
console.log('b starting');
exports.done = false;
const a = require('./a.js');
console.log('in b, a.done = %j', a.done);
exports.done = true;
console.log('b done');
```

`main.js`:

```
console.log('main starting');
cosnt a = require('./a.js');
const b = require('./b.js');
consonle.log('in main, a.done = %j, b.done = %j', a.done, b.done);
```

Cuando main.js carga a.js, entonces a.js carga b.js. En este punto, b.js trata de cargar a.js. Para prevenir un infinite loop, se retorna un objeto export de a.js sin acabar al b.js module. b.js entonces termina de cargar, y sus exports son provistos a a.js.

Al mismo tiempo main.js carga ambos módulos, ambos acabados. El output de este programa será entonces:

```
$node main.js
main starting
a starting
b starting
in b, a.done = false
b done
in a, b.done = true
a done
in main, a.done = true, b.done = true
```

## File modules

Si el filename exacto no es encontrado, entonces Node.js tratará de cargar el filename requerido con las extensiones .js, .json y finalmente .node. Cuando carga un fichero que tiene una extensión diferente (ej. .cjs), su nombre completo debe ser pasado a require(), incluyendo su file extension (ej. require('./file.cjs')).

Los ficheros .json son parseados como JSON text files, los .node son interpretados como addons compilados cargados con process.dlopen(). Cualquier otra extensión son parseadas como JS text files.

Un módulo requerido precedido por '/' es un path absoluto al fichero. Ejemplo, require('/home/marco/foo.js') cargará el fichero en /home/marco/foo.js.

Un módulo requerido precedido por './' es relativo al fichero que lo llama con require(). Esto es, cicrcle.js debe estar en el mismo directorio que foo.js para encontrarlo con require('./circle').

Sin agregar '/', './' o '../' para indicar fichero, el módulo debe ser un core module o ser cargado desde node_modules.

Si se da un path que no existe a require(), lanzará un MODULE_NOT_FOUND

## Carpetas como módulos (legacy)

Hay 3 formas en las que una carpeta puede ser pasado a require() como argumento.

La primera es crear un package.json en el root de la carpeta, que especifique un main module. Un ejemplo de package.json debe lucir como:

```
{
  "name": "some-library",
  "main": "./lib/some-library.js"
}
```

Si está en una carpeta './some-library', entonces require('./some-library') tratará de cargar './some-library/lib/some-library.js.

Si no hay package.json en el directorio, o el main entry no está o no puede ser resuelto, entonces Node.js tratará de cargar un index.js o index.node desde este directorio. Por ejemplo, no hay un package.json en el ejemplo previo, entonces require('./some-library') tratará de cargar:

- ./some-library/index.js
- ./some-library/index.node

Si estos intentos fallan, entonces Node.js reportará el módulo completo como faltante con el error por defecto:

```
Error: Cannot find module 'some-library'
```

En cualquiera de los 3 casos, un import('./some-library') resultaría en un ERR_UNSUPPORTED_DIR_IMPORT error. Usar subpath exports o subpath imports puede proveer el mismo contenido organizacional que benefica a carpetas y módulos, y functiona tanto con require como import.

## Cargar desde carpetas node_modules

Si el identificador del módulo pasado a require() no es un core module, y no comienza por '/', './' o '../', entonces Node.js busca en el módulo actual y agrega /node_modules, y trata de cargar el módulo desde este lugar. Node.js no va a tratar de agregar un node_modules a un path que ya acaba por node_modules.

Si no es encontrado aquí, entonces se mueve al directorio padre, y así hasta llegar a la raiz del file system.

Por ejemplo, si un fichero en '/home/ry/projects/foo.js' llamó a require('bar.js'), entonces Node.js buscará en los siguientes lugars, en este orden:

- /home/ry/projects/node_modules/bar.js
- /home/ry/node_modules/bar.js
- /home/node_modules/bar.js
- /node_modules/bar.js

Esto permite a los programas encontrar sus dependencias, evitando un crash.

Es posible requerir ficheros específicos o sub módulos distribuidos incluyendo un sufijo de path después del nombre del módulo. Por ejemplo require('example-module/path/to/file') resolvería path/to/file relativo a donde example-module está. El path sufijo sigue la misma semántica que los módulos.

## Cargar desde carpetas globales

Si la variable de entorno NODE_PATH está seteada a una lista de paths absolutas delimitadas por coma, entonces Node.js buscará aquellas partes de módulos si ellos no son encontrados.

En Windows, NODE_PATH es delimitado por ;

NODE_PATH fue creado originalmente para ayudar en la carga de módulos desde varios paths antes del actual algoritmo de module resolution.

NODE_PATH es aún soportado, pero menos necesario ahora que el ecosistema de Node.js tiene una conveción para encontrar módulos dependientes. Algunos deploys que usan NODE_PATH muestran comportamientos sorprendentes cuando las personas no saben que necesitan setear NODE_PATH. Algunas veces las dependencias de un móduo cambian, causando que una versión diferente (o incluso un módulo distinto) a ser cargado ya que se busca en NODE_PATH.

Adicionalmente, Node.js buscará en las siguientes GLOBAL_FOLDERS:

- $HOME/.node_modules
- $HOME/.node_libraries
- $PREFIX/lib/node

Donde $HOME es el directorio home del usuario, y $PREFIX es el node_prefix configurado.

Se recomienda colocar las dependencias en node_modules.

## El module wrapper

Antes de que el código de un módulo sea ejecutado, Node.js lo envolverá en una función wrapper como:

```
(function(exports, require, module, __filename, __dirname) {
  // El cóidog del módulo vive aquí
});
```

Haciendo esto Node.js consigue:

- Mantiene variables en el top level scope (definidas con var, const o let) al módulo en lugar el global object.
- Ayuda a proveer algunas variables que parecen globales pero son del módulo, como:
  - Objetos module y exports que el implementor puede usar para exportar valores desde el módulo.
  - Las variables __filename y __dirname, conteniendo el absoluto filename y dir path.

## El scope del módulo

### __dirname

- <string>

El nombre del directorio del módulo actual. Este es el mismo que path.dirname(__filename).

Ejemplo: correr node example.js desde /Users/mjr

```
console.log(__dirname);
// Imprime /Users/mrj
console.log(path.dirname(__filename));
// Imprime /Users/mrj
```

### __filename

- <string>

El nombre del fichero del actual módulo. Es el fichero del módulo resuelto su symlinks a absolute path

Para un main program esto no es necesariamente lo mismo que el file name usado en el command line

Ejemplos:

Correr node example.js desde /Users/mjr

```
console.log(__filename);
// Imprime /Users/mjr/example.js
console.log(__dirname);
// Imprime /Users/mjr
```

Dados dos módulos: a y b, donde b es una dependencia de a y donde a es un directorio estructurado de:

- /Users/mjr/app/a.js
- /Users/mjr/app/node_modules/b/b.js

Referencias a __filename dentro de b.js retornará /Users/mjr/app/node_modules/b/b.js mientras que referencias a __filename dentro de a.js retornará /Users/mjr/app/a.js

### exports

- <Object>

Una referencia al module.exports que más cercano al tipo.

### module

- <module>

Una referencia al módulo actual. En particular, module.exports es usado para definir qué exporta un módulo y hace disponible con require().

### require(id)

- id <string> Nombre del módulo o path
- return: <any> El contenido del módulo exportado.

Usado para importar módulos, JSON y local files. Los módulos pueden ser importados desde node_modules. Local modules y JSON files pueden ser importados usando un path relativo (./, ./foo, ./bar/baz, ../foo) que será resuelto contra el directorio llamado por __dirname (si está) o el directorio de trabajo actual. El path relativo de POSIX style son resueltos en un formato independiente de OS, por lo que funcionará tanto en Windows como en Unix.

```
// Imprtar un local module con un path relativo a __dirname o el directorio de trabajo actual.
const myLocalModule = require('./path/myLocalModule');

// Importar un JSON
const jsonData = require('./path/filename.json');

// Importar un módulo desde node_modules o Node.js built-in module:
const crypto = require('node:crypto');
```

### require.cache

- <Object>

Los módulos son cacheados en este objeto cuando son requeridos. Eliminando una key value de este objeto, el siguiente require recargará el módulo. No se aplica a addons nativos, para los cuales un reload resultará en error.

Agregar o reemplazar entradas es posible. Este caché es checkeado antes de los built-in modules y si un nombre no matchea con un built-in módulo es agregado al cache, solo node:-prefixed require reciben el módulo built-in module.

```
const assert = require('node:assert');
const realFs = require('node:fs');

const fakeFs = {};
require.cache.fs = { exports: fakeFs };

assert.strictEqual(require('fs'), fakeFs);
assert.strictEqual(require('node:fs), realFs);
```

### require.main

- <module> | <undefined>

El objeto Module representa un entry script cargado cuando el proceso Node.js es lanzado, o undefines si el entry point del programa no es un CommonJS module.

En entry.js script:

```
console.log(require.main);
```

```
node entry.js
```

```
Module: {
  id: '.',
  path: '/absolute/path/to',
  exports: {},
  filename: '/absolute/path/to/entry.js',
  loaded: false,
  children: [],
  paths:
  [ '/absolute/path/to/node_modules',
    '/absolute/path/node_modules',
    '/absolute/node_modules',
    '/node_modules' ]}
```

### require.resolve(request[, options ])

- request <string> El module path a resolver
- options <Object>
  - paths <string[]> Paths a resolver de las localizaciones de los módulos. Si está, estos paths son usados en lugar de los default reolution paths, con la excepción de GLOBAL_FOLDERS como $HOME/.node_modules.
- return <string>

Usa el interno require() para buscar la localización de un módulo, pero en lugar de cargar el módulo, retorna el resolved filename

Si el módulo no puede ser encontrado, un error MODULE_NOT_FOUND es lanzado.

require.resolve.paths(request)

- request <string> El modulo path cuyos paths van a ser usados.
- return <string[]> | <null>

Retorna un array que contiene los paths buscados durante la resolución de request o null si el request string referencia un core module, como http o fs.

## El objeto module

En cada módulo, la variable module es una referencia al objeto que representa el módulo actual. Por conveniencia, module.exports es también accesible desde el exports module-global. modulo no es global, sino local a cada módulo.

### module.children

- <module[]>

Objetos module requreidos la primera vez

### module.exports

- <Object>

El objeto module.exports es creado por el sistema Module. Algunas veces esto no es aceptable, muchos quieren que sus módulos sean isntancias de alguna clase. Para hacer esto, se asigna el objeto exportado a module.exports. Asignar el objeto deseado a exports sólo lo hará disponible en la variable local exports.

Por ejemplo, suponemos que estábamos haciendo un módulo llamado a.js:

```
const EventEmitter = require('node:events');

module.exports = new EventEmitter();

// Hacer algo, y después de un tiempo emitir el evento 'ready' desde el propio módulo.

setTimeout(() => {
  module.exports.emit('ready');
}, 1000);
```

Entonces en otro fichero podemos:

```
const a = require('./a');
a.on('ready', () => {
  console.log('modulo "a" is ready');
});
```

La asignación a module.exports debe ser hecha inmediatamente. No puede ser llamada en callbacks. Esto no funcionará:

`k.js`

```
setTimeout(() => {
  module.exports = { a: 'hello' };
}, 0);
```

`y.js`

```
const x = require('./x');
console.log(x.a);
```

## Atajo exports

La variable exports está disponible dentro del scope file-level de un módulo, y le es asignado el valor de module.exports antes de que el módulo sea evaluado.

Permite un atajo, por lo que module.exports.f = ... puede ser escrito como exports.f = ... Sin embargo, hay que tener cuidado ya que como cualquier variable, si un nuevo valor es asignado a exports, no estará más linkeado a module.exports.

```
module.exports.hello = true; // Exportado desde require del módulo
exports = { hello: false }; // No exportado, sólo disponible en el módulo.
```

Cuando el module.exports es completamente reemplazado por un nuevo objeto, es común también reasignar exports:

```
module.exports = exports = function Constructor() {
  // ...
};
```

Para ilustrar el comportamiento, imaginemos una implementación hipotética de require(), que es similar a lo que hace require()

```
function require(/* ... */) {
  const module = { exports: {} };
  ((module, exports) => {
    // Código del módulo aquí
    function someFunc() {}
    exports = someFunc;
    // En este punto, exports ya no es un atajo de module.exports y este módulo aún exporta un objeto vacío.
  }) (module, module.exports);
  return module.exports;
}
```

### modulo.filename

- <string>

El nombre resuelto del módulo. 

### module.id

- <string>

El identificador del módulo. Típicamente esto es el filename ya resuelto.

### module.isPreloading

- Type: <boolean> true si el módulo está corriendo durante la fase de preload de Node.js

### module.loaded

- <boolean>

Si la carga del módulo finalizó, o aún está en proceso de carga.

### module.path

- <string>

El nombre del directorio del módulo. Usualmente el mismo que path.dirname() del module.id

### module.paths

- <string[]>

El path de búsqueda del módulo

### module.require(id)

- id <string>
- return <any> contenido del módulo exportado.

El método module.require(id) provee una manera de cargar módulos como is rquire() fuese llamado desde el módulo original.

Para hacer esto, es necesario obtener la referencia al objeto module. Ya que require() retorna module.exports y el module es típicamente sólo disponible dentro del código del módulo específico, debe ser explícitamente exportado para usarlo.


