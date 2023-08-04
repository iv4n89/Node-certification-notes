## Events

Muchas de las APIs core de Node.js están cosntruídas alrededor de una arquitectura asíncrona basada en eventos donde cierto tipo de objetos (emitters) emiten eventos con nombre que causan que objetos de function (listeners) sean llamados.

Por ejemplo, un net.Servrer emite un evento cada vez que algo se conecta a él, un fs.ReadStream emite un evento cuando el fichero es abierto, etc.

Todos los objetos que emiten eventos son instancias de EventEmitter. Estos objetos exponen una función on() que permite a una o más funciones ser asociadas a la emisión de un evento. Típicamente, los nombres de eventos usan camel-case pero cualquier clave de propiedad de JS puede ser usada.

Cuando un objeto EventEmitter emite un evento, todas las funciones asociadas a ese evento específico son llamadas síncronamente. Cualquier valor retornado por la llamada de los listeners son ignorados o descartados.

En el ejemplo se muestra un EventEmitter simple con un único listener. El método on() es usado para registrar listeners, mientras que emit() es usado para llamar al evento.

```javascript
import { EventEmitter } from 'node:events';

class MyEmitter extends EventEmitter {}

const myEmitter = new MyEmitter();
myEmitter.on('event', () => {
  console.log('an event occurred');
});
myEmitter.emit('event');
```

## Pasar argumentos y this a los listeners

El método emit() permite un conjunto arbitrario de argumentos ser pasados a los listeners. Tener en mente que cuando un listener ordinario es llamado, la palabra this está intencionalmente seteado a la instancia de EventEmitter a la que el listener se ha asociado.

```javascript
import { EventEmitter } from 'node:events';

class MyEmitter extends EventEmitter {}

const myEmitter = new MyEmitter();
myEmitter.on('event', function(a, b) {
  console.log(a, b, this, this === myEmitter);
  // a b MyEmitter {
  // _events: [Object: null prototype] { event: [Function (anonymous)] },
  // _eventsCount: 1,
  // _maxListeners: undefined,
  // [Symbol(kCapture)]: false
  //} true
});
myEmitter.emit('event', 'a', 'b');
```

## Asíncrono vs síncrono

EventEmitter llama a todos los listeners síncronamente en el orden que hayan sido registrados. Esto se asegura que la secuencialidad de eventos y ayuda a evitar race conditions y errores de lógica. Cuando sea apropiado, los listeners pueden cambiar a un modo asíncrono de operación usando el setInmmediate() o process.nextTick():

```javascript
import { EventEmitter } from 'node:events';
class MyEmitter extends EventEmitter {}

cosnt myEmitter = new MyEmitter();
myEmitter.on('event', (a, b) => {
  setInmediate(() => {
    console.log('this happends async');
  });
});
myEmitter.emit('event', 'a', 'b');
```

## Manejar eventos sólo una vez

Cuando un listener es registrado usando el método on(), este listener es invocado cada vez que el evento es emitido

```javascript
import { EventEmitter } from 'node:events';
class MyEmitter extends EventEmitter {}
const myEmitter = new MyEmitter();
let m = 0;
myEmitter('event', () => console.log(++m); );
myEmitter.emit('event'); // 1
myEmitter.emit('event'); // 2
```

Usando el método once() es posible registrar un listener que es llamado al menos una vez para un evento en particular. Una vez el evento es emitido, el listener es eliminado y entonces llamado.

```javascript
import { EventEmitter } from 'node:events';
class MyEmitter extends EventEmitter {}
const myEmitter = new MyEmitter();
let m = 0;
myEmitter.once('event', () => console.log(++m); );
myEmitter.emit('event'); // 1
myEmitter.emit('event'); // Ignorado
```

## Eventos de error

Cuando ocurre un error dentro de un EventEmitter, la acción típica para un evento 'error' es ser emitido. Esto es tratado como un caso especial dentro de Node.js.

Si un EventEmitter no tiene al menos un listener registrado para el evento 'error', y un evento error es emitido, el error es lanzado, un stack trace es impreso y el proceso de Node.js es cortado.

```javascript
import { EventEmitter } from 'node:events';
class MyEmitter extends EventEmitter {}
const myEmitter = new MyEmitter();
myEmitter.emit('error', new Error('whoops!'));
// Lanza el error y crashea
```

Para resguardarse de los crashes de node.js el módulo domain puede ser usado (deprecado).

