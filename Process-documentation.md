## Process

El objeto process ofrece información sobre, y sobre el control de, el proceso actual de Node.js.

```javascript
import process from 'node:process';
```

## Process events

El objeto `process` es una instancia de `EventEmitter`.

### Event: 'beforExit'

Es emitido cuando Node.js vacía su event loop y no tiene más trabajos encolados. Normalmente el proceso de Node.js sale cuando no tiene más trabajo pendiente, pero un listener en el evento `beforeExit` puede hacer llamadas asíncronas y hacer que el proceso de Node.js continúe.

El callback del listener recibe el process.exitCode como argumento.

No se llamará si hay un cierre explícito, como una llamada a process.exit() o un uncaught exception.

No se debe usar como alternativa a `exit` a menos que necesitemos encolar trabajo adicional

```javascript
import process from 'node:process';

process.on('beforeExit', code => {
  console.log('Process beforeExit event with code: ', code);
});

process.on('exit', code => {
  console.log('Process exit event with code: ', code);
});

console.log('This message is displayed first');

// This message is displayed first
// Process beforeExit event with code: 0
// Process exit event with code: 0
```


### Event: 'disconnect'

Si el proceso de Node.js es generado en un canal IPC (child process, cluster), este evento será emitido cuando el canal IPC es cerrado.


### Event: 'exit'

- code `integer`


El evento es emitido cuando el proceso de Node.js está a punto de cerrarse como resultado de:
- Una llamada explícita a process.exit().
- El event loop de node.js ya no tiene trabajo adicional


No hay manera de prevenir la del event loop en este punto, y una vez que todos los listeners de `exit` han finalizado de correr el proceso de Node.js será matado.

El callback recibe process.exitCode o el exitCode pasado al método process.exit()

```javascript
import process from 'node:process';

process.on('exit', code => {
  console.log('About to exit with code: ', code);
});
```

Los listeners debe sólo realizar operaciones síncronas. El proceso de Node.js saldrá inmediatamente después de llamar a los listneres de exit cuasando que cualquier trabajo pendiente en el event loop sea abandonado.
En el ejemplo, el setTimeout nunca ocurrirá:

```javascript
import process from 'node:process';

process.on('exit', code => {
  setTimeout(() => {
    console.log('This will not run');
  }, 0);
});
```


### Event: 'message'

- message `Object`|`boolean`|`number`|`string`|`null` Un objeto JSON parseado o un valor primitivo serializable.
- sendHandle `net.Server`|`net.Socket` Un net.Server o net.Socket, o undefined.


Si el proceso de Node.js está en un canal IPC (child process, cluster), el evento `message` será emitido siempre que un mensaje sea enviado por el proceso padre usando childprocess.send().

Si la opción `serialization` fue seteada a `advanced` cuando se generó el proceso, el argumento `message` puede contener datos que un JSON no sea capaz de representar.


### Event: 'multipleResolves'
__Deprecated__


