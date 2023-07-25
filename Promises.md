## Promises en Node.js

JavaScript trabaja sobre un único hilo, esto significa que dos bits de script no pueden trabajar al mismo tiempo. Una promesa es un objeto que representa
la completación eventual (o fallo) de una operación asíncrona, y su valor resultante.

```
const promise = new Promise(function(resolve, reject) {
  // ...
  if (/* algo */) {
    resolve('It worked');
  } else {
    reject(Error('Not working at all...'));
  }
}
```

## Una promesa existe en uno de estos estados

- Pending: Estado incial, ni finalizado ni rechazado. 
- Fultilled: Operación completa con éxito. Llama internamente a onFulfilled()
- Rejected: Operación fallida. Llama internamente a onRejected()

El objeto de promesa funciona como un proxy para un valor no necesariamente conocido cuando la promesa es creada. Permite asociar handlers con un valor de 
operación realizada o razón de fallo.

Esto permite a métodos asíncronos retornar valores como si fuesen métodos síncronos: En lugar de retornar el valor inmediatamente, el método asíncrono retorna
una promesa para devolver el valor en algún punto en el futuro.

```
function noop() {}

function Promise(executor) {
  if (typeof this !== 'object') {
    throw new TypeError('Promises must be constructed via new');
  }
  if (typeof executor !== 'function') {
    throw new TypeError('Promises constructor\' argument is not a function);
  }
  this._deferredState = 0;
  this._state = 0;
  this._value = null;
  this._deferreds = null;
  if (executor === noop) return;
  doResolve(executor, this);
}
```

La propiedad `this._state` puede tener los tres estados señalados arriba:

```
0 - pending
1 - fulfilled con _value
2 - rejected con _value
3 - adopta el state de otra promesa, _value
```

Su valor será siempre 0 cuando creamos una nueva promesa.

Luego `doResolve(executor, this)` se invoca con el ejecutor y la promesa. Su definición:

```
function doResolve(fn, promise) {
  var done = false;
  var resolveCallback = function(value) {
    if (done) return;
    done = true;
    resolve(promise, value);
  };
  var rejectCallback = function(reason) {
    if (done) return;
    done = true;
    reject(promise, reason);
  };
  var res = tryCallTwo(fn, resolveCallback, rejectCallback);
  if (!done && res === IS_ERROR) {
    done = true;
    reject(promise, LAST_ERROR);
  }
}
```

Aquí llama a `tryCallTwo` con ejecutor y 2 callbacks. Los callbacks son los de resolve y reject.

La variable done es usada aquí para asegurarse que la promesa está resuelta o es rechazada sólo una vez, por lo que si se trata de rechazar o resolver de nuevo
acabará la ejecución de la función.

```
function(fn, a, b) {
  try {
    fn(a, b);
  } catch (ex) {
    LAST_ERROR = ex;
    return IS_ERROR;
  }
}
```

Esta función indirectamente llama al ejecutor con dos argumentos, de nuevo los callbacks de resolve y reject.

Si hay un error durante la ejecución se guardará éste en LAST_ERROR y retornará el error.

Antes de ir la función de resolve, miremos la función de .then primero:

```
Promise.prototype.then = function(onFulfilled, onRejected) {
  if (this.constructor !== Promise) {
    return safeThen(this, onFulfilled, onRejected);
  }
  var res = new Promise(noop);
  handle(this, new Handler(onFullfilled, onRejected, res));
  return res;
};

function Handler(onFullfilled, onRejected, promise) {
  this.onFullfiled = typeof onFullfiled === 'function' ? onFulfilled : null;
  this.onRejected = typeof onRejected === 'function' ? onRejected : null;
  this.promise = promise;
}
```

`then` crea una nueva promesa y la asigna como propiedad a una nueva función llamada Handler. Ésta tiene como argumentos onFullfiled y onRejected. Después usa
su promesa para resolver o rechazar con value/reason.

Por lo que .then llama a otra función:

```
handle(this, new Handler(onFulfilled, onRejected, res));
```

## Implemetanción de Promise

