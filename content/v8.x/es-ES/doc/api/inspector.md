# Inspector

<!--introduced_in=v8.0.0-->

> Estabilidad: 1 - Experimental

El módulo `inspector` proporciona una API para interactuar con el inspector de V8.

Puede ser accedido utilizando:

```js
const inspector = require('inspector');
```

## inspector.open([port[, host[, wait]]])

* `port` {number} Puerto en el cual escuchar para las conexiónes de inspector. Opcional. **Predeterminado:** lo que fue especificado en la CLI.
* `host` {string} Huésped en el cual escuchar para las conexiónes de inspector. Opcional. **Predeterminado:** lo que fue especificado en la CLI.
* `wait` {boolean} Bloquear hasta que un cliente se haya conectado. Opcional. **Predeterminado:** `false`.

Active el inspector en el host y en el puerto. Equivalente a `node
--inspect=[[host:]port]`, pero puede hacerse mediante programación después que el node haya iniciado.

Si la espera es `true`, se bloqueará hasta que un cliente se haya conectado al puerto inspector y el control de flujo haya sido pasado al cliente depurador.

### inspector.close()

Desactiva el inspector. Se bloquea hasta que no hayan conexiones activas.

### inspector.url()

Devuelve el URL del inspector activo o `undefined` si no hay ninguno.

## Clase: inspector.Session

La `inspector.Session` es utilizada para enviar mensajes al back-end del inspector de V8 y para recibir respuestas y notificaciones de mensajes.

### Constructor: nueva inspector.Session()
<!-- YAML
added: v8.0.0
-->

Crea una nueva instancia de la clase `inspector.Session`. La sesión de inspector necesita ser conectada a través de [`session.connect()`][] antes de que los mensajes puedan ser enviados al backend del inspector.

`inspector.Session` es un [`EventEmitter`][] con los siguientes eventos:

### Evento: 'inspectorNotification'
<!-- YAML
added: v8.0.0
-->

* {Object} El objeto de mensaje de notificación

Emitido cuando se recibe cualquier notificación del inspector de V8.

```js
session.on('inspectorNotification', (message) => console.log(message.method));
// Debugger.paused
// Debugger.resumed
```

También es posible suscribirse únicamente a notificaciones con método específico:

### Evento: &lt;inspector-protocol-method&gt;
<!-- YAML
added: v8.0.0
-->

* {Object} El objeto de mensaje de notificación

Emitido cuando se recibe una notificación del inspector que tiene un campo de método establecido al valor de `<inspector-protocol-method>`.

El siguiente snippet instala un listener en el evento de [`Debugger.paused`][] e imprime la razón para la suspensión del programa cada vez que la ejecución del programa es suspendida (a través de puntos de ruptura, por ejemplo):

```js
session.on('Debugger.paused', ({ params }) => {
  console.log(params.hitBreakpoints);
});
// [ '/the/file/that/has/the/breakpoint.js:11:0' ]
```

### session.connect()
<!-- YAML
added: v8.0.0
-->

Conecta una sesión al back-end del inspector. Se arrojará una excepción si ya hay una sesión conectada establecida a través del API o por un front-end conectado al puerto WebSocket del Inspector.

### session.post(method\[, params\]\[, callback\])
<!-- YAML
added: v8.0.0
-->

* method {string}
* params {Object}
* callback {Function}

Publica un mensaje al back-end del inspector. Se le notificará a `callback` cuando se reciba una respuesta. `callback` es una función que acepta dos argumentos opcionales: error y message-specific result.

```js
session.post('Runtime.evaluate', { expression: '2 + 2' },
             (error, { result }) => console.log(result));
// Salida: { type: 'number', value: 4, description: '4' }
```

La última versión del protocolo del inspector V8 está publicada en el [Chrome DevTools Protocol Viewer](https://chromedevtools.github.io/devtools-protocol/v8/).

Node.js inspector supports all the Chrome DevTools Protocol domains declared by V8. Chrome DevTools Protocol domain proporciona una interfaz para interactuar con uno de los agentes "runtime" utilizados para inspeccionar el estado de la aplicación y escuchar los eventos de "run-time".

### session.disconnect()
<!-- YAML
added: v8.0.0
-->

Cierra la sesión inmediatamente. Todos los callbacks de mensajes pendientes serán llamados con un error. Se necesitará llamar a [`session.connect()`] para poder enviar mensajes de nuevo. La sesión reconectada perderá todos los estados del inspector, como los agentes habilitados o los puntos de ruptura configurados.

## Ejemplo de uso

### CPU Profiler

Además del depurador, varios V8 Profilers están disponibles a través del protocolo DevTools. Aquí está un ejemplo sencillo que muestra cómo usar el [CPU profiler](https://chromedevtools.github.io/devtools-protocol/v8/Profiler):

```js
const inspector = require('inspector');

const session = new inspector.Session();
session.connect();

session.post('Profiler.enable', () => {
  session.post('Profiler.start', () => {
    // invoke business logic under measurement here...

    // un tiempo después...
    session.post('Profiler.stop', ({ profile }) => {
      // write profile to disk, upload, etc.
    });
  });
});
```