Como buena práctica, se debe agregar un listener para error

```javascript
import { EventEmitter } from 'node:events';
class MyEmitter extends EventEmitter {}
const myEmitter = new MyEmitter();
myEmitter.on('error', err => console.error('whoops! there has an error'); );
myEmitter.emit('error', new Error('whoops!');
// Imprime whoops! there has an error
```

Es posible monitorizar eventos de error sin consumir el error emitido usando el symbol events.errorMonitor

```javascript
import { EventEmitter, errorMonitor } from 'node:events';

const myEmitter = new EventEmitter();
myEmitter.on(errorMonitor, err => MyMonitoringTool.log(err); );
myEmitter.emit('error', new Error());
// Aún hace throw del error
```

## Capturar rejections de promesas

Usar async functions con los event handlers es problemático, ya que puede llevar a un unhandled rejection en caso que lanzar una excepción

```javascript
import { EventEmitter } from 'event:node';
const ee = new EventEmitter();
ee.on('something', async (value) => throw new Error('kaboom'); );
```

La opción de captureRejections en el constructor de EventEmitter o el global cambia este comportamiento, instalando un .then(undefined, handler) en la promesa. Este handler enruta las excepciones asíncronamente al método Symbol.for('nodejs.rejection') si hay uno, o al event handler 'error' si hay uno.

```javascript
import { EventEmitter } from 'node:events';
const ee1 = new EventEmitter({ captureRejections: true });
ee1.on('somethin', async (value) => throw new Error('kaboom'); );
ee1.on('error', console.log);
const ee2 = new EventEmitter({ captureRejections: true });
ee2.('something', async (value) => throw new Error('kaboom'); );
ee2[Symbol.for('nodejs.rejection')] = console.log;
```

Setear events.captureRejections a true cambiará el valor por defecto de todos los EventEmitters

```javascript
import { EventEmitter } from 'node:events';

EventEmitter.captureRejections = true;
const ee1 = new EventEmitter();
ee1.on('something', async (value) => throw new Error('kaboom'); );

ee1.on('error', console.log);
```

Los eventos error que son generados por el comportamiento captureRejections no tienen un catch handler para evitar loops infinitos, la recomendación es NO usar async functions como handlers del evento error.

## Class: EventEmitter

La clase EventEmitter es definida y expuesta por el módulo 'node:events'

```javascript
import { EventEmitter } from 'node:events';
```

Todos los EventEmitters emiten el evento 'newListener' cuando un nuevo listener ha sido agregado y 'removeListener' cuando un listener existente ha sido eliminado.

Soporta las opciones:

- captureRejections <boolean> Permite capturar autmáticamente toda rejection de promesa. Default: false.

### Event: 'newListener'

- eventName <string> | <symbol> El nombre del evento al que va a escucharse
- listener <Function> El event handler

 La instancia de EventEmitter emitirá su propio newListener event antes de que un listener sea agregado a su array interno de listeners.

 Los listeners registrados para 'newListeners' pasados son el nombre del evento y una referencia al listener siendo añadido. 

 El hecho de que el evento sea lanzado antes de añadir el listener tiene un sutil pero importante efecto colateral: cualquier listener adicional registrado al mismo name dentro del callback 'newListener' son insertados antes del listener que esté en progreso de ser agregado.

 ```javascript
import { EventEmitter } from 'node:events';
class MyEmitter extends EventEmitter {}

const myEmitter = new MyEmitter();
// Sólo hacer esto una vez para no entrar en loop infinito
myEmitter.once('newListener', (event, listener) => {
  if (event === 'event') {
    // Insertar un nuevo listener en el frente
    myEmitter.on('event', () => {
      console.log('B');
    });
  ;
});
myEmitter.on('event', () => {
  console.log('A');
});
myEmitter.emit('event');
// Imprime
// B
// A
```

### Event: 'removeListener'

- eventName <string> | <symbol> El nombre del evento
- listener <Function> La función del event handler

Emitido después de que un listener sea removido.

### emitter.addListener(eventName, listener)

- eventName <string> | <symbol>
- listener <Function>

Alias para emitter.on(eventName, listener)

### emitter.emit(eventName[, ...args])

- eventName <string> | <symbol>
- ...args <any>
- return <boolean>

Síncronamente llama a cada listener registrado para el evento llamado eventName, en el orden en el que fueron registrados, pasado los argumentos a cada uno.

Retorna true si el evento tiene listeners, false si no.

