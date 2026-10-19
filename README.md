# Chat WebSocket - Guía para Alumnos

Este proyecto implementa un sistema de chat en tiempo real utilizando **WebSocket** y el protocolo **STOMP** con Spring Boot. A continuación se explica el flujo completo de funcionamiento y los conceptos clave.

## 📋 Índice

1. [Descripción del Proyecto](#descripción-del-proyecto)
2. [Arquitectura y Flujo](#arquitectura-y-flujo)
3. [Componentes del Sistema](#componentes-del-sistema)
4. [Flujo de Mensajes](#flujo-de-mensajes)
5. [Glosario de Conceptos](#glosario-de-conceptos)
6. [Ejecución del Proyecto](#ejecución-del-proyecto)

---

## 📝 Descripción del Proyecto

ChatWS es una aplicación de chat en tiempo real que permite a múltiples usuarios conectarse, enviar mensajes y recibir actualizaciones instantáneas sin necesidad de recargar la página. Utiliza una arquitectura basada en eventos donde los mensajes se propagan automáticamente a todos los clientes conectados.

**Tecnologías principales:**
- Spring Boot 4.1.1
- WebSocket para comunicación bidireccional
- STOMP como protocolo de mensajería
- RabbitMQ como broker STOMP externo
- SockJS para compatibilidad con navegadores antiguos
- Jackson para serialización JSON

---

## 🏗️ Arquitectura y Flujo

### Diagrama de Arquitectura

```
┌─────────────────┐         WebSocket          ┌──────────────────┐
│   Cliente Web   │ ◄────────────────────────► │   Servidor       │
│  (Navegador)    │     (ws://localhost:8080)  │   Spring Boot    │
└─────────────────┘                            └────────┬─────────┘
                                                        │ STOMP relay
                                                        ▼
                                               ┌──────────────────┐
                                               │    RabbitMQ      │
                                               │ 5672 / 61613     │
                                               └──────────────────┘
       │                                                 │
       │ 1. Conexión WebSocket                          │
       │ 2. Suscripción a /topic/public                 │
       │ 3. Envío a /app/chat.sendMessage               │
       │                                                 │
       │                                                 │
       │ 4. Recepción de mensajes                       │
       │    (broadcast desde /topic/public)             │
```

### Flujo Completo de un Mensaje

1. **Conexión:** El cliente web establece una conexión WebSocket con el servidor
2. **Suscripción:** El cliente se suscribe al canal `/topic/public` para recibir mensajes
3. **Envío:** El cliente envía un mensaje a `/app/chat.sendMessage`
4. **Procesamiento:** El `ChatController` recibe el mensaje
5. **Broadcast:** El mensaje se reenvía a todos los suscriptores de `/topic/public`
6. **Recepción:** Todos los clientes conectados reciben el mensaje simultáneamente

---

## 🔧 Componentes del Sistema

### 1. Configuración WebSocket (`WebSocketConfig.java`)

Esta clase configura la infraestructura de mensajería:

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer
```

**Responsabilidades:**
- Reenvía los mensajes STOMP a RabbitMQ mediante el broker relay
- Configura el prefijo `/topic` para suscripciones
- Configura el prefijo `/app` para mensajes del cliente
- Registra el endpoint `/chat` para conexiones WebSocket
- Habilita SockJS como fallback

### 2. Controlador de Chat (`ChatController.java`)

Maneja el flujo de mensajes:

```java
@Controller
public class ChatController {
    @MessageMapping("/chat.sendMessage")
    @SendTo("/topic/public")
    public ChatMessage enviarMensaje(ChatMessage mensaje)
}
```

**Responsabilidades:**
- Recibe mensajes enviados por los clientes
- Reenvía los mensajes a todos los suscriptores
- Gestiona la entrada de nuevos usuarios

### 3. Entidad de Mensaje (`ChatMessage.java`)

Representa la estructura de un mensaje:

```java
public record ChatMessage(String tipo, String usuario, String contenido)
```

**Campos:**
- `tipo`: Tipo de mensaje (CHAT, JOIN, LEAVE)
- `usuario`: Nombre del usuario que envía el mensaje
- `contenido`: Texto del mensaje

### 4. Cliente Web (`index.html` + `script.js`)

Interfaz de usuario para el chat:

**Funciones principales:**
- `connect()`: Establece conexión WebSocket con SockJS y STOMP
- `disconnect()`: Cierra la conexión
- `sendMessage()`: Envía mensajes al servidor
- `showMessage()`: Muestra mensajes recibidos en la UI

### 5. Cliente de Prueba (`ChatBrowser.java`)

Cliente Java para pruebas automatizadas:

**Funcionalidades:**
- Conecta al servidor WebSocket programáticamente
- Se suscribe al canal de mensajes
- Envía mensajes de prueba
- Recibe y procesa respuestas

---

## 🔄 Flujo de Mensajes Detallado

### Paso 1: Conexión del Cliente

**Cliente (JavaScript):**
```javascript
const socket = new SockJS('http://localhost:8080/chat');
stompClient = Stomp.over(socket);
stompClient.connect({}, function (frame) {
    stompClient.subscribe('/topic/public', function (message) {
        showMessage(JSON.parse(message.body));
    });
});
```

**Servidor (WebSocketConfig):**
- Registra endpoint `/chat` con soporte SockJS
- Configura broker para `/topic/*`

### Paso 2: Envío de Mensaje

**Cliente (JavaScript):**
```javascript
stompClient.send("/app/chat.sendMessage", {}, JSON.stringify({
    tipo: 'CHAT',
    usuario: username,
    contenido: content
}));
```

**Servidor (ChatController):**
```java
@MessageMapping("/chat.sendMessage")
@SendTo("/topic/public")
public ChatMessage enviarMensaje(ChatMessage mensaje) {
    return mensaje; // Se envía automáticamente a /topic/public
}
```

### Paso 3: Recepción por Todos los Clientes

Cada cliente suscrito a `/topic/public` recibe el mensaje automáticamente a través del callback de suscripción.

---

## 📚 Glosario de Conceptos

### WebSocket
**Definición:** Protocolo de comunicación que proporciona canales full-duplex (bidireccionales) sobre una sola conexión TCP.

**En este proyecto:** Permite comunicación en tiempo real entre el servidor y los clientes sin necesidad de recargar la página. A diferencia de HTTP, donde el cliente debe solicitar información, WebSocket permite que el servidor envíe mensajes activamente a los clientes.

**Características clave:**
- Comunicación bidireccional
- Baja latencia
- Mantenimiento de conexión persistente
- Ahorro de ancho de banda (sin headers HTTP repetidos)

---

### STOMP (Simple Text Oriented Messaging Protocol)
**Definición:** Protocolo de mensajería simple y basado en texto que define el formato y la semántica para intercambiar mensajes entre clientes y servidores.

**En este proyecto:** Actúa como el protocolo de aplicación sobre WebSocket. Define cómo se estructuran los mensajes, cómo se suscriben los clientes a canales y cómo se envían mensajes al servidor.

**Conceptos clave de STOMP:**
- **FRAME:** Unidad básica de comunicación en STOMP (ej: SEND, SUBSCRIBE, MESSAGE)
- **DESTINATION:** Donde se envía o de donde se recibe un mensaje (ej: `/topic/public`)
- **SUBSCRIPTION:** Registro de interés en recibir mensajes de un destino específico

**Ejemplo de frame STOMP:**
```
SEND
destination:/app/chat.sendMessage
content-type:application/json

{"tipo":"CHAT","usuario":"Juan","contenido":"Hola"}
```

---

### Broker de Mensajes
**Definición:** Componente intermedio que recibe mensajes de productores y los distribuye a consumidores interesados. Actúa como un intermediario que desacopla a emisores y receptores.

**En este proyecto:** RabbitMQ gestiona la distribución de mensajes; Spring reenvía los destinos STOMP con `enableStompBrokerRelay`.

**Tipos de destinos que maneja:**
- **Topics (`/topic/*`):** Mensajes enviados a múltiples suscriptores (broadcast)
- **Queues (`/queue/*`):** Mensajes enviados a un solo consumidor (no usado en este proyecto)

**Flujo a través del broker:**
```
Cliente A → /app/chat.sendMessage → Broker → /topic/public → Cliente B, C, D
```

---

### SockJS
**Definición:** Biblioteca JavaScript que proporciona una capa de abstracción sobre WebSocket, ofreciendo alternativas de transporte cuando WebSocket no está disponible.

**En este proyecto:** Se usa como fallback para navegadores que no soportan WebSocket nativo o cuando hay firewalls/proxies que bloquean conexiones WebSocket.

**Transportes que SockJS puede usar (en orden de preferencia):**
1. WebSocket (transporte nativo)
2. XHR Streaming
3. XHR Polling
4. Iframe-based methods

**Uso en el código:**
```java
registry.addEndpoint("/chat").withSockJS();
```

---

### Topic
**Definición:** Tipo de destino en un sistema de mensajería donde los mensajes se distribuyen a todos los suscriptores activos (patrón publish-subscribe).

**En este proyecto:** `/topic/public` es el topic donde se publican todos los mensajes del chat. Todos los clientes conectados se suscriben a este topic para recibir los mensajes.

**Diferencia con Queue:**
- **Topic:** Un mensaje va a múltiples suscriptores (broadcast)
- **Queue:** Un mensaje va a un solo consumidor (point-to-point)

---

### Endpoint
**Definición:** Punto de entrada específico en una API o servicio donde los clientes pueden enviar solicitudes o establecer conexiones.

**En este proyecto:** `/chat` es el endpoint WebSocket donde los clientes establecen la conexión inicial.

**Jerarquía de endpoints en el proyecto:**
- `/chat` - Endpoint de conexión WebSocket
- `/app/chat.sendMessage` - Endpoint para enviar mensajes (desde cliente)
- `/topic/public` - Topic para recibir mensajes (hacia clientes)

---

### Message Mapping
**Definición:** Anotación en Spring que mapea mensajes entrantes a métodos específicos del controlador basándose en su destino.

**En este proyecto:** `@MessageMapping("/chat.sendMessage")` indica que los mensajes enviados a `/app/chat.sendMessage` se procesan en el método `enviarMensaje`.

**Relación con @RequestMapping:**
- `@RequestMapping`: Para endpoints HTTP REST
- `@MessageMapping`: Para endpoints WebSocket/STOMP

---

### SendTo
**Definición:** Anotación que especifica el destino donde se enviará el valor de retorno de un método del controlador.

**En este proyecto:** `@SendTo("/topic/public")` hace que el mensaje retornado por el método se envíe automáticamente a todos los suscriptores del topic público.

**Flujo:**
```
Cliente → @MessageMapping → Método del Controller → @SendTo → Topic → Clientes
```

---

### Full-Duplex
**Definición:** Capacidad de un canal de comunicación para transmitir datos en ambas direcciones simultáneamente.

**En este proyecto:** WebSocket es full-duplex, lo que significa que el servidor y el cliente pueden enviar mensajes al mismo tiempo sin interferencia.

**Comparación:**
- **Half-Duplex (HTTP tradicional):** Solo una dirección a la vez
- **Full-Duplex (WebSocket):** Ambas direcciones simultáneas

---

### JSON (JavaScript Object Notation)
**Definición:** Formato ligero de intercambio de datos basado en texto, fácil de leer y escribir para humanos, y fácil de parsear y generar para máquinas.

**En este proyecto:** Se usa para serializar los objetos `ChatMessage` al enviarlos por la red y deserializarlos al recibirlos.

**Ejemplo de mensaje JSON:**
```json
{
  "tipo": "CHAT",
  "usuario": "Ana",
  "contenido": "Hola a todos"
}
```

---

### Jackson
**Definición:** Biblioteca Java para procesamiento JSON, usada para serializar objetos Java a JSON y viceversa.

**En este proyecto:** Spring usa Jackson automáticamente (a través de `MappingJackson2MessageConverter`) para convertir los objetos `ChatMessage` a JSON al enviar mensajes y convertir JSON a objetos al recibirlos.

---

## 🚀 Ejecución del Proyecto

Antes de iniciar Spring Boot, levanta RabbitMQ con el plugin STOMP habilitado. Con Docker Desktop instalado y abierto, ejecuta:

```powershell
docker run -d --name chatws-rabbitmq `
  -p 5672:5672 -p 61613:61613 -p 15672:15672 `
  rabbitmq:3-management
```

Habilita el plugin STOMP (una vez por contenedor):

```powershell
docker exec chatws-rabbitmq rabbitmq-plugins enable rabbitmq_stomp
```

La aplicación se conecta a RabbitMQ en `localhost:5672` con `guest/guest`, y el relay STOMP usa `localhost:61613`. El panel de administración queda en <http://localhost:15672> (usuario y contraseña: `guest`). Si el contenedor ya existe, puedes iniciarlo con `docker start chatws-rabbitmq`.

Luego ejecuta la aplicación desde la raíz del proyecto:

```powershell
./mvnw.cmd spring-boot:run
```

Requisitos: JDK compatible con el proyecto (Java 26 según `pom.xml`), Docker Desktop con RabbitMQ ejecutándose, y conexión a Internet la primera vez para descargar dependencias Maven.

### Requisitos Previos
- JDK 26 (el proyecto declara Java 26 en `pom.xml`)
- Docker Desktop con Docker iniciado
- Navegador web moderno

### Pasos para Ejecutar

1. **Clonar o descargar el proyecto**

2. **Iniciar RabbitMQ** como se indica arriba. Para los siguientes usos, si el contenedor ya existe, basta con ejecutar `docker start chatws-rabbitmq`.

3. **Compilar el proyecto:**
   ```bash
   ./mvnw.cmd clean install
   ```

4. **Ejecutar el servidor:**
   ```bash
   ./mvnw.cmd spring-boot:run
   ```

5. **Abrir el cliente web:**
   - Navegar a `http://localhost:8080/index.html`
   - Ingresar nombre de usuario
   - Hacer clic en "Conectar"

6. **Probar el cliente Java (opcional):**
   ```bash
   # Ejecutar la clase ChatBrowser
   java -cp target/classes:target/dependency/* org.example.chatws.utility.ChatBrowser
   ```

### Probar con Múltiples Clientes

1. Abre varias pestañas del navegador
2. Conecta cada una con diferentes nombres de usuario
3. Envía mensajes desde una pestaña
4. Observa cómo los mensajes aparecen en todas las pestañas en tiempo real

---

## 📁 Estructura del Proyecto

```
chatws/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── org/example/chatws/
│   │   │       ├── ChatwsApplication.java       # Clase principal
│   │   │       ├── config/
│   │   │       │   └── WebSocketConfig.java      # Configuración WebSocket
│   │   │       ├── controller/
│   │   │       │   └── ChatController.java       # Controlador de mensajes
│   │   │       ├── entity/
│   │   │       │   └── ChatMessage.java          # Entidad de mensaje
│   │   │       └── utility/
│   │   │           ├── Constants.java           # Constantes del sistema
│   │   │           └── ChatBrowser.java          # Cliente de prueba
│   │   └── resources/
│   │       ├── application.yaml                  # Configuración Spring
│   │       └── static/
│   │           ├── index.html                    # Interfaz web
│   │           ├── script.js                     # Lógica cliente
│   │           └── style.css                     # Estilos
├── pom.xml                                        # Dependencias Maven
└── README.md                                      # Este archivo
```

---

## 💡 Conceptos Adicionales

### Patrón Publish-Subscribe
El chat implementa el patrón publish-subscribe donde:
- **Publisher:** El cliente que envía un mensaje
- **Subscriber:** Los clientes que se suscriben al topic
- **Topic:** El canal `/topic/public` que distribuye los mensajes

### Asincronía
La comunicación es completamente asíncrona:
- Los clientes no bloquean esperando respuestas
- Los mensajes se entregan cuando están disponibles
- Múltiples mensajes pueden estar en tránsito simultáneamente

### Escalabilidad (Consideraciones)
El broker RabbitMQ permite compartir mensajería STOMP entre instancias. La aplicación usa un relay STOMP local y las credenciales de desarrollo predeterminadas; para un despliegue real, configura credenciales y reglas de acceso apropiadas.

---

## 🔍 Referencias Útiles

- [Spring WebSocket Documentation](https://docs.spring.io/spring-framework/reference/web/websocket.html)
- [STOMP Protocol Specification](https://stomp.github.io/)
- [SockJS Documentation](https://github.com/sockjs/sockjs-client)
- [WebSocket MDN Guide](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)

---

## 📝 Notas para Alumnos

### Preguntas Frecuentes

**¿Por qué usar STOMP sobre WebSocket?**
WebSocket solo proporciona el canal de comunicación. STOMP define el protocolo para estructurar los mensajes, gestionar suscripciones y manejar errores, haciendo el desarrollo más sencillo.

**¿Qué pasa si un navegador no soporta WebSocket?**
SockJS detecta automáticamente la capacidad del navegador y usa transportes alternativos (como HTTP polling) si WebSocket no está disponible.

**¿Cómo se manejan múltiples usuarios?**
Cada conexión WebSocket es independiente. El broker mantiene registro de todas las suscripciones y distribuye los mensajes a los suscriptores correspondientes automáticamente.

**¿Por qué usar `/app` y `/topic`?**
- `/app`: Prefijo para mensajes que el cliente envía al servidor (para procesamiento)
- `/topic`: Prefijo para mensajes que el servidor envía a los clientes (para broadcast)
Esta separación clarifica la dirección del flujo de mensajes.

---

**¡Explora el código, experimenta con diferentes configuraciones y aprende cómo funciona la comunicación en tiempo real!** 🎓
