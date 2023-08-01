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
