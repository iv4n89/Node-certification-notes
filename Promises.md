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
- Fultilled: Operación completa con éxito.
- Rejected: Operación fallida.

El objeto de promesa funciona como un proxy para un valor no necesariamente conocido cuando la promesa es creada. Permite asociar handlers con un valor de 
operación realizada o razón de fallo.

Esto permite a métodos asíncronos retornar valores como si fuesen métodos síncronos: En lugar de retornar el valor inmediatamente, el método asíncrono retorna
una promesa para devolver el valor en algún punto en el futuro.

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

Referencia: https://www.freecodecamp.org/news/javascript-promises-explained/
