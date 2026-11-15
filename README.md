# 🚀 Chat Server - Estación Espacial Orbital (DOSW Semana 15)

Servidor en tiempo real construido con **Spring Boot 3.4.x**, **WebSockets**, protocolo **STOMP** y observabilidad integrada de nivel empresarial (**Prometheus**, **Loki**, **Zipkin** y **Grafana**).

---

## 🛰️ 1. Descripción y Arquitectura

El servidor actúa como broker de mensajería y punto de enlace para múltiples tripulantes espaciales:
- **Protocolo de transporte:** WebSocket nativo (RFC 6455) con handshake HTTP `101 Switching Protocols` en el endpoint `/ws`.
- **Protocolo de aplicación:** **STOMP 1.2** sobre WebSocket para enrutamiento publish/subscribe a destinos específicos.
- **Manejo de archivos:** Controlador REST (`/api/archivos`) con soporte para subida de transmisiones visuales y notas de voz codificadas.
- **Pila de observabilidad:** Métricas con Micrometer + Prometheus, trazas con Brave + Zipkin y registros distribuidos con Loki4j + Loki.

```
                              ┌────────────────────────────────────────┐
                              │          chat-server (Spring)          │
[chat-client 1] ───STOMP───▶  │  /ws (Handshake 101)                   │  ──Métricas──▶ Prometheus (:9090)
[chat-client 2] ◀──STOMP───   │  /app/chat.enviar ──▶ /topic/sala      │  ──Trazas────▶ Zipkin (:9411)
[chat-client 3] ───HTTP────▶  │  /api/archivos (Upload imágenes/audio) │  ──Logs──────▶ Loki (:3100)
                              └────────────────────────────────────────┘
                                                  │
                                                  ▼
                                       Grafana Dashboard (:3000)
```

---

## 📡 2. Destinos STOMP y Endpoints

### Destinos STOMP
| Dirección / Canal | Tipo | Propósito | Formato / Contenido |
| :--- | :--- | :--- | :--- |
| `/ws` | Endpoint WebSocket | Handshake inicial HTTP -> WebSocket | Cabeceras Upgrade: websocket |
| `/app/chat.unirse` | Destino Envío (Cliente ➔ Servidor) | Registrar ingreso del tripulante en sesión | `ChatMessage(tipo="UNION", autor="...")` |
| `/app/chat.enviar` | Destino Envío (Cliente ➔ Servidor) | Difusión de mensajes a la sala | `ChatMessage(tipo, autor, contenido, fecha)` |
| `/app/chat.escribiendo`| Destino Envío (Cliente ➔ Servidor) | Notificar que el usuario está escribiendo | String con nombre de usuario |
| `/topic/sala` | Destino Suscripción (Servidor ➔ Clientes) | Recepción de transmisiones globales | Objeto `ChatMessage` serializado en JSON |
| `/topic/escribiendo` | Destino Suscripción (Servidor ➔ Clientes) | Transmisión de evento de tecleo | Nombre del usuario redactando |

### Tipos de Mensajes Soportados
- `TEXTO`: Transmisión de mensaje escrito con emojis y formato.
- `IMAGEN`: URL relativa al recurso gráfico subido en `/api/archivos`.
- `AUDIO`: URL relativa al registro de audio grabado en tiempo real.
- `UNION`: Notificación del sistema ante conexión de un nuevo tripulante.
- `SALIDA`: Notificación automática interceptada por `SessionDisconnectEvent`.

---

## 🛠️ 3. Ejecución en Local (Paso a Paso)

### Prerrequisitos
- **Java 21** LTS instalado (`java -version`).
- **Apache Maven 3.9+** (`mvn -version`).
- **Docker Desktop** activo para el stack de observabilidad.

### Pasos para iniciar el servidor
```bash
# 1. Clonar el repositorio
git clone https://github.com/Kesthen/chat-server.git
cd chat-server

# 2. Compilar el proyecto
mvn clean compile

# 3. Ejecutar la aplicación Spring Boot
mvn spring-boot:run
```
El servidor estará escuchando en `http://localhost:8080`.

---

