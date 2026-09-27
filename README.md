# Ice Chat — Chat Distribuido con ZeroC Ice

## Descripción

Este proyecto implementa un **sistema de chat distribuido de consola** utilizando **ZeroC Ice** como middleware para la comunicación entre clientes y un servidor.

El sistema permite que varios usuarios se conecten a una misma sala de chat, envíen mensajes, consulten los usuarios conectados y cierren su sesión.

La comunicación entre el cliente y el servidor se realiza mediante **invocaciones remotas (RPC)** definidas mediante un contrato **Slice**.

---

## Tecnologías utilizadas

* **Java 17**
* **Gradle**
* **ZeroC Ice 3.7.x**
* **Slice (`.ice`)** para definir la interfaz remota
* **TCP/IP** para la comunicación entre cliente y servidor
* **Git** para el control de versiones

---

## Arquitectura del proyecto

El proyecto está dividido en tres módulos principales:

```text
ice-chat-project/
│
├── common/
│   └── src/main/
│       ├── java/
│       │   └── ChatApp/
│       │       ├── ChatRoom.java
│       │       ├── ChatRoomPrx.java
│       │       ├── ChatException.java
│       │       ├── ChatMessage.java
│       │       └── MessageSeqHelper.java
│       │
│       └── slice/
│           └── Chat.ice
│
├── server/
│   └── src/main/java/chat/server/
│       ├── ServerMain.java
│       └── ChatRoomI.java
│
├── client/
│   └── src/main/java/chat/client/
│       └── ClientMain.java
│
├── build.gradle
├── settings.gradle
├── gradlew
└── gradlew.bat
```

### `common`

Contiene el contrato de comunicación y las clases compartidas entre el cliente y el servidor.

El archivo:

```text
common/src/main/slice/Chat.ice
```

define la interfaz remota `ChatRoom`, las estructuras de mensajes y las excepciones utilizadas por el sistema.

A partir de este archivo, `slice2java` genera automáticamente las clases Java necesarias para la comunicación con ZeroC Ice.

### `server`

Contiene la implementación del servidor.

Sus principales clases son:

* `ServerMain`: inicializa ZeroC Ice, crea el adaptador y publica el servicio.
* `ChatRoomI`: implementa las operaciones definidas en la interfaz `ChatRoom`.

El servidor mantiene:

* Usuarios conectados.
* Historial de mensajes.
* Identificadores consecutivos para los mensajes.
* Mensajes de entrada y salida de los usuarios.

### `client`

Contiene la aplicación de consola que utiliza el usuario para conectarse al servidor.

`ClientMain` permite:

* Iniciar sesión mediante un nickname.
* Enviar mensajes.
* Consultar los usuarios conectados.
* Recibir mensajes nuevos.
* Cerrar sesión.

---

## Interfaz remota

La interfaz definida en `Chat.ice` es:

```slice
interface ChatRoom {

    void login(string nickname) throws ChatException;

    void postMessage(string nickname, string message) throws ChatException;

    idempotent MessageSeq getPendingMessages(
        string nickname,
        long lastMessageId
    );

    idempotent UserSeq getOnlineUsers();

    void logout(string nickname);
};
```

### Operaciones principales

| Operación              | Descripción                                             |
| ---------------------- | ------------------------------------------------------- |
| `login()`              | Registra un usuario en la sala.                         |
| `postMessage()`        | Envía un mensaje al chat.                               |
| `getPendingMessages()` | Obtiene los mensajes posteriores al último ID recibido. |
| `getOnlineUsers()`     | Obtiene la lista de usuarios conectados.                |
| `logout()`             | Desconecta un usuario de la sala.                       |

---

## Requisitos

Antes de ejecutar el proyecto se necesita tener instalado:

* Java 17 o una versión compatible con la configuración del proyecto.
* Gradle o utilizar el Gradle Wrapper incluido.
* ZeroC Ice.
* La herramienta `slice2java` disponible en el `PATH`.

Para comprobar Java:

```bash
java -version
```

Para comprobar `slice2java`:

```bash
slice2java --version
```

---

## Instalación

Clonar el repositorio:

```bash
git clone <URL_DEL_REPOSITORIO>
```

Entrar al proyecto:

```bash
cd ice-chat-project
```

En macOS/Linux se puede utilizar el Gradle Wrapper incluido:

```bash
./gradlew
```

En Windows:

```bash
gradlew.bat
```

---

## Compilación

Desde la carpeta `ice-chat-project`, ejecutar:

```bash
./gradlew build
```

Durante la compilación, el módulo `common` genera las clases Java a partir del archivo:

```text
common/src/main/slice/Chat.ice
```

mediante `slice2java`.

También se puede compilar específicamente el módulo común:

```bash
./gradlew :common:build
```

---

## Ejecución

El servidor debe iniciarse antes que los clientes.

### 1. Iniciar el servidor

Desde la carpeta `ice-chat-project`:

```bash
./gradlew :server:run
```

Si el servidor inicia correctamente, se mostrará información similar a:

```text
=================================================
 SERVIDOR ZEROC ICE INICIADO EXITOSAMENTE
 Puerto TCP: 10000 | Endpoint: default -p 10000
 Identidad del Servicio: ChatService
=================================================
Esperando llamadas remotas de clientes...
```

