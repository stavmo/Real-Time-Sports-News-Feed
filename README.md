# World Cup 2026 Informer — Real-Time Sports News Feed

A distributed, community-led sports news and match-reporting service implemented using the **STOMP 1.2** (Simple Text Oriented Messaging Protocol) network protocol.

The platform enables users to connect to a centralized messaging server, subscribe to live game channels, report live game events from parsed JSON files, receive real-time match events from other connected clients, and generate localized game stats summaries.

---

## Architecture Overview

## Architecture Overview

The system consists of a multi-threaded C++ client communicating over standard STOMP wire frames with a high-performance Java STOMP server backed by an SQLite persistence service:

```text
       ┌────────────────────────┐
       │     C++ STOMP Client   │  (Multi-threaded: Stdin Reader + Socket Listener)
       └───────────┬────────────┘
                   │ STOMP 1.2 Frames (\0-terminated)
                   ▼
       ┌────────────────────────┐
       │    Java STOMP Server   │  (Thread-Per-Client or Non-Blocking Reactor)
       └───────────┬────────────┘
                   │ SQL Query Strings
                   ▼
       ┌────────────────────────┐
       │   Python SQLite Server │  (User authentication, login/logout timestamps, uploads)
       └────────────────────────┘
```


### 1. Java STOMP Server
- **Dual Server Architectures:** Supports runtime selection between **Thread-Per-Client (TPC)** and non-blocking **Reactor** (Java NIO) patterns.
- **STOMP Protocol Engine:** Implements the `StompMessagingProtocol` and `Connections<T>` interfaces to manage active connections, distribute topic subscriptions, and broadcast frames across clients.
- **Database Tracking:** Communicates with a dedicated Python SQLite backend to persist user registration, login/logout history, and uploaded match event files.

### 2. C++ Multi-Threaded Client (`StompWCIClient`)
- **Dual-Threaded Design:** One thread continuously polls user input from standard input (`stdin`), while a separate worker thread listens to the network socket to handle incoming asynchronous frames.
- **Event File Parser:** Ingests match event JSON files (using `Event.cpp`/`Event.h`) and translates structured event maps into STOMP `SEND` frames.
- **Match Aggregation & Summaries:** Tracks incoming updates per game channel and user, generating lexicographically sorted statistics and chronological match summaries.

---

## STOMP 1.2 Protocol Specification

All frames transmitted across the client-server boundary are null-byte (`\0`) terminated and formatted per STOMP specifications:

* **Client Frames:**
  * `CONNECT` — Initiates connection with version negotiation and authentication.
  * `SUBSCRIBE` — Registers a subscription to a specific game topic (e.g., `/usa_mexico`).
  * `UNSUBSCRIBE` — Removes an active subscription by subscription ID.
  * `SEND` — Dispatches a match event payload to all clients subscribed to the topic.
  * `DISCONNECT` — Gracefully terminates client session by requesting a receipt acknowledgment.
* **Server Frames:**
  * `CONNECTED` — Acknowledges successful connection and credentials.
  * `MESSAGE` — Delivers a broadcast message to subscribed clients with corresponding subscription and message IDs.
  * `RECEIPT` — Confirms receipt and processing of frames requesting a `receipt` header.
  * `ERROR` — Indicates protocol violations, auth failures, or malformed frames, closing the underlying connection.

---

## Supported Client CLI Commands

| Command | Syntax | Description |
|---|---|---|
| **Login** | `login {host:port} {username} {password}` | Connects to the server, registering new users or authenticating existing credentials. |
| **Join Channel** | `join {game_name}` | Subscribes the client to updates for `{game_name}` (e.g., `usa_mexico`). |
| **Exit Channel** | `exit {game_name}` | Unsubscribes the client from the specified game channel. |
| **Report** | `report {file}` | Parses the given JSON event file and streams the individual events as `SEND` frames. |
| **Summary** | `summary {game_name} {user} {file}` | Exports an aggregated report of game stats and chronological events reported by `{user}` to `{file}`. |
| **Logout** | `logout` | Unsubscribes from all channels, dispatches a `DISCONNECT` frame with a receipt check, and closes the socket. |

---

## Building & Running

### Prerequisites
- **Java:** JDK 11+ and Apache Maven
- **C++:** GCC / G++ (C++11 or higher) and GNU Make
- **Python:** Python 3.x with `sqlite3`

---

### Step 1: Start the Python SQLite Database Server
The database service executes relational queries for user activity and file tracking:
```bash
python3 sql_server.py <sql_port>
```
---

### Step 2: Build and Run the Java STOMP Server
Navigate to the server directory containing pom.xml[cite: 11]:

    cd server
    mvn compile[cite: 11]

Run with Thread-Per-Client (TPC) Model[cite: 11]:

    mvn exec:java -Dexec.mainClass="bgu.spl.net.impl.stomp.StompServer" -Dexec.args="<port> tpc"[cite: 11]

Run with Reactor Model (Java NIO)[cite: 11]:

    mvn exec:java -Dexec.mainClass="bgu.spl.net.impl.stomp.StompServer" -Dexec.args="<port> reactor"[cite: 11]

---

### Step 3: Build and Run the C++ Client
Navigate to the client directory and compile using make[cite: 8, 13]:

    cd client
    make

Launch the executable from the bin directory[cite: 13]:

    ./bin/StompWCIClient[cite: 13]

Example Interactive Session:

    login 127.0.0.1:7777 meni films[cite: 14]
    join usa_mexico[cite: 15]
    report data/events1_partial.json[cite: 17, 20]
    summary usa_mexico meni summary_output.txt[cite: 18]
    logout[cite: 19]

---

## 📁 Project Structure

```text
.
├── server/                         # Java STOMP Server
│   ├── src/main/java/bgu/spl/net/
│   │   ├── api/                    # Protocol and Encoder/Decoder interfaces
│   │   ├── impl/stomp/             # StompServer main entry point
│   │   └── srv/                    # Connections, TPC, and Reactor implementations
│   └── pom.xml                     # Maven build descriptor
│
├── client/                         # C++ Multi-Threaded STOMP Client
│   ├── include/                    # Header files (ConnectionHandler, Event, StompProtocol)
│   ├── src/                        # Implementation files and event parser
│   ├── bin/                        # Compiled executable output (StompWCIClient)
│   ├── data/                       # Sample match JSON files (e.g., events1_partial.json)
│   └── makefile                    # C++ compilation script
│
├── sql_server.py                   # Python SQLite persistence server
└── README.md