## 🔭 4. Observabilidad y Monitoreo

### Levantar el Stack con Docker
En la carpeta `observabilidad/`:
```bash
cd observabilidad
docker compose up -d
```

### Servicios expuestos:
- **Prometheus:** `http://localhost:9090` (Scraping del servidor en `host.docker.internal:8080/actuator/prometheus`).
- **Zipkin:** `http://localhost:9411` (Visualización de trazas y spans distribuidos).
- **Loki:** `http://localhost:3100` (Recepción de logs estructurados con `traceId` y `spanId`).
- **Grafana:** `http://localhost:3000` (Credenciales por defecto: `admin` / `admin`).

### Tablero en Grafana
Importa el tablero preconfigurado ubicado en:
`observabilidad/dashboard-chat.json`

**Paneles disponibles en el Dashboard:**
1. **Usuarios conectados:** Métrica en tiempo real (`chat_usuarios_conectados`) mediante Gauge.
2. **Mensajes por minuto, por tipo:** Tasa de tráfico clasificada (`rate(chat_mensajes_total[1m]) * 60`).
3. **Latencia p95 de subida de archivos:** Percentil 95 en `/api/archivos`.
4. **Memoria Heap JVM:** Consumo de memoria en vivo (`jvm_memory_used_bytes`).
5. **Logs del chat:** Registros estructurados en Loki (`{app="chat-server"}`).
6. **Solo errores:** Filtrado en tiempo real de excepciones y alertas (`level="ERROR"`).

---

## 🧠 5. Preguntas de Análisis y Reflexión

1. **¿Por qué un chat con HTTP normal tendría que estar preguntándole al servidor cada segundo, y qué cambia con WebSocket?**
   - *Respuesta:* HTTP es un protocolo "request-response" unidireccional y sin estado donde la conexión TCP se cierra al finalizar cada respuesta. Para emular tiempo real con HTTP se requiere *polling*, provocando un consumo masivo de ancho de banda y latencia innecesaria. Con **WebSocket**, tras un único handshake HTTP (101 Switching Protocols), la conexión TCP se mantiene abierta de forma bidireccional y persistente (*full-duplex*), permitiendo al servidor empujar datos (*server push*) sin costo de reconexión.

2. **¿Por qué las imágenes y los audios se suben por HTTP y no viajan dentro del WebSocket?**
   - *Respuesta:* WebSocket está optimizado para tramas pequeñas y de baja latencia (señalización y control en tiempo real). Enviar binarios pesados o Base64 por el WebSocket congestionaría el broker de mensajes, aumentaría la latencia de todos los clientes conectados y consumiría memoria en el canal de eventos. Subirlos por HTTP multipart aprovecha el streaming eficiente del servidor web y por WebSocket solo se envía la URL liviana.

3. **Si el servidor se reinicia, ¿qué pasa con los usuarios conectados y qué hace `reconnectDelay`?**
   - *Respuesta:* Las sesiones TCP activas se interrumpen forzosamente y el estado volátil del broker se reinicia. En el cliente, la directiva `reconnectDelay: 3000` de `@stomp/stompjs` reintenta automáticamente restablecer la conexión cada 3 segundos sin que el usuario tenga que recargar la página.

4. **¿Qué métrica te avisaría primero de que el chat está fallando, y qué alerta pondrías sobre ella?**
   - *Respuesta:* La tasa de errores HTTP 5xx (`http_server_requests_seconds_count{status=~"5.."}`) o la caída abrupta de mensajes procesados con conexiones activas. Se configuraría una alerta en Grafana: `rate(http_server_requests_seconds_count{status=~"5.."}[1m]) > 0.05` durante más de 30 segundos.

5. **En Style Radar, ¿qué servicio se beneficiaría de WebSockets y qué tablero le armarías a cada microservicio?**
   - *Respuesta:* El microservicio de notificaciones de radar de tendencias y recomendaciones en vivo. Se estructuraría un tablero con: (1) Clientes conectados por región, (2) Mensajes de radar emitidos por minuto, (3) Latencia p95 de propagación de eventos, y (4) Tasa de errores por microservicio.