```javascript
import { EventEmitter } from 'node:events';
const myEmitter = new EventEmitter();

// Primer listener
myEmitter.on('event', function firstListener() {
  console.log('Helloooo! first listener');
});

// Segundo listener
myEmitter.on('event', function secondListener(arg1, arg2) {
  console.log(`event with parameters ${arg1}, ${arg2} en el segundo listener`);
});

// Tercer listener
myEmitter.on('event', function thirdListener(...args) {
  const paramteres = args.join(', ');
  console.log(`event with parameters ${parameters ${parameters} in third listener`);
});

console.log(myEmitter.listeners('event'));

myEmitter.emit('event', 1, 2, 3, 4, 5);

// Imprime:
// [
//   [Function: firstListener],
//   [Function: sencodListener],
//   [Function: thirdListener]
// ]
// Hellooo! first listener
// event with paramteres 1, 2 in second listener
// event with parameters 1, 2, 3, 4, 5 in third listener
```

### emitter.eventNames()

- return <Array>

Retorna un array listando los eventos para los que el emitter tiene listeners registrados. Los valores en el array son string o Symbos.

```javascript
import { EventEmitter } from 'node:events';

const myEE = new EventEmitter();
myEE.on('foo', () => {});
myEE.on('bar', () => {});

const sym = Symbol('symbol');
myEE.on(sym, () => {});

console.log(myEE.eventNames());
// Imprime ['foo', 'bar', Symbol(symbol)]
```

### emitter.getMaxListeners()

- return <integer>

Retorna el valor actual de listeners máximos pare el EventEmitter que es seteado por emitter.setMaxListeners(n) o defaults a event.defaultMaxListeners.

### emitter.listenerCount(eventName[, listener])

- eventName <string> | <symbol> El nombre del evento para el cual el listener está siendo registrado
- listener <Function> La función del event handler
- return <integer>

Retorna el número de listeners escuchando el evento llamado eventName. Si listener es provisto, retorna cuántas veces el listener es encontrado en la lista de listeners del evento

### emitter.listeners(eventName)

- eventName <string> | <symbol>
- return <Function[]>

Retorna una copia del array de listeners para el evento llamado eventName

```javascript
server.on('connection', (stream) => {
  console.log('someone connected!');
});
console.log(util.inspect(server.listeners('connection')));
// Imprime [ [Function] ]
```

### emitter.off(eventName, listener)

- eventName <string> | <symbol>
- listener <Function>
- return <EventEmitter>

Alias para emitter.removeListener()

### emitter.on(eventName, listener)

- eventName <string> | <symbol> El nombre del evento
- listener <Function> El callback
- retorna <EventListener>

Añade el listener al final del array de listeners para el evento llamado eventName. No se realizan checks para ver si el listener ya ha sido agregado. Múltiples llamadas con la misma combinación de eventName y listener resultará en el listener siendo agregado, y llamado múltiples veces:

```javascript
server.on('connection', (stream) => {
  console.log('someone connected!');
});
```

Retorna una referencia al EventEmitter, por lo que llamadas encadenadas pueden ser usadas.

Por defecto, los event listeners son invocados en el orden que son añadidos. El método emitter.prependListener() puede ser usado para alternativamente añadir el event listener al inicio del array.

```javascript
import { EventEmitter } from 'node:events';
const myEE = new EventEmitter();
myEE.on('foo', () => console.log('a'));
myEE.prependListener('foo', () => console.log('b'));
myEE.emit('foo');
// Imprime
// b
// a
```

### emitter.once(eventName, listener)

- eventName <string> | <symbol> El nombre del evento
- listener <Function> El callback function
- return <EventEmitter>

Agrega un listener de un solo uso para el evento llamado eventName. La siguiente vez que eventName sea llamado, este listener es removido y entonces invocado.

```javascript
server.once('connection', (stream) => {
  console.log('Ah, we have our first user!);
});
```

Retorna una referencia al EventEmitter, por lo que se pueden encadenar llamadas.

Por defecto, los event listeners son invocados en el orden que son añadidos. El métodos emitter.prependOnceListener() puede ser usado como una llamada alternativa para añadir el event listener al inicio del array.

```javascript
import { EventEmitter } from 'node:events';
const myEE = new EventEmitter();
myEE.once('foo', () => console.log('a'));
myEE.prependOnceListener('foo', () => console.log('b'));
myEE.emit('foo');
// Imprime
// b
// a
```

### emitter.prependListener(eventName, listener)

- eventName <string> | <symbol> El nombre del evento
- listener <Function> El callback
- return <EventEmitter>

Añade el listener al inicio del array de listeners para el evento llamado eventName. No se revisa si ya está añadido. Se añadirá varias veces si se llama varias veces.

```javascript
server.prependListener('connection', (stream) => {
  console.log('someone connected!');
});
```

Retorna una referencia al EventEmitter, por lo que se pueden encadenar llamadas

### emitter.prependOnceListener(eventName, listener)

- eventName <string> | <symbol>
- listener <Function>
- return <EventEmitter>

Igual que .prependListener, pero de un único uso. Se borra y luego se llama al listener al saltar

### emitter.removeAllListeners([eventName])

- eventName <string> | <symbol>
- return <EventEmitter>

Remueve todos los listeneres específicos para eventName

Es una mala práctica eliminar listeners añadidos en alguna otra parte del código, particularmente cuando la instancia de EventEmitter fue creado por cualquier otro componente o módulo

Retorna una referencia al EventEmitter, por lo que se pueden encadenar llamadas

### emitter.removeListener(eventName, listener)

- eventName <string> | <symbol>
- listener <Function>
- return <EventEmitter>

Remueve el listener específico del array de listeners para el evento eventName

```javascript
const callback = (stream) => {  
  console.log('someone connected!');
});
server.on('connection', callback);
//...
server.removeListener('connection', callback);
```

removeListener() eliminará, al menos, una instancia del listener desde el array de listeners. Si algún listener ha sido agregado múltiples veces, removeListener() debe ser llamado múltiples vceces para eliminar cada instancia

Una vez el evento es emitido, todos los listeners asociados a él al mismo tiempo de la emisión son llamados en orden. Esto implica que cualquier removeListener() o removeAllListeners() es llamado después de emitir y antes de que el último listener acabe su ejecución no serán removidos mientras que emit() esté en progreso. Subsecuentes eventos se comportarán de manera esperada.

```javascript
import { EventEmitter } from 'node:events';
class MyEmitter extends EventEmitter {}
const myEmitter = new MyEmitter();

const callbackA = () => {
  console.log('A');
  myEmitter.removeListener('event', callbackB);
};

const callbackB = () => {
  console.log('B');
};

myEmitter.on('event', callbackA);

myEmitter.on('event', callbackB);

// callbackA remueve el listener callbackB pero aún así será llamado.
// Internal listeners array al tiempo de emit [callbackA, callbackB]
myEmitter.emit('event');
// Imprime
// A
// B

// callbackB es ahora removido
// Internal listener array [callbackA]
myEmitter.emit('emit');
// Imprime
// A
```

Ya que los listeners son manejados usando un array interno, llamar esto cambiará la posición de los índices de cualquier listener registrado después del listener siendo removido. Esto no impactará  al orden en el que los listeners son llamados, pero significa que cualquier copia del listener array como el retornado por emitter.listeners() necesitará ser recreado.

Cuando una función única ha sido agregada como un handler múltiples veces para un único evento, removeListener() eliminará el añadido más reciente. En el ejemplo once('ping') es removido

```javascript
import { EventEmitter } from 'node:events';
const ee = new EventEmitter();

function pong() {
  console.log('pong');
}

ee.on('ping', pong);
ee.once('ping', pong);
ee.removeListener('ping', pong);

ee.emit('ping');
ee.emit('ping');
```

Retorna una referencia el EventEmitter, por lo que se pueden encadenar llamadas.

### emitter.setMaxListeners(n)

- n <integer>
- return <EventEmitter>

Por defecto los EventEmitters imprimen un warning si más de 10 listeners son agregados para un evento en particular. Esto es útil para ayudar a encontrar memory leaks. El método emitter.setMaxListeners() permite modificar el máximo para esta instancia específica de EventEmitter. El valor puede ser seteado a Infinity (o 0) para indicar un número ilimitado de listeners.

Retorna una referencia al EventEmitter, por lo que se pueden encadenar llamadas.

### emitter.rawListeners(eventName)

- eventName <string> | <symbol>
- return <Function[]>

Retorna una copia del array de listeners para el evento llamado eventName, incluyendo cualquier wrapper (como aquellos creados por .once())

```javascript
import { EventEmitter } from 'node:events';
const emitter = new EventEmitter();
emitter.once('log', () => console.log('log once'));

// Retorna un nuevo array con una función 'onceWrapper' que tiene una propiedad 'listener' que contiene el listener original abajo
const listeners = emitter.rawListeners('log');
cosnt logFnWrapper - listeners[0];

// Log 'log once' en la consola y no elimina desvincula el once del evento
logFnWrapper.listener();

// Log 'log once' en la consola y elimina el listener
logFnWrapper();

emitter.on('log', () => console.log('log persistently'));
// Retornará un nuevo array con un single function vinculado por .on()
const newListeners = emitter.rawListeners('log');

// Logea 'log persistently' dos veces
newListeners[0]();
emitter.emit('log');
```

### emitter[Symbol.for('nodejs.rejection')](err, eventName[, ...args])

- err Error
- eventName <string> | <symbol>
- ...args <any>

El método Symbol.for('nodejs.rejection') es llamado en caso que una promesa sea rechazada cuando se emite un evento y captureRejection está habilitado en el emitter. Es posible usar events.captureRejectionSymbol en lugar de Symbol.for('nodejs.rejection');

```javascript
import { EventEmitter, captureRejectionSymbol } from 'node:events';

class MyClass extends EventEmitter {
  constructor() {
    super({ captureRejections: true });
  }

  [captureRejectionSymbol](err, event, ...args) {
    console.log('rejection happened for', event, 'with', err, ...args);
  }

  destroy(err) {
    // Tear the resource down here
  }
}
```

### events.defaultMaxListeners

Por defecto, un máximo de 10 listeners pueden ser registrados para un evento. Este límite puede ser cambiado por cada instancia de EventEmitter usando emitter.setMaxListeners(n). Para cambiar el valor por defecto de todos los EventEmitters, se puede usar events.defaultMaxListeners. Si el valor no es positivo lanza un RangeError.

Precaución cuando se setea defaultMaxListeners ya que cambiarlo afecta a todos los EventEmitters, incluidos aquellos que se crearon antes de hacer el cambio. Sin embargo, llamar a emitter.setMaxListeners(n) aún tiene precedencia sobre events.defaultMaxListeners.

No es un limitador. La instancia de eventEmitter permitirá más listeners a ser agregados pero el aoutput trace warning a stderr dobre possible EventEmitter memory leak puede aparecer. Para un único EventEmitter, los métodos emitter.getMaxListeners() y emitter.setMaxListeners() pueden usarse termoralmente.

```javascript
import { EventEmitter } from 'node:events';
const emitter = new EventEmitter();
emitter.setMaxListeners(emitter.getMaxListeners() + 1);
emitter.once('event', () => {
  // hacer algo
  emitter.setMaxListeners(Math.max(emitter.getMaxListeners() -1, 0));
});
```

El command flag --trace-warnings puede ser usado para mostrar el stack trace para dichos warnings.

El warning emitido puede ser inspeccionado con process.on('warning') y tendrá las propiedades adicionales emitter, type y count, refiriéndose a la instancia de eventEmitter, el nombre del evento y el número de listeners, respectivamente. La nombre de la propiedad es seteada a 'MaxListenersExceededWarning'.

### events.errorMonitor

Este symbol debe ser usado para instalar un listener para sólo monitorizar eventos error. Los listeners instalados usando este symbol son llamados antes de los listeners regulares de error.

Instalar un listener usando este symbol no cambia el comportamiento una vez el evento error es emitido. Aún así, el proceso crasheará si no hay listeners para error instalados.

### events.getEventListeners(emitterOrTarget, eventName)

- emitterOrTarget <EventEmitter> | <EventTarget>
- eventName <string> | <symbol>
- return <Function[]>

Retorna una copia del array de listeners para el evento llamado eventName.

Para EventEmitter se comporta exactamente igual que llamando a .listeners en el emitter.
Para EventTarget es la única manera de obtener los event listeners para el event target. Es útil para debug y diagnóstico.

```javascript
import { getEventListeners, EventEmitter } from 'node:events';

{
  const ee = new EventEmitter();
  const listener = () => conosole.log('Events are fun');
  ee.on('foo', listener);
  console.log(getEventListeners(ee, 'foo')); // [ [Function: listener] ]
}
{
  const et = new EventTarget();
  const listener = () => console.log('Events are fun');
  et.addEventListener('foo', listener);
  console.log(getEventListeners(et, 'foo')); // [ [Function: listener] ]
}
```

### events.getMaxListeners(emitterOrTarget)

- emitterOrTarget <EventEmitter> | <EventTarget>
- return <number>

Return el máximo número de listeners seteado.

En EventEmitters tiene el mismo comportamiento que .getMaxListeners

En EventTarget es la única forma de obtener el máximo de listeners para el event target.

```javascript
import { getMaxListeners, setMaxListeners, EventEmitter } from 'node:events';

{
  const ee = new EventEmitter();
  console.log(getMaxListeners(ee)); // 10
  setMaxListeners(11, ee);
  console.log(getMaxListeners(ee)); // 11
}
{
  const et = new EventTarget();
  console.log(getMaxListeners(et)); // 10
  setMaxListeners(11, et);
  console.log(getMaxListeners(et)); // 11
}
```

### events.once(emitter, name[, options])

- emitter <EventEmitter>
- name <string>
- options <Object>
  - signal <AbortSignal> Puede ser usado para cancelar el evento que espera
- return <Promise>

Crea una promesa que es completada cuando el EventEmitter emite el evento dado o es rechazado si el EventEmitter emite 'error' mientras espera. La promesa será resuelta con un array de todos los argumentos emitidos con el evento.

Este método es intencionalmente genérico y funciona la interfaz web EventTarget, que no tiene tiene un evento especial 'error' ni un listener para 'error'.

```javascript
import { once, EventEmitter } from 'node:events';
import process from 'node:process';

const ee = new EventEmitter();

process.nextTick(() => {
  ee.emit('myevent', 42);
});

const [value] = await once(ee, 'myevent');
console.log(value);

const err = new Error('kaboom');
process.nextTick(() => {
  ee.emit('error', err);
});

try {
  await once(ee, 'myevent');
} catch (err) {
  console.log('error happened', err);
}
```

El handling especial del evento error es sólo usado cuando events.once() es usado para esperar por otro evento. Si events.once() es usado para esperar por error, entonces éste es tratado como cualquier otro tipo de evento sin especial handling.

```
import { EventEmitter, once } from 'node:events';

const ee = new EventEmitter();

once(ee, 'error')
  .then(([err]) => console.log('ok', err.message)
  .catch((err) => console.error('error', err.message));

ee.emit('error', new Error('boom'));

// Imprime: Ok boom
```

Un <AbortSignal> puede ser usado para cancelar la espera por un evento

```
import { EventEmitter, once } from 'node:events';

const ee = new EventEmitter();
const ac = new AbortController();

async function foo(emitter, event, signal) {
  try {
    await once(emitter, event, { signal });
    console.log('event emitted);
  } catch (err) {
    if (error.name === 'AbortError') {
      console.error('Waiting for the event was canceled!');
    } else {
      console.error('There was an error', error.message);
    }
  }
}

foo(ee, 'foo', ac.signal);
ac.abort(); // Aborta la espera del evento
ee.emit('foo'); // Imprime esperando ...
```

### Esperar múltiples eventos emitidos en process.nextTick()

Hay un caso especial cuando usamos la función events.once() para esperar múltiples eventos emitidos en el mismo tick de process.nextTick(), o cuando hay múltiples eventos para ser emitidos síncronamente. Especialmente porque la cola de nextTick() es drenada antes de la cola de micro tasks de promise, y porque EventEmitter emite todos los eventos síncronamente, es posible para events.once() perderser un evento

```javascript
import { EventEmitter, once } from 'node:events';
import process from 'node:process';

const myEE = new EventEmitter();

async function foo() {
  await once(myEE, 'bar');
  console.log('bar');

  // Esta promesa nunva va a resolver porque el evento foo será emitido antes de que la promesa sea creada
  await once(myEE, 'foo');
  console.log('foo');
}

process.nextTick(() => {
  myEE.emit('bar');
  myEE.emit('foo');
});

foo().then(() => console.log('done'));
```

Para hacer catch de ambos eventos, crear cada uno de las promesas antes de esperar por ellas, entonces es posible usar Promise.all(), Promise.race() o Promise.allSettled():

```javascript
imprt { EventEmitter, once } from 'node:events';
import process from 'node:process';

const myEE = new EventEmitter();

async function foo() {
  await Promise.all([once(myEE, 'bar'), once(myEE, 'foo')]);
  console.log('foo', 'bar');
}

process.nextTick(() => {
  myEE.emit('bar');
  myEE.emit('foo');
});

foo().then(() => console.log('done'));
```

### events.captureRejections
