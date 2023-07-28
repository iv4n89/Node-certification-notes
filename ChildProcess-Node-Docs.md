## Child process

El módulo child_process provee la habilidad de generar procesos hijos de una manera similar a popen(3). Esto es principalmente realizado por la función spawn().

```
const { spawn } = require('child_process');
const ls = spawn('ls', ['-lh', '/usr']);

ls.stdout.on('data', (data) => {
  console.log(`stdout: ${data}`);
});

ls.stderr.on('data', (data) => {
  console.error(`stderr: ${data}`);
});

ls.on('close', (code) => {
  console.log(`child process exited with code ${code}`);
});
```

Por defecto, los pipes para stdin, stdout y stderr son establecidos entre el proceso padre y el hijo generado. Estos pipes tienen capacidad limitada. El el proceso hijo
escribe hacia stdout en exceso de este límite sin que el output haya sido capturado, el proceso hijo se bloqueará esperando por el pipe por más datos. Esto es identico al comportamiento
de pipes en un shell. Usar { stdio: 'ignore' } si el output no va a ser consumido.

La búsqueda de comandos será realizado usando el options.env.PATH si es pasado en options, de lo contrario se usará process.env.PATH. 

La función spawn() genera un proceso hijo asíncronamente, sin bloquear el event loop de node.js. La función spawnSync() provee una funcionalidad equivalente de una manera síncrona, que bloquea el event loop hasta que el proceso generado acaba o es terminado.

Por conveniencia, el módulo child_process provee alternativas síncronas y asíncronas.

- child_process.exec(): Genera un shell y corre un comando dentro de este shell, pasando el stdout y stderr a un callback cuando es completado.
- child_process.execFile(): Similar a exec excepto que genera el comando directamente sin primero generar un shell por defecto.
- child_process.fork(): Genera un nuevo proceso de node e invoca un modulo específico con un canal de comunicación IPC establecido que permite intercambiar mensajes.
- child_process.execSync(): Una versión síncrona de child_process.exec() que bloquea el event loop de node.
- child_process.exeFileSync(): Una versión síncrona de child_process.execFile() que bloquea el event loop de node.

Para ciertos casos, como automatizar shell scripts, los procesos síncronos pueden ser más convenientes. En muchos casos, sin embargo, el método síncrono
puede tiene un impacto significativo en el rendimiento debido al parón en el event loop mientras el child process se completa.

## Creación de proceso asíncrono

Los métodos spawn, fork, exec y execFile siguen el patrón asíncrono típico de otras APIs de node.

Cada método retorna una instancia de ChildProcess. Estos objetos implementan EventEmitter, permitiendo al padre registrar listeners que son llamados
cuando ciertos eventos ocurren durante el ciclo de vida del proceso hijo.

Los métodos exec y execFile adicionalmente permiten un callback adicional que es invocado cuando el proceso hijo acaba.

## Generar ficheros .bat y .cmd en Windows

La importancia de la distinción entre exec y execFile puede variar entre plataformas. En Unix, execFile puede ser más eficiente ya que no genera un shell por defecto.
En Windows, los .bat o .cmd no pueden ser ejecutados por sí mismos sin un terminal, y no pueden ser ejecutados por execFile. En Windows podemos correr .bat o .cmd
con spawn usando la opción de shell, con exec o generando cmd.exe y pasándole el .bat o .cmd como argumento (que es lo que hace spawn con la opción shell y exec)

```
// Sólo Windows

const { spawn } =
