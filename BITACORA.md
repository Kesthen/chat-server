#  BITÁCORA TÉCNICA - TALLER SEMANA 15 (DOSW)
## WebSockets en Tiempo Real con Spring Boot & React + Observabilidad Integral

- **Estudiante:** Kevin
- **Asignatura:** Desarrollo de Operaciones de Software (DOSW) - Escuela Colombiana de Ingeniería
- **Fecha:** Noviembre 2026
- **Tema Seleccionado:** **Estación Espacial Orbital (DOSW Orbital Telemetry)**

---

##  Resumen Ejecutivo de la Solución

El presente proyecto implementa un ecosistema de comunicación bidireccional y reactivo de alta fidelidad:
1. **Backend (`chat-server`):** Arquitectura sobre Spring Boot 3.4.x, configuración de endpoints STOMP en `/ws`, interceptor de presencia para desconexiones (`SessionDisconnectEvent`), API REST para subida y persistencia de archivos (`/api/archivos`) y exportación de métricas de negocio con Micrometer.
2. **Frontend (`chat-client`):** Single Page Application en React + Vite con `@stomp/stompjs`, con temática visual espacial, selector de insignias/avatares, selector de emojis (`emoji-picker-react`), subida de imágenes, notas de voz con `MediaRecorder` y telemetría de tecleo en tiempo real.
3. **Observabilidad:** Pila completa integrada con Docker Compose: **Prometheus** (raspado cada 5s en `/actuator/prometheus`), **Loki** (recolección de logs con `traceId` y `spanId`), **Zipkin** (rastreo distribuido con Brave) y tablero interactivo en **Grafana** (`dashboard-chat.json`).

---

##  Registro Detallado de Pasos (1 a 16)

### PARTE 1: El Chat (WebSockets & STOMP)

#### Paso 1: Inicialización del Servidor (`chat-server`)
- Se configuró el `pom.xml` con `spring-boot-starter-web` y `spring-boot-starter-websocket`.
- Se estructuró el paquete base `com.chat` y la clase principal `ChatServerApplication.java`.
- **Verificación:** Compilación exitosa con Maven 3.9 y Java 21 LTS (`BUILD SUCCESS`).

#### Paso 2: Configuración del WebSocket (`WebSocketConfig.java`)
- Se habilitó `@EnableWebSocketMessageBroker`.
- Se registró el endpoint `/ws` con `setAllowedOriginPatterns("*")` para permitir conexiones CORS desde cualquier origen de desarrollo (`http://localhost:5173`).
- Se configuró el broker simple con prefijo `/topic` y el prefijo de aplicación `/app`.

#### Paso 3: Modelo de Mensajes y Controlador (`ChatMessage.java`, `ChatController.java`)
- Se implementó el record inmutable `ChatMessage(String tipo, String autor, String contenido, String fecha)`.
- Se definieron los métodos `@MessageMapping("/chat.unirse")` y `@MessageMapping("/chat.enviar")` con `@SendTo("/topic/sala")`.

#### Paso 4: Detección Automática de Desconexiones (`PresenciaListener.java`)
- Se creó un listener de eventos de Spring `@EventListener` sobre `SessionDisconnectEvent`.
- Al cerrarse el socket o la pestaña del navegador, se recupera el atributo `"usuario"` de la sesión STOMP y se emite un mensaje de tipo `SALIDA` hacia `/topic/sala`.

#### Paso 5: Inicialización del Cliente React (`chat-client`)
- Se configuró el proyecto con Vite y React 18.
- Se instalaron los paquetes `@stomp/stompjs` y `emoji-picker-react`.

#### Paso 6: Conexión STOMP y Componente Principal (`App.jsx`, `index.css`)
- Se implementó la clase `Client` de `@stomp/stompjs` apuntando a `ws://localhost:8080/ws`.
- Se enlazaron las suscripciones a `/topic/sala` y las publicaciones a `/app/chat.unirse` y `/app/chat.enviar`.
- Se implementó desplazamiento suave automático (`scrollIntoView`) al recibir nuevas transmisiones.

#### Paso 7: Comprobación del Handshake HTTP 101 y Multi-Usuario
- En las herramientas de desarrollador (DevTools -> Network -> WS), se valida la negociación:
  - **Petición HTTP:** `GET /ws HTTP/1.1` con cabeceras `Upgrade: websocket` y `Connection: Upgrade`.
  - **Respuesta Servidor:** `HTTP/1.1 101 Switching Protocols`.
  - **Frames STOMP:** `CONNECT`, `CONNECTED`, `SUBSCRIBE`, `SEND`, `MESSAGE`.

---

### PARTE 2: Identidad Visual y Funciones Dinámicas (+3 Puntos)

#### Paso 8: Tema "Estación Espacial Orbital"
- Se rediseñó la experiencia con una paleta cyberpunk/sci-fi: fondo estelar `#070913`, acentos cian eléctrico `#06b6d4`, violeta `#8b5cf6`, tipografías *Orbitron* y *Poppins*.
- Tarjeta de ingreso estilizada con selector de insignias de tripulante 
- Burbujas de chat diferenciadas con efectos de gradiente, avatar del autor y marcas de tiempo de misión.