El servidor queda esperando conexiones de los clientes mediante el puerto TCP `10000`.

---

### 2. Iniciar un cliente

Abrir otra terminal y entrar nuevamente a la carpeta:

```bash
cd ice-chat-project
```

Ejecutar:

```bash
./gradlew :client:run
```

El cliente solicitará un nickname:

```text
=== BIENVENIDO AL CHAT DISTRIBUIDO ZEROC ICE ===
Ingrese su nickname:
```

Después de ingresar un nickname válido, el usuario podrá interactuar con el chat.

---

## Comandos disponibles

### `/users`

Muestra los usuarios conectados actualmente:

```text
/users
```

Ejemplo:

```text
>>> Usuarios activos (2): Gaby, Michelle
```

### `/exit`

Cierra la sesión del usuario:

```text
/exit
```

El cliente informa al servidor que el usuario abandonó la sala.

---

## Envío y recepción de mensajes

Una vez conectado, cualquier texto que no corresponda a un comando se interpreta como un mensaje.

Ejemplo:

```text
> Hola, ¿cómo están?
```

Otro cliente conectado puede recibir:

```text
[15:30:25] <Gaby>: Hola, ¿cómo están?
```

El cliente utiliza un hilo en segundo plano para consultar periódicamente si existen nuevos mensajes.

La consulta se realiza utilizando el identificador del último mensaje recibido:

```text
getPendingMessages(nickname, lastMessageId)
```

Esto permite obtener únicamente los mensajes posteriores al último mensaje procesado.

---

## Manejo de usuarios

El servidor evita que dos usuarios utilicen simultáneamente el mismo nickname.

Si se intenta ingresar con un nombre que ya está conectado, el servidor responde con una excepción:

```text
[RECHAZADO POR SERVIDOR] El nickname 'Gaby' ya se encuentra conectado.
```

El usuario debe ingresar otro nickname para continuar.

---

## Historial de mensajes

El servidor mantiene en memoria el historial de mensajes mediante una estructura concurrente.

Cada mensaje contiene:

```text
id
sender
text
timestamp
```

Ejemplo:

```text
ID: 5
Sender: Gaby
Text: Hola
Timestamp: 15:32:10
```

También se generan mensajes internos cuando un usuario entra o abandona la sala.

Ejemplo:

```text
SISTEMA: Gaby se unio a la sala.
SISTEMA: Gaby ha abandonado la sala.
```

---

## Comunicación cliente-servidor

El cliente utiliza el siguiente proxy para conectarse al servicio remoto:

```text
ChatService:default -h 127.0.0.1 -p 10000
```

Esto significa:

* **Servicio:** `ChatService`
* **Host:** `127.0.0.1`
* **Puerto:** `10000`

El servidor publica el servicio mediante el endpoint:

```text
default -p 10000
```

Por defecto, el proyecto está preparado para ejecutar cliente y servidor en la misma máquina.

---

## Flujo general del sistema

```text
                 ┌──────────────────────┐
                 │       CLIENTE 1      │
                 │      ClientMain      │
                 └──────────┬───────────┘
                            │
                            │ ZeroC Ice / RPC
                            │
                            ▼
                 ┌──────────────────────┐
                 │       SERVIDOR       │
                 │      ServerMain      │
                 │                      │
                 │      ChatRoomI       │
                 └──────────┬───────────┘
                            │
                            │ ZeroC Ice / RPC
                            │
                            ▼
                 ┌──────────────────────┐
                 │       CLIENTE 2      │
                 │      ClientMain      │
                 └──────────────────────┘
```

El servidor centraliza la información de los usuarios y mensajes, mientras que los clientes realizan invocaciones remotas para interactuar con la sala.

---

## Concurrencia

El servidor utiliza estructuras preparadas para trabajar con múltiples clientes:

* `ConcurrentHashMap.newKeySet()` para los usuarios conectados.
* `CopyOnWriteArrayList` para el historial de mensajes.
* `AtomicLong` para generar identificadores de mensajes.
* Métodos sincronizados para operaciones que requieren coordinación.

El cliente también utiliza un hilo independiente para consultar mensajes nuevos mientras el hilo principal permanece disponible para recibir comandos desde la consola.

---

## Manejo de errores

El sistema utiliza `ChatException` para comunicar errores relacionados con las operaciones del chat.

Entre los casos contemplados se encuentran:

* Nickname vacío.
* Nickname duplicado.
* Intento de enviar mensajes sin iniciar sesión.
* Fallos de comunicación con el servidor.
* Imposibilidad de establecer la conexión con el servicio Ice.

---

## Tecnologías y conceptos de computación distribuida

Este proyecto permite aplicar conceptos como:

* Middleware para sistemas distribuidos.
* Invocación de métodos remotos (RPC).
* Arquitectura cliente-servidor.
* Interfaces remotas.
* Serialización y deserialización de datos.
* Comunicación mediante TCP/IP.
* Concurrencia.
* Manejo de excepciones distribuidas.
* Identificación de servicios mediante proxies.
* Generación de código a partir de contratos Slice.

---

## Autores

**Gabriela Cardona Bernal - Codigo: A00410856**

**Jazmin Michelle Galvez Orozco - Codigo: A00410285**

Universidad Icesi

---
