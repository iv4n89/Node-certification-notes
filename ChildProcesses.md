## Child processes

Con un único hilo, el rendimiento no-bloqueante en Node.js funciona bien para un único proceso. Pero eventualmente, un proceso en una CPU no será suficiente para
manejar el trabajo incremental de la aplicación.

El hecho de que Node.js corra en un único hilo no significa que no podamos tomar ventaja de múltiple procesos y, por supuesto, múltiples máquinas.

Usar múltiples procesos es la mejor manera de escalar una aplicación de Node. Node.js está diseñado para construir aplicaciones distribuidas con muchos nodos.

## El módulo Child Processes

Podemos crear fácilmente un proceso hijo con el módulo de Node child_process, y hacer que se comuniquen entre ellos con un sistema de mensajes.

El módulo child_process permite acceder a funcionalidades del SO corriendo cualquier sistema de comando dentro de un proceso hijo.

Podemos controlar el input stream del proceso hijo y escuchar su salida. Podemos también controlar los argumentos que serán pasados al subyacente SO command,
y podemos hacer lo que queramos con este output del comando. Podemos, por ejemplo, hacer pipe el output de uno de los comandos como el input de otro (como en linux)
ya que todos los inputs y outputs de estos comandos pueden ser prsentados a nosotros usando Node.js streams.

Hay 4 formas diferentes de crear procesos hijos en Node.js: spawn(), fork(), exec() y execFile().

## Child processes con spawn

La función spawn lanza un comando en un nuevo proceso y podemos usarlo para pasarle cualquier argumento al comando. Por ejemplo, se crea un nuevo proceso hijo que ejecuta
el comando pwd:

```
const  { spawn } = require('child_process');

const child = spawn('pwd');
```

El resultado de ejecutar la función spawn (el objeto child) es una instancia de ChildProcess, que implementa el API EventEmitter. Esto significa que podemos registrar
handlers para eventos en estos child objects directamente. Por ejemplo, podemos hacer algo cuando el proceso hijo sale registrando un handler para el evento exit:

```
child.on('exit', function (code, signal) {
  console.log('child process exited with ' + `code ${code} and signal ${signal}`);
});
```

El handler de arriba nos da el exit code del proceso hijo y el signal, si existe, que fue usado para terminar el proceso hijo. Esta variable signal es null
cuando el child process sale normalmente.

Los otros eventos que podemos registrar handlers en el ChildProcess son: disconnect, error, close y message.

- El evento disconnect es emitido cuando el proceso padre manualmente llama a child.disconnect
- El evento error es emitido si el proceso no pudo se generado o matado.
- El evento close es emitido cuando el stream stdio de un child process es cerrado.
- El evento message es emitido cuando el child process usa la función process.send() para enviar mensajes. Así es como los procesos padre/hijo pueden comunicarse entre ellos

Cada child process también obtiene 3 streams stdio standards, que podemos acceder usando child.stdin, child.stdout y child.stderr.

Cuando estos streams son cerrados, el child process que los estaba usando emitirá el evento close. Éste es diferente al evento exit ya que múltiples child process pueden 
compartir el mismo stdio stream por lo que la salida de un child process no significa que un stream haya sido cerrado.

Ya que todos los streams son event emitter, podemos escuchar diferentes eventos en estos stdio streams que están unidos a diferentes child process. Al contrario que un
proceso normal, los streams stdout y stderr son readable en un proceso hijo, mientras que stdin es el stream writable. A la inversa que un proceso normal. Los eventos
que podemos usar en ellos son los stándard. Más importante, en los readable streams podemos escuchar el evento data

```
child.stdout.on('data', (data) => {
  console.log(`child stdout: \n${data}`);
});

child.stderr.on('data', (data) => {
  console.log(`child stderr: \n${data}`);
});
```

Los dos handlers anteriores lanzarán logs hacia los stdout y stderr del proceso main. Cuando ejecutamos la función spawn, el output de pwd se imprime y el child process
sale con código 0, que significa no errors.

Podemos pasar argumentos al comando que ejecutado por la función spawn usando el segundo el segundo argumento, que es un array de todos los argumentos pasados al comando.
Por ejemplo, para ejecutar el find command en el directorio actual con un -type f (para listar files solo): 

```
const child = spawn('find', ['-', '-type', 'f'];
```

Si un error ocurre durante la ejecución del comando, el evento data de child.stderr y su handler son llamados, y el event handler de exit reportará un exit code de 1.

Un child process stdin es un writable. Podemos usarlo para enviar algo de input a un command. 

```
const { spawn } = require('child_process');

const child = spawn('wc');

process.stdin.pipe(child.stdin);

child.stdout.on('data', (data) => {
  console.log(`child stdout:\n${data}`);
});
```

Este child process invoca el comando wc, que cuenta líneas, palabras y caracteres en linux. Le hacemos entonces pipe con el stdin del main process (que es un readable stream) junto con el stdin (que es un writable). El resultado de esta combinación es que obtenemos un input mode standard donde podemos escribir algo y cuando pulsemos ctrl+d lo que escribamos va como input a wc.

