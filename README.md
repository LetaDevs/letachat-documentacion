# Leta chat DOCUMENTACIÓN

![diseños](pages.png)

### Casos de uso:

- [x] crear cuenta
- [x] iniciar sesión
- [ ] elegir chat
- [x] enviar mensaje al chat actual
- [x] buscar chat

## Base de datos:

Se usará **MongoDB** como base de datos, será necesario crear 2 colecciones: **usuarios** y **mensajes**.

**usuarios -** la colección de usuarios tendrá la siguiente estructura:

```js
  {
    name: String,
    email: String (unique),
    password: String,
    online: Boolean (defaul false),
  }
```

**mensajes -** la colección de mensajes tendrá la siguiente estructura:

```js
  {
    from: ObjectId,
    to: ObjectId,
    msg: String,
    createdAt: Date,
    updatedAt: Date,
  }
```

---

## Backend

El backend será desarrollado en Express.js

### routes:

el manejo de las rutas se hará en archivos independientes para mantener un orden.

Todo lo relacionado a la autenticación de usuarios se manejará en el archivo **authRouter.js**

---

### crear cuenta

**POST** _referer/api/v1/auth/signup_ -> para crear una cuenta.

En el body se recibirá un objeto con los datos del usuario, así:

```js
  {
    name: 'Pepito Perez',
    email: 'pepito@correo.com',
    password: 'mipassword'
  }
```

se retornará un objeto con la siguiente información:

```js
  {
    ok: Boolean,
    token: jwt
    user: {
      id: ObjectId,
      name: String
    }
  }
```

---

### iniciar sesión

**POST** _referer/api/v1/auth/login_ -> para iniciar sesión.

En el body se recibirá un objeto con las credenciales del usuario, así:

```js
  {
    email: 'pepito@correo.com',
    password: 'mipassword'
  }
```

se retornará un objeto con la siguiente información:

```js
  {
    ok: Boolean,
    token: jwt
    user: {
      id: ObjectId,
      name: String
    }
  }
```

---

Todo lo relacionado al chat (obtener lista usuarios, obtener mensajes entre dos usuarios, etc) se manejará en el archivo **chatRouter.js**

---

### obtener mensajes de un chat

**GET** _referer/api/v1/chat/messages_ -> para obtener los últimos 30 mensajes entre el usuario y el chat actual

Será necesario un jwt en los headers (x-auth-token) para devolver la información.

En el body se recibirá un objeto con los ObjectId de los usuarios implicados, así:

```js
  {
    from: ObjectId,
    to: ObjectId,
  }
```

---

### guardar mensaje

**POST** _referer/api/v1/chat/new-msg_ -> para almacenar un nuevo mensaje

Será necesario un jwt en los headers (x-auth-token) para realizar la acción.

En el body se recibirá un objeto con la información del mensaje, así:

```js
  {
    from: ObjectId,
    to: ObjectId,
    msg: 'hola mundo'
  }
```

---

### buscar chat

**GET** _referer/api/v1/chat/search?q=(keyword)_ -> para buscar un chat

Será necesario un jwt en los headers (x-auth-token) para realizar la acción.

se retornarán todos los usuarios cuyo campo name contenga la keyword recibida

## Frontend

Para el frontend se usará React.js

### Context

---

#### AuthContext.js

En el AuthContext se manejarán todos los estados relacionados a la autenticación del usuario. Como no se requieren muchos estados en esta parte, éstos se menajarán localmente con el useState hook.

**auth**

```js
const [auth, setAuth] = useState({
	fetching: true,
	logged: false,
	user: {},
});
```

---

#### ChatContext.js

En el ChatContext se manejarán todos los estados relacionados al chat en sí. En este caso se usará el useReducer hook para el manejo del estado global.

```js
const initialState = {
	users: [],
	messages: [],
	activeChat: ObjectId,
};
```

---

#### SocketContext.js

En el SocketContext se manejará directamente la escucha de eventos para disparar las acciones necesarias, y se compartirá la variable **socket** obtenida desde el useSocket que contiene la conexión con los sockets, de esta forma los componentes desde los cuales se necesité emitir un evento, podran hacerlo mediante dicha variable.

---

## Custom Hooks

### useSocket

Desde este custom hook se manejará la conexión con el **socket server** mediante la librería _socket.io-client_

---

## Comunicación en tiempo real

Para manejar la comunicación en tiempo real entre clientes y servidor, se utilizará la librería **socket.io@4.2.0**

se creara un controller **socketController** con las siguientes funciones:

**setUserOnline**

cambiará el estado online del usuari a true `online: true`

**setUserOffline**

cambiará el estado online del usuari a false `online: false`

**getUsers**

retornará un array con todos los usuarios registrados

**saveMessage(message)**

guardará en la base de datos el **message** y retornará el _msg_ con la estructura completa.

la estructura del message debe ser esta:

```js
  {
    from: ObjectId,
    to: ObjectId,
    msg: String
  }
```

---

### on connection

Cuando un cliente se conecte al servidor, será necesario válidar el jwt asociado al evento. Si el jwt no es válido se desconectará al cliente:

```js
const [ok, uid] = verifyJWT(socket.handshake.query['x-auth-token']);

if (!ok) {
	return socket.disconnect();
}
```

Si el jwt es válido, se debe cambiar el estado online del usuario a true `online: true`

```js
await setUserOnline(uid);
```

Luego, se agregará al usuario a una sala con un **id** igual a su **uid**

```js
socket.join(uid);
```

Se emitirá a todos los clientes la lista de usuarios con el fin de refrescar el estado **online** de todos.

```js
this.io.emit('user-list', await getUsers());
```

Escuchando los nuevos mensajes.

```js
socket.on('new-msg', async (message) => {
	const msg = await saveMessage(message);
	// enviar el mensaje al destinatario y al emisor
	this.io.to(message.to).emit('new-msg', msg);
	this.io.to(message.from).emit('new-msg', msg);
});
```

Cuando un cliente se desconecte hay que informar a los demás que ese usuario ya no está online.

```js
socket.on('disconnect', async () => {
	const msg = await setUserOffline(uid);
	// emitir la lista de usuarios actualizada
	this.io.emit('user-list', await getUsers());
});
```

---

## Creando cuenta...

![diseños](signup.png)

- Para crear una cuenta el usuario tendrá que ingresar su nombre, email y password.

- Con el fin de prevenir errores, el botón del formulario estará deshabilitado hasta que todos los inputs tengan información

- Se validarán los posibles errores enviados desde el backend, mostrando una alerta en función del error recibido.

- Si la acción es correcta, se guardará en el localstorage el jwt recibido y se actualizará el state del AuthContext, así:

```js
  {
    fetching: false,
    logged: true,
    user:{
      id: ObjectId,
      name: String
    }
  }
```

- se hará la conexión al socket server (enviando el jwt en el evento)

---

## Iniciando sesión...

![diseños](login.png)

- Para iniciar sesión, el usuario tendrá que ingresar su email y password.

- Con el fin de prevenir errores, el botón del formulario estará deshabilitado hasta que todos los inputs tengan información

- Se validarán los posibles errores enviados desde el backend, mostrando una alerta en función del error recibido.

- Si la acción es correcta, se guardará en el localstorage el jwt recibido y se actualizará el state del AuthContext, así:

```js
  {
    fetching: false,
    logged: true,
    user:{
      id: ObjectId,
      name: String
    }
  }
```

- se hará la conexión al socket server (enviando el jwt en el evento)
