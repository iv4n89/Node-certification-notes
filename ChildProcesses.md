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