#### Paso 9: Funcionalidades Dinámicas Implementadas
1. **Selector de Emojis:** Componente flotante `EmojiPicker` integrado en la barra de control para enriquecer las transmisiones de texto.
2. **Subida y Transmisión de Imágenes:** Controlador REST `ArchivoController.java` (`POST /api/archivos`). El archivo se almacena en el servidor y por WebSocket se difunde el mensaje con `tipo: "IMAGEN"` y la URL resultante. Al hacer clic, se abre en vista ampliada.
3. **Notas de Voz:** Grabación de audio en vivo usando `navigator.mediaDevices.getUserMedia` y `MediaRecorder`. Se envía el archivo de audio al servidor y se reproduce con reproductor HTML5 integrado en el chat.
4. **Indicador "Está redactando...":** Canal dedicado `/app/chat.escribiendo` -> `/topic/escribiendo` con temporizador de 2 segundos y animación de pulsos de telemetría.

---

### PARTE 3: Observabilidad Completa (Métricas, Trazas y Logs)

#### Paso 10: Métricas de Sistema con Actuator y Prometheus
- Se añadieron las dependencias `spring-boot-starter-actuator` y `micrometer-registry-prometheus`.
- Se expuso el endpoint `/actuator/prometheus` con distribución de percentiles para latencias HTTP.

#### Paso 11: Métricas de Negocio Personalizadas (`ChatMetrics.java`)
- Se crearon dos métricas específicas del negocio:
  - `chat_usuarios_conectados`: Métrica tipo **Gauge** mediante `AtomicInteger` (se incrementa en `unirse` y se decrementa en `alDesconectar`).
  - `chat_mensajes_total`: Métrica tipo **Counter** clasificada con etiqueta (`tipo="TEXTO|IMAGEN|AUDIO|UNION|SALIDA"`).

#### Paso 12: Trazabilidad Distribuida con Brave y Zipkin
- Se incorporaron las librerías `micrometer-tracing-bridge-brave` y `zipkin-reporter-brave`.
- Se configuró probabilidad de muestreo del 100% (`management.tracing.sampling.probability=1.0`) y destino en `http://localhost:9411/api/v2/spans`. Cada petición HTTP genera automáticamente identificadores `[chat-server,traceId,spanId]`.

#### Paso 13: Centralización de Logs en Loki (`logback-spring.xml`)
- Se configuró el appender `Loki4jAppender` apuntando a `http://localhost:3100/loki/api/v1/push`.
- Los logs incluyen el `traceId` y `spanId` para permitir la correlación cruzada con Zipkin.

#### Paso 14: Infraestructura Docker Compose (`observabilidad/`)
- Se creó `docker-compose.yml` que orquestará:
  - **Prometheus** (`:9090`): Raspando al servidor cada 5 segundos.
  - **Loki** (`:3100`): Ingesta de registros de log.
  - **Zipkin** (`:9411`): Servidor de trazas y spans.
  - **Grafana** (`:3000`): Aprovisionado automáticamente con los datasources de Prometheus, Loki y Zipkin (`datasources.yml`).

#### Paso 15: Tablero Integral de Grafana (`dashboard-chat.json`)
Se definieron y exportaron los 6 paneles clave solicitados:
1. `chat_usuarios_conectados` (Stat)
2. `sum by (tipo) (rate(chat_mensajes_total[1m])) * 60` (Time Series)
3. `histogram_quantile(0.95, sum by (le) (rate(http_server_requests_seconds_bucket{uri="/api/archivos"}[5m])))` (Time Series p95)
4. `sum(jvm_memory_used_bytes{application="chat-server", area="heap"})` (Time Series Heap)
5. `{app="chat-server"}` (Logs Loki)
6. `{app="chat-server", level="ERROR"}` (Logs Loki de sólo errores)

#### Paso 16: Validación y Correlación Cruzada
- Al subir una imagen o audio mediante `/api/archivos`, el servidor genera un log con `traceId`.
- El `traceId` puede copiarse y buscarse en la interfaz de Zipkin para ver la duración exacta de cada fase de la petición.
- En Grafana (Explore Loki), la consulta `{app="chat-server"} |= "<traceId>"` filtra exclusivamente los logs pertenecientes a esa transacción.

---

## 📷 Guía de Capturas para la Entrega

Para la sustentación ante el docente, incluye las siguientes capturas en la carpeta `evidencias/`:

| No. | Nombre de Captura | Qué debe mostrar |
| :--- | :--- | :--- |
| **01** | `01_handshake_ws_101.png` | DevTools (F12) -> pestaña Network -> filtro WS -> cabeceras `Upgrade: websocket` y estado `101 Switching Protocols`. |
| **02** | `02_frames_stomp.png` | Subpestaña **Messages** del WebSocket en DevTools mostrando las tramas STOMP (`CONNECT`, `SUBSCRIBE /topic/sala`, `SEND`, `MESSAGE`). |
| **03** | `03_chat_multiusuario_3pestanas.png` | Tres navegadores/pestañas en paralelo con diferentes avatares y usuarios chateando en tiempo real. |
| **04** | `04_selector_emojis_y_multimedia.png` | Interfaz mostrando el selector de emojis abierto y mensajes con imágenes y notas de voz transmitidas. |
| **05** | `05_aviso_desconexion.png` | Mensaje del sistema con insignia de desconexión (` Usuario salió del chat`) al cerrar una pestaña. |
| **06** | `06_actuator_prometheus.png` | Navegador en `http://localhost:8080/actuator/prometheus` mostrando las métricas `chat_mensajes_total` y `chat_usuarios_conectados`. |
| **07** | `07_docker_containers.png` | Terminal con `docker compose ps` o Docker Desktop mostrando Prometheus, Loki, Zipkin y Grafana activos. |
| **08** | `08_tablero_grafana.png` | Tablero de Grafana (`http://localhost:3000`) con los 6 paneles en verde y graficando tráfico real. |
| **09** | `09_traza_zipkin.png` | Interfaz de Zipkin (`http://localhost:9411`) mostrando el desglose del span de subida de archivo. |