![1_s9dQY9GdgkkIf9zC1BL6Bg](https://github.com/iv4n89/Node-certification-notes/assets/69988988/f662738e-575e-4ff7-82c0-ad80a85f18e1)

```
const { spawn } = require('child_process');

const find = spawn('find', ['.', '-type', 'f']);
const wc = spawn('wc', ['-l']);

find.stdout.pipe(wc.stdin);

wc.stdout.on('data', (data) => {
  console.log(`Number of files ${data}`);
});
```

## Sintáxis de las funciones shell y exec

Por defecto, la función spawn no crea un shell para ejecutar el comando que le pasamos. Esto lo hace ligeramente más eficiente que la función exec, que sí crea un shell. La función exec tiene otra gran diferencia, guarda su output en un buffer y pasa todo su valor a un callback (en lugar de streams)

```
const { exec } = require('child_process);

exec('find . -type f | wc -l', (err, stdout, stderr) => {
  if (err) {
    console.error(`exec error; ${err}`);
    return;
  }

  console.log(`Number of files ${stdout}`);
});
```

Ya que la función exec usa un shell para ejecutar el comando, podemos usar la sintaxis de shell directamente y hacer uso del pipe de shell.

La función exec es una buena elección si necesitamos usar la sintaxis de shell y si el tamaño de los datos que esperamos desde el comando es pequeño

La función spawn es mucha mejor opción cuando el tamaño de los datos esperados desde el comando es grande, ya que estos datos serán stremeados con el standard IO Object.

Podemos hacer a los child process generados herede al standard IO objects de sus padres si queremos. Y más importante, podemos hacer que spawn use la sintaxis de shell.

```
const child = spawn('find . -type f | wc -l, {
  stdio: 'inherit',
  shell: true,
});
```

Debido a la opción stdio: 'inherit' está activa, cuando ejecutemos el código el child process hereda los stdin, stdout y stderr del proceso main. Esto hace que los event handlers del child process sean llamados en el process.stdout del main.

Debido a que la opción shell está en true, podremos usar la sintaxis de shell.

Hay algunas otras buenas opciones que podemos usar en el último argumento de child_process. Por ejemplo, usar el cdw option para cambiar el working directory del script.

```
const child = spawn('find . -type f | wc -l, {
  stdio: 'inherit',
  shell: true,
  cwd: '/Users/samer/Downloads
});
```

Otra es la env option para especificar variables de entorno que serán visibles en el child process. Por defecto es process.env

```
const child = spawn('echo $ANSWER', {
  stdio: 'inherit',
  shell: true,
  env: { ANSWER: 42 },
});
```

Otra opción importante aquí es detached, que hace que el child process corra de manera independiente al proceso padre.

Asumiendo que tenemos un fichero timer.js que mantiene el event loop ocupado:

```
setTimeout(() => {
  // Mantener el event loop ocupado
}, 20000);
```

Podemos ejecutar por detrás el child process con la opción detached:

```
const { spawn } = require('child_process');

const child = spawn('node', ['timer.js'], {
  detached: true,
  stdio: 'ignore'
});

child.unref();
```

Si la función unref es llamada en un detached process, el padre puede salir independientemente del hijo. Esto puede ser útil si el hijo será ejecutado por un largo tiempo, pero para mantenerlo corriendo el stdio también debe ser independiente. 

![1_WhvMs8zv-WS6v7nDXmDUzw](https://github.com/iv4n89/Node-certification-notes/assets/69988988/3be00885-c18f-4543-8272-30f6351ce2a5)

## La función execFile

Ejecuta un fichero usando un shell. Se comporta igual que la función exec, pero no hace uso de un shell, lo que lo hace más eficiente. En Windows algunos ficheros no pueden ser ejecutados por sí mismos, como .bat o .cmd. Éstos no pueden ser ejecutados por execFile.

## La función *Sync

Las funciones hasta ahora vistas también tienen versiones síncronas bloqueantes

```
const {
  spawnSync,
  execSync,
  execFileSync
} = require('child_process');
```

## La función fork

fork es una variación de spawn para generar procesos de nodo. Su diferencia es que establece un canal de comunicación hacia el child process, pudiéndose usar send para intercambiar mensajes. Esto ocurre a través del módulo EventEmitter. 

```
## parent.js

const { fork } = require('child_process');

const forked = fork('child.js');

forked.on('message', (msg) => {
  console.log('Message from child', msg);
});

forked.send({ hello: 'world' });


## child.js


process.on('message', (msg) => {
  console.log('Message from parent: ', msg);
});

let counter = 0;

setInterval(() => {
  process.send({ counter: counter++ });
}, 1000);
```

fork ejecuta con el comando node y nos deja escuchar lo que ocurre.

![1_GOIOTAZTcn40qZ3JwgsrNA](https://github.com/iv4n89/Node-certification-notes/assets/69988988/a5cb6b7c-bceb-4a2d-96f4-11e4f68f01b8)

```
const http = require('http');

const longComputation = () => {
  let sum = 0;
  for (let i = 0; i < 1e9; i++) {
    sum += i;
  };
  return sum;
};

const server = http.createServer();

server.on('request', (req, res) => {
  if (req.url === '/compute') {
    const sum = longConputation();
    return res.end(`Sum is ${sum}`);
  } else {
    res.end('Ok');
  }
});

server.listen(3000);
```

Referencias: https://www.freecodecamp.org/news/node-js-child-processes-everything-you-need-to-know-e69498fe970a/
