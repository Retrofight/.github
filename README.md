# RetroFight

RetroFight e' una piattaforma per matchmaking multiplayer 1v1 su giochi arcade e picchiaduro, costruita attorno a RetroArch + FBNeo, un client Electron e un backend Socket.IO.

L'obiettivo principale e' rendere il netplay accessibile senza chiedere agli utenti di aprire porte sul router. RetroArch resta inalterato: signaling, NAT traversal, hole punching, proxy locale e fallback relay vivono nel layer RetroFight.

## Repository

L'ecosistema RetroFight e' diviso in tre repository principali.

### `retrofight-server`

Backend Node.js + TypeScript + Socket.IO.

Responsabilita':

- lobby pubblica e lista player online;
- matchmaking 1v1;
- creazione e cleanup delle stanze;
- signaling netplay tra i due peer;
- scambio endpoint UDP/TCP;
- gestione stati stanza;
- allocazione relay TCP quando la connessione diretta fallisce.

Il server mantiene lo stato in memoria. Non usa ancora un database persistente.

Comandi principali:

```powershell
cd retrofight-server
npm install
npm start
```

Build e test:

```powershell
npm run build
npm test
```

Per test LAN/WAN con relay:

```powershell
$env:RETROFIGHT_RELAY_HOST="192.168.1.8"
npm start
```

### `retrofight-client`

Client Windows basato su Electron.

Responsabilita':

- UI launcher;
- connessione al server Socket.IO;
- flusso lobby e sfida 1v1;
- avvio di RetroArch con core FBNeo;
- training/offline mode;
- lifecycle del processo RetroArch;
- chiusura forzata del match se il peer esce;
- proxy locale UDP/TCP usato dal netplay RetroFight;
- probe UDP, report `netplay:punch_result` e switch TCP verso relay.

Il flusso principale e' `retrofight-client/electron-client/`. Il vecchio `launcher.js` e' legacy.

Comandi principali:

```powershell
cd retrofight-client/electron-client
npm install
npm run dev:local
```

Per test LAN, `RETROFIGHT_SERVER_URL` deve puntare all'IPv4 LAN del server, non a `localhost`:

```powershell
npm run dev
```

Per test proxy locale:

```powershell
npm run dev:local:proxy
```

Per diagnostica lunga LAN/proxy:

```powershell
npm run dev:custom:proxy:debug
```

### `retrofight-web`

Sito web e landing page del progetto, costruito con Next.js, React, TypeScript e Tailwind CSS.

Responsabilita':

- presentare RetroFight;
- spiegare matchmaking, netplay e obiettivo zero port forwarding;
- fornire entry point per download, documentazione e aggiornamenti;
- ospitare componenti interattivi come simulatori e sezioni FAQ.

Comandi principali:

```powershell
cd retrofight-web
npm install
npm run dev
```

Il server di sviluppo ascolta su:

```txt
http://localhost:3001
```

Build e lint:

```powershell
npm run build
npm run lint
```

## Architettura

RetroFight separa il netplay RetroArch dalla complessita' di rete moderna.

Flusso target:

1. Due client entrano in lobby.
2. Il server crea una stanza 1v1.
3. Ogni client avvia il proxy locale UDP/TCP.
4. Ogni client pubblica endpoint locali, diretti e candidati pubblici.
5. Il server scambia gli endpoint tra i peer.
6. I proxy eseguono UDP hole punching.
7. Se la connessione diretta funziona, il traffico passa peer-to-peer.
8. Se la connessione diretta fallisce, il traffico TCP passa da un relay.
9. RetroArch parla solo con endpoint locali/proxy e non conosce NAT, STUN o relay.

## Stati netplay

Il server usa questi stati stanza:

- `pending_signaling`: la stanza esiste, ma manca ancora l'endpoint di uno o entrambi i peer.
- `pending_punch`: entrambi i peer hanno pubblicato endpoint e stanno provando la connessione diretta.
- `direct_connected`: entrambi i peer hanno confermato il direct path.
- `relay_needed`: almeno un peer non ha confermato il direct path e serve il relay.
- `connected`: percorso relay confermato da entrambi i peer.

Eventi principali:

- `join_lobby`
- `create_room`
- `match_ready`
- `match_ended`
- `force_close_match`
- `netplay:endpoint_ready`
- `netplay:room_state`
- `netplay:peer_endpoint`
- `netplay:punch_result`
- `netplay:relay_endpoint`
- `netplay:relay_connected`

## Stato attuale

Gia' funzionante:

- matchmaking 1v1 in memoria;
- lobby Socket.IO;
- creazione stanze;
- avvio RetroArch da Electron;
- modalita training/offline;
- cleanup match e `force_close_match`;
- signaling endpoint UDP/TCP;
- UDP probe tra client;
- report client di `netplay:punch_result`;
- relay TCP server-side base;
- switch TCP client/proxy verso relay;
- forwarder TCP reale RetroArch guest -> proxy guest -> proxy host -> RetroArch host;
- smoke test same-machine con due istanze Electron/RetroArch;
- test same-LAN con due macchine e build packaged.

Ancora in corso:

- STUN o discovery equivalente dell'endpoint pubblico;
- validazione con due NAT reali;
- validazione relay con RetroArch reale su reti restrittive;
- UX completa per stati rete, errori e fallback;
- documentazione release beta e troubleshooting utente finale.

## Decisioni tecniche

- RetroArch, FBNeo e il netcode interno dell'emulatore non vengono modificati.
- Socket.IO e' il signaling broker.
- Il client Electron possiede il proxy locale UDP/TCP.
- Il proxy separa la porta locale RetroArch dalla porta peer-facing.
- Il direct path via hole punching e' preferito quando disponibile.
- Il relay e' il fallback per NAT simmetrici o reti restrittive.
- Il flusso RetroArch in produzione non deve cambiare verso `127.0.0.1` finche' il forwarder non e' validato in modo sufficiente.

## Requisiti di sviluppo

- Windows per il client Electron e runtime RetroArch.
- Node.js 18 o superiore.
- npm.
- RetroArch Windows e core FBNeo nella struttura attesa dal client.
- ROM di test sotto `retrofight-client/ROMs/`.

## Avvio rapido locale

Avviare il server:

```powershell
cd retrofight-server
npm install
npm start
```

Avviare il client Electron:

```powershell
cd retrofight-client/electron-client
npm install
npm run dev:local
```

Avviare il sito:

```powershell
cd retrofight-web
npm install
npm run dev
```

## Roadmap beta

La milestone 1.0.0 beta punta a una build distribuibile con:

- zip e installer Windows generati in modo ripetibile;
- runtime RetroArch/FBNeo pinato;
- match 1v1 LAN stabile da build packaged;
- test due NAT documentato;
- direct path quando possibile;
- fallback relay quando necessario;
- diagnostica attivabile via variabili d'ambiente;
- UX chiara per server non raggiungibile, peer uscito, RetroArch mancante, ROM/core mancanti e fallback rete.

## Licenza

I repository usano attualmente licenza ISC dove indicata nei package. Verificare e allineare i metadati di licenza prima di una release pubblica stabile.
