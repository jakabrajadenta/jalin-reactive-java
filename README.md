# Reactive Java — ISO 8583 Socket Gateway

A Spring Boot application that acts as a dynamic ISO 8583 TCP gateway using Netty. It can start as either a TCP **client** (connecting to a remote host) or a TCP **server** (listening for incoming connections), managed at runtime via REST API. Received ISO 8583 messages are parsed using jPOS and forwarded to a downstream HTTP service using Spring WebFlux (reactive WebClient).

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                  Spring Boot Application                 │
│                                                          │
│  REST API  ──►  IsoConnectionService                     │
│                    ├── IsoNettyClient  (TCP connect-out) │
│                    └── IsoNettyServer  (TCP listen)      │
│                           │                              │
│                    IsoNettyHandler (Netty pipeline)      │
│                           │                              │
│                    ProcessIsoThread (per-message thread) │
│                      ├── jPOS GenericPackager            │
│                      │   (ISO 8583 parse/pack)           │
│                      └── RouteRestService                │
│                          (WebClient → downstream HTTP)   │
└──────────────────────────────────────────────────────────┘
```

**Message flow:**
1. An ISO 8583 message arrives over TCP.
2. Netty decodes the length-prefixed frame and passes it to `IsoNettyHandler`.
3. A dedicated thread (`ProcessIsoThread`) parses the MTI to determine if the message is a request or response.
4. Request messages are forwarded via reactive `WebClient` (POST) to `http://localhost:3000/transaction/iso-raw`.
5. The response from the downstream service is written back over the same TCP channel.

---

## Tech Stack

| Component | Library / Version |
|---|---|
| Runtime | Java 17 |
| Framework | Spring Boot 3.3.2 |
| Reactive HTTP | Spring WebFlux (WebClient) |
| TCP / NIO | Netty 4.1.100.Final |
| ISO 8583 | jPOS 2.1.8 |
| Distributed Tracing | Micrometer Tracing 1.3.2 + Brave bridge |
| Build | Maven (Maven Wrapper included) |
| Utilities | Lombok, ModelMapper |

---

## Prerequisites

- Java 17+
- Maven 3.8+ (or use the included `mvnw` wrapper)
- A downstream HTTP service running on `http://localhost:3000` that accepts `POST /transaction/iso-raw` with a raw ISO 8583 string body (required only when processing inbound request messages)

---

## Installation

```bash
# Clone the repository
git clone https://github.com/your-org/jalin-reactive-java.git
cd jalin-reactive-java

# Build (skip tests for a quick start)
./mvnw clean package -DskipTests
```

On Windows use `mvnw.cmd` instead:
```cmd
mvnw.cmd clean package -DskipTests
```

---

## Running the Application

```bash
# Run directly with Maven
./mvnw spring-boot:run

# Or run the packaged JAR
java -jar target/rxjava-0.0.1-SNAPSHOT.jar
```

The application starts on **port 8080** (Spring Boot default).

---

## Configuration

Edit `src/main/resources/application.properties` to customize:

```properties
# Application name
spring.application.name=Rx Java with Socket Connection

# Change the HTTP server port (optional)
server.port=8080

# Downstream service URL that receives forwarded ISO 8583 requests
# Default (hardcoded in RouteRestService): http://localhost:3000/transaction/iso-raw
```

**WebClient timeouts (configured in `WebClientConfiguration`):**

| Setting | Value |
|---|---|
| Connect timeout | 5 000 ms |
| Response timeout | 30 000 ms |
| Read timeout | 30 000 ms |
| Write timeout | 30 000 ms |

**ISO 8583 packager:** configured via `src/main/resources/iso87ascii.xml` (ISO 8583-1987 ASCII format).

---

## REST API

### 1. Create a Socket Connection

**`POST /iso-connection/connect`**

Dynamically opens a new TCP socket — either as a client (connecting out to a remote host) or as a server (listening on a local port).

**Request body:**

```json
{
  "connectionName": "my-connection",
  "ip": "192.168.1.10",
  "port": "8585",
  "clientServer": "SERVER"
}
```

| Field | Type | Description |
|---|---|---|
| `connectionName` | String | Unique name to identify this connection in the connection manager |
| `ip` | String | Remote host IP — required when `clientServer = "SERVER"` |
| `port` | String | TCP port to connect to or listen on |
| `clientServer` | String | `"SERVER"` → connect **out** to a remote host · `"CLIENT"` → listen for incoming connections |