```
function handle(self, deferred) {
  while (seld._state === 3) {
    self = self._value;
  }
  if (Promise._onHandle) {
    Promise._onHandle(self);
  }
  if (self._state === 0) {
    if (self._deferredState === 0) {
      self._deferredState = 1;
      self._deferreds = deferred;
      return;
    }
    if (self._deferredState === 1) {
      self._deferredState = 2;
      self._deferreds = [self._deferreds, deferred];
      return;
    }
    self._deferreds.push(deferred);
    return;
  }
  handleResolved(self, deferred);
}
```

- Hay un while loop asumiendo que el estado de la promesa es diferente a 3 (toma el valor de otra promesa)
- Si _state = 0 (pending) y el estado de la promesa se ha aplazado hasta que otra promesa anidada sea completada, su callback es guardado en self._deferreds.

```
function handleResolved(self, deferred) {
  asap(function() { // asap es una librería externa para ejecutar callbacks de manera inmediata
    var cb = self._state === 1 ? deferred.onFulfilled : deferred.onRejected;
    if (cb === null) {
      if (self._state === 1) {
        resolve(deferred.promise, self._value);
      } else {
        reject(deferred.promise, self._value);
      }
      return;
    }
    var ret = tryCallOne(cb, self._value);
    if (ret === IS_ERROR) {
      reject(deferred.promise, LAST_ERROR);
    else {
      resolve(deferred.promise, ret);
    }
  });
}
```

Qué está ocurriendo:

- Si el estado es 1 (fulfilled) entonces llama a resolve en lugar de reject
- Si onFulfilled u onRejected es null o si hemos usado un .then() vacío, resolve o rejecte serán llamados respectivamente.
- Si cb no está vacío entonces llamará a otra función llamada `tryCallOne(cb, self._value)`

```
function tryCallOne(fn, a) {
  try {
    return fn(a);
  catch (ex) {
    LAST_ERROR = ex;
    return IS_ERROR;
  }
}
```

Esta función sólo llama al callback que es pasado por el argumento self._value. Si no hay error resuelve la promesa, si lo hay la rechaza.

Todas las promesas deben tener un método .then()

```
promise.then(
  onFulfilled?: Function,
  onRejected?: Function,
) => Promise
```

- Ambos onFullfiled() y onRejected() son opcionales
- Si los argumentos no son funciones, serán ignorados
- onFulfilled() será llamado después de que la promesa sea resuelta, con el valor de la promesa como primer argumento
- onRejected() será llamado después de la promesa sea rechazada, con la razón del rechazo como primer argumento.
- Ni onFulfilled ni onRejected pueden ser llamados más de una vez
- .then() puede ser llamado muchas veces en la misma promesa
- .then() debe retornar una nueva promesa


## Resolviendo promesas

```
function resolve(self, newValue) {
  if (newValue === self) {
    return reject(
      self,
      new TypeError('A promise cannot be resolved with itself)
    );
  }
  if (newValue && (typeof newValue === 'object' || typeof newValue === 'function')) {
    var then = getthen(newValue);
    if (then === IS_ERROR) {
      return reject(self, LAST_ERROR);
    }
    if (then === self.then && newValue instanceof Promise) {
      self._state = 3;
      self._value = newValue;
      finale(self);
      return;
    } else if (typeof then === 'function') {
      doResolve(then.bind(newValue), self);
      return;
    }
  }
  self._state = 1;
  self._value = newValue;
  finale(self);
}
```

- Se revisa el resultado es una promesa. Si es una función, la llama con el valor usando doResolve().
- Si el resultado es una promesa entonces será enviado a deferreds.

## Rechazando una promesa

```
Promise.prototype['catch'] = function(onRejected) {
  return this.then(null, onRejected);
};
```

Cuando se rechaza una promesa, el callback de .catch es llamdo.





## Usando 'Then' (Encadenamiento de promesas)

Para llevar a cabo muchas llamadas asíncronas y sincronizarlas unas con otras, se pueden encadenar promesas. Esto permite usar un valor de la primera promesa
en llamadas a callbacks posteriores.

