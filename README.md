# Wireshark Network Traffic & Protocol Analysis Lab

## Panoramica del Progetto
Questo laboratorio pratico è focalizzato sull'analisi del traffico di rete sui **Layer 3, 4 e 7 della pila ISO/OSI** mediante l'utilizzo dello sniffer di pacchetti **Wireshark**. 

L'obiettivo principale è analizzare le dinamiche di creazione e chiusura di una sessione TCP(Three-Way Handshake TCP), ispezionare il traffico in chiaro (HTTP/DNS) per identificare l'esposizione di dati sensibili (come credenziali di login) e applicare filtri di visualizzazione avanzati per l'analisi e la ricerca di anomalie.

---

## Topologia di Rete & Setup

L'analisi è stata condotta simulando e analizzando il traffico generato su un'interfaccia di rete locale(eth0) su VM Kali Linux(Loopback/LAN).

```
+-------------------------------------------------------------+
|                           VM(Kali)                          |
|                                                             |
|  +------------------+             +----------------------+  |
|  |  cURL Client     |   (HTTP)    |  HTTP Web Server     |  |
|  |  (User-Agent)    | ----------> |  (Python / Apache)   |  |
|  +------------------+             +----------------------+  |
|            |                                 |              |
|            +-----------------+---------------+              |
|                              |                              |
|                    [ Network Interface ]                    |
|                              |                              |
|                     [ Wireshark Packet ]                    |
|                     [  Capture Engine  ]                    |
+-------------------------------------------------------------+
```

### Strumenti Utilizzati
* **Packet Analyzer:** Wireshark 4.6.6
* **Traffic Generator:** `curl` CLI / Browser(Firefox)
* **Web Server di Test:** Localhost Python HTTP Server (`127.0.0.1:8080`)

---

## Fondamenti Teorici: TCP Three-Way Handshake

Il protocollo **TCP (Transmission Control Protocol)** è un protocollo **Layer 4** di tipo *connection-oriented*. Questo vuol dire che prima che due host possano scambiarsi dati a livello applicativo (es. HTTP), devono stabilire un canale di comunicazione affidabile tramite la procedura di **Three-Way Handshake**.

```
    Client (Initiator)                           Server (Listener)
            |                                           |
            | ------------ 1. [SYN]  -----------------> |  (Client richiede sincronizzazione)
            |                                           |
            | <---------- 2. [SYN, ACK]   ------------- |  (Server conferma e richiede)
            |                                           |
            |                                           |
            | ------------- 3. [ACK] -----------------> |  (Client conferma l'apertura)
            |                                           |
            |                                           |
   [ESTABLISHED]                               [ESTABLISHED]
            |                                           |
            | === SCAMBIO DATI APPLICATIVI (HTTP) ===== |
```

### I 3 Passaggi nel Dettaglio:
1. **`SYN` (Synchronize):** Il client invia un pacchetto con il flag `SYN` attivo e un numero di sequenza casuale iniziale per richiedere l'apertura di una sessione.
2. **`SYN-ACK` (Synchronize-Acknowledge):** Il server risponde con il flag `SYN` per la propria sincronizzazione e il flag `ACK` per confermare la ricezione del pacchetto del client.
3. **`ACK` (Acknowledge):** Il client conferma la ricezione del `SYN` del server inviando un ultimo pacchetto con flag `ACK`.La connessione passa allo stato **ESTABLISHED**.

### Chiusura della Sessione:
La chiusura avviene in 4 fasi mediante i flag **`FIN` (Finish)** ed **`ACK`**, garantendo che tutti i pacchetti pendenti vengano consegnati prima di interrompere il canale.

---

## Esecuzione dei Test & Analisi dei Flussi

### 1. Avvio Wireshark e apertura Server
Avviamo tre terminali
Nel 1° inizializziamo il server Python localhost:

```bash
   python3 -m http.server 8080
```
Nel 2° avviamo Wireshark e una volta aperto selezioniamo l'interfaccia **Localhost(LO)** e avviamo la cattura dei pacchetti:

```bash
    sudo wireshark
```
Nel 3° invece il comando CURL per inviare richieste HTTP al server locale.

### 2. Intercettazione delle Credenziali in Chiaro (HTTP POST)
Inviando una richiesta di login HTTP senza cifratura TLS/SSL (porta 80/8080), l'intero corpo della richiesta e gli header di trasporto viaggiano in chiaro.

**Comando di test eseguito:**
```bash
curl -v -X POST http://127.0.0.1:8080/login \
  -H "User-Agent: Mozilla/5.0 (X11; Linux x86_64)" \
  -d 'uname=admin&pass=Password123!'
```

**Analisi del Flusso Estratto (Follow -> TCP Stream):**
HTTP
```bash
POST /login HTTP/1.1
Host: 127.0.0.1:8080
Accept: */*
User-Agent: Mozilla/5.0 (X11; Linux x86_64)
Content-Length: 29
Content-Type: application/x-www-form-urlencoded

uname=admin&pass=Password123!
```

> **Security Risk Note:** L'assenza di cifratura consente ad un attaccante posizionato sullo stesso segmento di rete (tramite Man-in-the-Middle o ARP Spoofing) di estrarre credenziali ad esempio di login(`uname=admin`, `pass=Password123!`) senza nessuna operazione di decifrazione.

---

## 🔍 Filtri Wireshark Utilizzati (Display Filters Reference)

Durante il lab sono stati utilizzati i seguenti filtri per isolare il traffico rilevante:

| Filtro Wireshark | Scopo / Descrizione |
| :--- | :--- |
| `http \|\| dns` | Isola esclusivamente il traffico Web non cifrato e di risoluzione nomi . |
| `http.request.method == "POST"` | Filtra solo gli invii di dati con riechieste POST (es. login). |
| `tcp.flags.syn == 1 and tcp.flags.ack == 0` | Identifica i tentativi di avvio connessione TCP (utile per rilevare SYN Flood(DoS)). |
| `tcp.port == 8080` | Mostra l'intero traffico TCP che passa per la porta 8080(server locale). |
---

## 📂 Struttura del Repository

```text
.
├── README.md                    # Documentazione del progetto
├── captures/                    # File PCAP con le tracce catturate
│   ├── tcp_handshake.pcapng     # Cattura del 3-way handshake
│   └── http_post_login.pcapng   # Cattura con dati in chiaro
└── screenshots/                 # Prove visive dell'analisi
    ├── tcp_stream.png           # Screenshot del Follow TCP Stream
    └── wireshark_filters.png    # Screenshot dei filtri applicati
```

---

## 🎯 Competenze Acquisite (Key Takeaways)
* Padronanza della sintassi dei filtri di visualizzazione di Wireshark.
* Comprensione approfondita delle dinamiche di rete Layer 4 (TCP Flags e Sequence Numbers).
* Capacità di orientarsi nell'interfaccia Wireshark e utilizzo comando cURL.
* Consapevolezza dei rischi di sicurezza legati all'impiego di protocolli privi di cifratura (HTTP vs HTTPS).