> **Note on `clientServer` values:**
> - `"SERVER"` means this app acts as the TCP **client** connecting to a remote server at the given `ip:port`.
> - `"CLIENT"` means this app starts a TCP **server** listening on the given `port`.

**Response (HTTP 200):**

```json
{
  "connectionName": "my-connection",
  "ip": "192.168.1.10",
  "port": "8585",
  "clientServer": "SERVER",
  "responseCode": "00",
  "responseMessage": "Success"
}
```

---

### 2. Echo / Health Check

**`GET /transaction/echo`**

Simple health check — returns a success response. Useful for verifying the service is up.

**Response (HTTP 200):**

```json
{
  "responseCode": "00",
  "responseMessage": "Success"
}
```

---

## Usage Examples (curl)

**Start a TCP server listening on port 9090:**
```bash
curl -X POST http://localhost:8080/iso-connection/connect \
  -H "Content-Type: application/json" \
  -d '{
    "connectionName": "server-9090",
    "port": "9090",
    "clientServer": "CLIENT"
  }'
```

**Connect to a remote ISO host at 192.168.1.10:8583:**
```bash
curl -X POST http://localhost:8080/iso-connection/connect \
  -H "Content-Type: application/json" \
  -d '{
    "connectionName": "remote-host",
    "ip": "192.168.1.10",
    "port": "8583",
    "clientServer": "SERVER"
  }'
```

**Health check:**
```bash
curl http://localhost:8080/transaction/echo
```

---

## Supported ISO 8583 Message Types (MTI)

| MTI | Description |
|---|---|
| `0100` | Authorization Request |
| `0110` | Authorization Response |
| `0200` | Financial Transaction Request |
| `0210` | Financial Transaction Response |
| `0420` | Reversal Request |
| `0421` | Reversal Repeat Request |
| `0430` | Reversal Response |
| `0800` | Network Management (Echo / Sign-On / Sign-Off) |
| `0810` | Network Management Response |
| `9xxx` | Error variants of the above request types |

**Processing logic:**
- **Request MTIs** (`0100`, `0200`, `0420`, `0421`, `0800`) → forwarded to the downstream HTTP service; the HTTP response is sent back over TCP.
- **Response MTIs** (`0110`, `0210`, `0430`, `0810`, `9xxx`) → logged only; no further action.

---

## Project Structure

```
src/main/java/com/braja/rxjava/
├── RxJavaWithSocketConnectionApplication.java   # Entry point
├── configuration/
│   ├── GenericPackagerConfiguration.java        # jPOS ISO 8583 packager bean
│   ├── IsoConnectionConfiguration.java          # ConcurrentHashMap connection registry
│   ├── MapperConfiguration.java                 # ModelMapper bean
│   ├── WebClientConfiguration.java              # Reactive WebClient with timeouts
│   └── handler/
│       ├── IsoNettyHandler.java                 # Netty inbound handler (reads ISO frames)
│       └── WebClientLogHandler.java             # HTTP request/response logging
├── controller/
│   ├── IsoConnectionController.java             # POST /iso-connection/connect
│   └── TransactionController.java               # GET /transaction/echo
├── dto/
│   ├── IsoConnectionDto.java                    # Connection request/response model
│   └── EchoDto.java                             # Echo response model
├── service/
│   ├── IsoConnectionService.java               # Creates Netty client or server
│   ├── RouteRestService.java                   # Forwards ISO message via WebClient
│   ├── TransactionService.java                 # Echo logic
│   └── model/
│       ├── IsoNettyClient.java                 # Netty TCP client (connect-out)
│       ├── IsoNettyServer.java                 # Netty TCP server (listen)
│       └── ChannelServerHandler.java           # Registers channel in connection manager
├── thread/
│   └── ProcessIsoThread.java                   # Per-message processing thread (parse + route)
└── util/constant/
    ├── IsoConstant.java                         # MTI constants, frame field offsets
    └── IsoDataElement.java                      # ISO 8583 data element definitions

src/main/resources/
├── application.properties
└── iso87ascii.xml                               # ISO 8583-1987 ASCII packager definition
```

---

## Actuator Endpoints

Spring Boot Actuator is included. Default endpoints are available at `/actuator`:

```bash
curl http://localhost:8080/actuator/health
```

---

## License

This project is for learning and demonstration purposes.