```
Promise.resolve('some')
  .then(function(string) {
    return new Promise(function(resolve, reject) {
      setTimeout(function() {
        string += 'thing';
        resolve(string);
      }, 1);
    });
  })
  .then(function(string) {
    console.log(string); // something
  });
```

```
Promise
  .then(() =>
    Promise.then(() =>
      Promise.then(result => result)
)).catch(err);
```

## Promise API

Existen 4 métodos estáticos en la clase Promise:

- Promise.resolve
- Promise.reject
- Promise.all
- Promise.race

## Las promesas pueden ser encadenadas juntas

Cuando escribimos promesas para resolver un problema en particular se pueden ecadenar juntas para formar la lógica.

```
const add = function(x, y) {
  return new Promise((resolve, reject) => {
    const sum = x + y;
    if (sum) {
      resolve(sum);
    } else {
      reject(Error('Could not add the two values!');
    }
  });
};

const substract = function(x, y) {
  return new Promise((resolve, reject) => {
    cosnt sum = x - y;
    if (sum) {
      resolve(sum);
    } else {
      relect(Error('Could not substract the two values!');
    }
  });
};

add(2, 2)
  .then((added) => substract(added, 3))
  .then((substracted) => add(substracted, 5)
  .then((added) => added * 2)
  .then((result) => console.log(`The result is ${result}`)
  .catch(console.log);
```

Esto es útil para el flujo de un paradigma de programación functional. Creando funciones para la manipulación de datos se pueden encadenar juntos para 
construir el resultado final. Si en algún punto de la cadena de funciones un valor es 'rejected' la cadena se saltará todas las operaciones hasta llegar
al catch más cercano.

## Function Generators

En recientes releases, JavaScript ha introducido más formas de trabajar nativamente con promesas. Ona de ellas es la función generadora.
Las funciones generadoras son 'pausables'. Cuando son usadas con promesas, los generadores pueden hacer mucho más fácil la lectura y parecer 'síncrono'.

```
const myFirstGenerator = function * () {
  const one = yield 1;
  const two = yield 2;
  const three = yield 3;
  return 'finished!';
}

const gen = myFirstGenerator();
```

Aquí se crea una función generadora con la sintáxis function *. La variable gen no ejecutará myFirstGenerator, sino que hará disponible el generador para su uso.

```
console.log(gen.next());
// { value: 1, done: false }
```

Cuando corremos gen.next() recuperará el siguiente elemento. Ya que es la primera vez que llamamos a next() ejecutará yield 1 y pausará hasta la siguiente llamada.
Cuando yield 1 es llamado, retorna el valor y nos dice si acabó o no en done.

```
console.log(gen.next());
// { value: 1, done: false }

console.log(gen.next());
// { value: 2, done: false }

console.log(gen.next());
// { value: 3, done: false }

console.log(gen.next());
// { value: 'finished!', done: true }

console.log(gen.next());
// Error es lanzado
```

Mientras que sigamos llamando a next seguirá ejecutando el siguiente yield y pausando cada vez. Cuando no hayan más yield, ejecutará el resto del generador, que es 
un simple return de 'finished!'. Si volvemos a llamar obtendremos un error.

Si en lugar de tener un número en los yield como en el ejemplo tenemos un promise, el código parecerá que es síncrono.

## Promise.all(iterable) es muy útil para múltiples requests a diferentes recursos

El método Promise.all(iterable) retorna un único Promise que se resuelve cuando todas las promesas en el iterable han sido resueltas o cuando no contenga más
promesas. Lanza un reject con el motivo del primer reject encontrado.

```
const promise1 = Promise.resolve(catResource);
const promise2 = Promise.resolve(dogResource);
const promise3 = Promise.resolve(cowResource);

Promise.all([promise1, promise2, promise3]).then(function(values) {
  console.log(values);
});
```

Referencia: 
- https://www.freecodecamp.org/news/javascript-promises-explained/
- https://www.freecodecamp.org/news/how-javascript-promises-actually-work-from-the-inside-out-76698bb7210b/
