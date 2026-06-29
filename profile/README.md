# RetroFight — System Architecture

## Repositories

| Repo | Tech | Description |
|------|------|-------------|
| `retrofight-server` | Node.js + Socket.IO + TypeScript | Signaling broker, matchmaking, UDP relay, ranking, match history |
| `retrofight-client` | Electron + React/Vite (CommonJS) | Desktop app — Windows & Linux |
| `retrofight-fbneo` | C++ (Win32) | Custom FinalBurn Neo with GGPO UDP netplay |
| `retrofight-web` | Next.js + Supabase | Landing page, auth, profiles, match history, leaderboards |
| `retrofight-releases` | — | Public release artifacts, release notes, wiki |
| `retrofight-legal` | — | Terms of Use, Privacy Policy, disclaimers |

---

## System Overview

```mermaid
flowchart TB
    subgraph USERS["Users"]
        WU["Web User\n(Browser)"]
        PA["Player A\n(Windows / Linux)"]
        PB["Player B\n(Windows / Linux)"]
    end

    subgraph CLOUD["Cloud"]
        subgraph VERCEL["Vercel"]
            WEB["retrofight-web\nNext.js"]
        end

        subgraph SUPABASE["Supabase"]
            SAUTH["Auth\n(JWT / Sessions)"]
            SDB[("Database\nprofiles · match_history\nplayer_game_ratings\nranking_history\nranking_seasons")]
            SSTORE["Storage\n(runtime artifacts)"]
        end

        subgraph SRVHOST["Server Host"]
            SRV["retrofight-server\nNode.js + Socket.IO"]
            RELAY["UDP Relay Pool\n(udpRelayServer.ts)\nports 40000–41000"]
        end
    end

    subgraph NET["Network"]
        STUN["STUN Servers\n(public IP/port discovery)"]
    end

    subgraph CLIENTA["Player A — retrofight-client (Electron)"]
        ELEA["Main Process\n(matchFlowState)"]
        FBNEA["retrofightfbneo.exe\nFBNeo + GGPO"]
    end

    subgraph CLIENTB["Player B — retrofight-client (Electron)"]
        ELEB["Main Process\n(matchFlowState)"]
        FBNEB["retrofightfbneo.exe\nFBNeo + GGPO"]
    end

    %% Web flows
    WU --> WEB
    WEB -->|"register / login"| SAUTH
    WEB -->|"profiles · match history\nleaderboards"| SDB

    %% Client auth & runtime download
    PA --> ELEA
    PB --> ELEB
    ELEA -->|"login (JWT)"| SAUTH
    ELEB -->|"login (JWT)"| SAUTH
    ELEA -->|"download exe"| SSTORE
    ELEB -->|"download exe"| SSTORE

    %% Server ↔ Supabase
    SRV -->|"verify JWT"| SAUTH
    SRV -->|"persist match results\n& rating updates"| SDB
    SRV -->|"serve runtime artifacts"| SSTORE

    %% Signaling
    ELEA <-->|"Socket.IO\n(lobby · matchmaking · signaling)"| SRV
    ELEB <-->|"Socket.IO\n(lobby · matchmaking · signaling)"| SRV

    %% STUN
    ELEA -->|"UDP preflight\n(get public IP:port)"| STUN
    ELEB -->|"UDP preflight\n(get public IP:port)"| STUN

    %% Runtime spawn
    ELEA -->|"spawn\n(quark:direct, AES session)"| FBNEA
    ELEB -->|"spawn\n(quark:direct, AES session)"| FBNEB

    %% GGPO — happy path
    FBNEA <-->|"GGPO UDP Direct\n(peer-to-peer)"| FBNEB

    %% Relay fallback
    FBNEA <-->|"GGPO via UDP Relay\n(fallback if punch fails)"| RELAY
    FBNEB <-->|"GGPO via UDP Relay\n(fallback if punch fails)"| RELAY
```

---

## Netplay Match Flow

The server first attempts UDP direct (hole-punching). If punch fails, it falls back to a server-side UDP relay — GGPO is agnostic to the transport as long as the port numbers match.

```mermaid
sequenceDiagram
    participant A as Player A (Client)
    participant STUN as STUN Server
    participant SRV as retrofight-server
    participant B as Player B (Client)

    A->>SRV: join_lobby (ROM driver filter)
    B->>SRV: join_lobby (ROM driver filter)
    SRV-->>A: room created
    SRV-->>B: join room → match_ready

    A->>STUN: UDP preflight probe (LAN + public candidates)
    B->>STUN: UDP preflight probe (LAN + public candidates)

    A->>SRV: netplay:endpoint_ready (candidate list)
    B->>SRV: netplay:endpoint_ready (candidate list)

    SRV-->>A: netplay:peer_endpoint (B's candidates)
    SRV-->>B: netplay:peer_endpoint (A's candidates)

    Note over A,B: Candidate selection — same public IP → LAN, else WAN

    A->>B: UDP preflight punch probe
    B->>A: UDP preflight punch probe

    alt Punch succeeds (both probes arrive)
        A->>SRV: netplay:punch_result (direct confirmed)
        B->>SRV: netplay:punch_result (direct confirmed)
        A->>A: spawn retrofightfbneo.exe (quark:direct peer endpoint)
        B->>B: spawn retrofightfbneo.exe (quark:direct peer endpoint)
        loop GGPO real-time netplay
            A<<->>B: GGPO UDP Direct (peer-to-peer)
        end
    else Punch fails
        SRV->>SRV: allocate relay pair from UDP pool\n(portA for A, portB for B)
        SRV-->>A: RELAY_ASSIGNED (relayIp, portA)
        SRV-->>B: RELAY_ASSIGNED (relayIp, portB)
        A->>A: spawn retrofightfbneo.exe (quark:direct relay endpoint)
        B->>B: spawn retrofightfbneo.exe (quark:direct relay endpoint)
        loop GGPO via relay
            A<<->>SRV: UDP packets through relay pool
            SRV<<->>B: UDP packets through relay pool
        end
    end

    Note over A,B: Match ends — winner event in events.jsonl
    A->>SRV: match_ended (winner, scores, chars)
    SRV->>SRV: persist match_history (Supabase)
    SRV->>SRV: update Glicko-2 ratings (ranked matches only)
    SRV-->>A: return to lobby
    SRV-->>B: return to lobby
```

---

## UDP Relay Pool

The relay is server-side, UDP-only, designed to be GGPO-transparent. Each session gets two dedicated sockets: one port per player. Packets arriving on player A's port are forwarded via player B's socket (so GGPO sees the source as the relay port it expects), and vice versa.

```mermaid
flowchart LR
    PA["Player A\nGGPO → relay:portA"]
    PB["Player B\nGGPO → relay:portB"]

    subgraph RELAY["UDP Relay Pool (server)"]
        SA["sockA\nbinds portA"]
        SB["sockB\nbinds portB"]
    end

    PA -->|"send to portA"| SA
    SA -->|"forward via sockB\nto P2's address"| PB
    PB -->|"send to portB"| SB
    SB -->|"forward via sockA\nto P1's address"| PA
```

**Pool parameters (defaults):**

| Parameter | Default |
|-----------|---------|
| Port range | 40000 – 41000 (1 000 ports) |
| Session timeout | 90 s of inactivity |
| Sweep interval | every 30 s |
| Bind IP | `0.0.0.0` |

If the pool is exhausted the server closes the room with reason `punch_failed_relay_pool_exhausted`.

---

## Glicko-2 Ranking System

Ratings are computed per-player, per-game, per-season using the Glicko-2 algorithm (Illinois convergence for volatility). The internal Glicko-2 values are never exposed to clients; only the integer `visible_rank` (0–6) is shown.

```mermaid
flowchart TB
    subgraph MATCH["Ranked Match ends"]
        RES["match_ended\n(winner confirmed / forfeit)"]
    end

    subgraph RANKING["ranking.ts — Glicko-2 engine"]
        FETCH["Fetch current ratings\nfrom player_game_ratings\n(or use defaults if new player)"]
        PAIR["fetchRecentPairMatchCount\n(anti-boosting: last 24 h)"]
        BOOST["boostMultiplier\n≤2 matches → 1.0×\n≤4 matches → 0.5×\n>4 matches → 0.25×"]
        COMPUTE["computeRatingUpdate\n(Glicko-2 step 5 — Illinois algo)\nupdate winner (score=1)\nupdate loser  (score=0)"]
        RANK["visibleRankFromRating\n(map rating → rank tier)"]
    end

    subgraph PERSIST["rankingPersist.ts"]
        UPSERT["upsert player_game_ratings\n(winner + loser)"]
        HIST["insert ranking_history\n(2 rows — audit trail)"]
    end

    RES --> FETCH
    RES --> PAIR
    PAIR --> BOOST
    FETCH --> COMPUTE
    BOOST --> COMPUTE
    COMPUTE --> RANK
    RANK --> UPSERT
    COMPUTE --> HIST
```

### Rank tiers

| Rank | ID | Min rating |
|------|----|-----------|
| NR   | 0  | — (0 ranked matches completed) |
| E    | 1  | 0 |
| D    | 2  | 1 300 |
| C    | 3  | 1 500 (starting point) |
| B    | 4  | 1 700 |
| A    | 5  | 1 900 |
| S    | 6  | 2 100 |

### Glicko-2 parameters

| Parameter | Value |
|-----------|-------|
| Initial rating | 1 500 |
| Initial RD | 350 |
| Initial volatility | 0.06 |
| τ (system constant) | 0.5 |
| Scale factor | 173.7178 |

RD inflation is applied per rating period for inactive players (`applyRdInflation`).

---

## Database Schema (Supabase)

```mermaid
erDiagram
    auth_users {
        uuid id PK
    }

    profiles {
        uuid id PK
        text display_name
        text avatar_url
        text country
        bool is_public
        timestamptz created_at
        timestamptz updated_at
    }

    match_history {
        uuid id PK
        uuid p1_id FK
        uuid p2_id FK
        text game
        text match_type
        integer ft_n
        integer winner
        text status
        integer p1_score
        integer p2_score
        text p1_char
        text p2_char
        timestamptz played_at
    }

    ranking_seasons {
        uuid id PK
        text name
        timestamptz started_at
        timestamptz ended_at
        bool active
    }

    player_game_ratings {
        uuid player_id PK
        text game PK
        uuid season_id PK
        float rating
        float rd
        float volatility
        smallint visible_rank
        int wins
        int losses
        int streak
        int best_streak
        int total_matches
        timestamptz last_match_at
    }

    ranking_history {
        uuid id PK
        uuid player_id FK
        text game
        uuid season_id FK
        float rating_before
        float rating_after
        float rd_before
        float rd_after
        smallint rank_before
        smallint rank_after
        text match_room_id
        text outcome
        float boost_multiplier
        timestamptz occurred_at
    }

    auth_users ||--o{ profiles : "has"
    auth_users ||--o{ match_history : "plays as p1"
    auth_users ||--o{ match_history : "plays as p2"
    auth_users ||--o{ player_game_ratings : "rated"
    auth_users ||--o{ ranking_history : "history"
    ranking_seasons ||--o{ player_game_ratings : "season"
    ranking_seasons ||--o{ ranking_history : "season"
```

**RLS summary:**

| Table | Public read | Own read | Write |
|-------|-------------|----------|-------|
| `profiles` | if `is_public = true` | yes | own row |
| `match_history` | if profile is public | yes | service role |
| `player_game_ratings` | if profile is public | yes | service role |
| `ranking_history` | if profile is public | yes | service role |
| `ranking_seasons` | yes (everyone) | — | service role |

---

## Client Internal Architecture

```mermaid
flowchart LR
    subgraph MAIN["Main Process (index.js)"]
        MFS["matchFlowState.js\nstate machine"]
        CS["candidateSelection.js\nLAN vs WAN selection"]
        NFL["netplayForwardLogger.js"]

        subgraph RUNTIME["Runtime"]
            SESS["rfbneoSession.js\nquark:direct builder\nname sanitization"]
            SSESS["rfbneoSecureSession.js\nAES-256-GCM payload"]
            LAUNCH["rfbneoLaunch.js\nprocess spawn"]
            EVTS["rfbneoEvents.js\nevents.jsonl tail parser"]
            RMGR["rfbneoRuntimeManager.js\nlifecycle management"]
        end

        subgraph AUTH["Auth"]
            SUPA["supabaseAuth.js\nJWT session (persistent)"]
        end

        subgraph CONFIG["Config"]
            ENV["env.js"]
            UDATA["userData.js\npaths / ROM folder"]
            CTRL["clientControls.js\nhotkeys / presets"]
        end
    end

    subgraph PRELOAD["preload.js\n(contextBridge IPC)"]
    end

    subgraph RENDERER["Renderer (React + Vite)"]
        GSP["GameSelectPage"]
        LP["LobbyPanel"]
        SS["StatusSidebar"]
        GC["gameCatalog.js\nremote catalog"]
        CI["clientInput.js\ninput lock/unlock"]
    end

    subgraph FBNEO["retrofightfbneo.exe"]
        GGPO["fbn_ggpo.cpp\nGGPO + quark:direct parser"]
        OVL["vid_overlay.cpp\nrank badge rendering"]
        EJSONL[("events.jsonl\nW · X · C · D")]
    end

    MFS --> CS
    MFS --> SESS
    SESS --> SSESS
    SESS --> LAUNCH
    LAUNCH --> FBNEO
    EVTS -->|tail parse| EJSONL
    EVTS --> MFS

    MAIN <-->|IPC| PRELOAD
    PRELOAD <-->|IPC| RENDERER
```

---

## Server Internal Architecture

```mermaid
flowchart LR
    subgraph SERVER["retrofight-server/src/"]
        ST["index.ts / server.ts\nSocket.IO wiring\nmain entry"]
        AU["auth.ts\nJWT verify\nduplicate socket block"]
        LB["lobby.ts\nlobby join/leave\ndriver filter"]
        RM["rooms.ts\nroom state"]
        RT["roomTransitions.ts\nstate machine"]
        SR["staleRooms.ts\nstale room cleanup"]
        NP["netplayProtocol.ts\nnetplay events & states"]
        MR["matchResults.ts\nwinner / forfeit / dispute"]
        MHP["matchHistoryPersist.ts\nwrite to Supabase"]
        MHC["matchHistoryClient.ts\nSupabase service-role client"]
        RNK["ranking.ts\nGlicko-2 engine"]
        RNKP["rankingPersist.ts\nrating upsert + history insert"]
        RNKC["rankingClient.ts\nSupabase rating fetch"]
        UDR["udpRelayServer.ts\nUDP relay pool"]
        RAA["runtimeArtifactApi.ts\nexe upload / download"]
        EP["endpoints.ts\nREST routes"]
        CFG["config.ts\nenv vars"]
        LOG["logger.ts\nstructured logging"]
    end

    ST --> AU
    ST --> LB
    ST --> RM
    ST --> NP
    ST --> UDR
    RM --> RT
    RT --> SR
    MR --> MHP
    MR --> RNKP
    RNKP --> RNK
    RNKP --> RNKC
    MHP --> MHC
    RAA --> EP
```

---

## FBNeo Runtime Events

Events written by `retrofightfbneo.exe` to `events.jsonl` and consumed by `rfbneoEvents.js`:

```mermaid
flowchart LR
    FBNEO["retrofightfbneo.exe"]
    EJSONL[("events.jsonl")]
    RFEVTS["rfbneoEvents.js\n(tail parser)"]
    MFS["matchFlowState.js"]

    FBNEO -->|"write JSON Lines"| EJSONL
    RFEVTS -->|"tail read"| EJSONL

    RFEVTS -->|"cmd: V — protocol version"| MFS
    RFEVTS -->|"cmd: W — winner,score1,score2,char1,char2"| MFS
    RFEVTS -->|"cmd: X — turbo/input anomaly"| MFS
    RFEVTS -->|"cmd: D — loaded / missing ROM"| MFS
```

> `cmd: C` (internal peer command) is ignored — never interpreted as a score event.

---

## Authentication Flow

```mermaid
flowchart LR
    subgraph WEB["retrofight-web"]
        WREG["Registration\n(email + password)"]
        WLOG["Login"]
        WPROF["Profile / Match History\n(SSR, cookie session)"]
    end

    subgraph CLIENT["retrofight-client"]
        CLOG["Login only\n(no registration)"]
        CSESS["Persistent session\n(userData)"]
    end

    subgraph SUPA["Supabase"]
        SAUTH["Auth\n(JWT / email+password)"]
        SDB[("Database")]
    end

    subgraph SERVER["retrofight-server"]
        SJWT["JWT verification\n(SUPABASE_JWT_SECRET)"]
        SDUP["Duplicate socket\nblock per sub"]
    end

    WREG -->|"signUp"| SAUTH
    WLOG -->|"signIn"| SAUTH
    WPROF -->|"SSR session cookie"| SAUTH
    WPROF -->|"read profiles\nmatch history\nleaderboard"| SDB

    CLOG -->|"signIn → JWT"| SAUTH
    CLOG --> CSESS

    CSESS -->|"attach JWT to Socket.IO handshake"| SJWT
    SJWT -->|"verify via Supabase"| SAUTH
    SJWT --> SDUP
```

---

## Secure Session (quark:direct payload)

```mermaid
flowchart LR
    A["rfbneoSession.js\nbuild quark:direct string\nsanitize player names"]
    B["rfbneoSecureSession.js\nencrypt with AES-256-GCM\nwrite to temp file"]
    C["rfbneoLaunch.js\nspawn retrofightfbneo.exe"]
    D["retrofightfbneo.exe\n--retrofight-session path\nkey via stdin\nRETROFIGHT_FBNEO_SESSION_MODE=secure"]
    E["GGPO\n(direct or relay endpoint)"]

    A --> B
    B -->|"temp session file"| C
    C -->|"spawn + stdin key"| D
    D -->|"parse quark:direct\nstart GGPO"| E
```

---

## Key Notes

- **Rank overlay** — always shows `rank0` because `RetrofightSetDirectGameInfo()` hardcodes `#0,0,0`; Glicko-2 ranks are stored in DB but a protocol extension is needed to pass them into the runtime badge.
- **`ranked` field** in `quark:direct` is the match format (0 = versus, 1 = ranked, 2+ = FTN), not the player's rank badge.
- **`launcher.js` is legacy** — not part of the main match flow.
- **Anti-boosting** — the same pair playing more than 2 ranked matches in 24 h receives reduced rating deltas (0.5× or 0.25×) to prevent farming.
- **NULL season** — `season_id IS NULL` in `player_game_ratings` represents all-time totals; a unique partial index enforces one row per `(player_id, game)` when no season is active.


## Legal documents

- [Legal Notice](https://github.com/Retrofight/retrofight-legal/blob/main/LEGAL_NOTICE.md)
- [Terms of Use](https://github.com/Retrofight/retrofight-legal/blob/main/TERMS_OF_USE.md)
- [First Run Disclaimer](https://github.com/Retrofight/retrofight-legal/blob/main/FIRST_RUN_DISCLAIMER.md)
- [Download Disclaimer](https://github.com/Retrofight/retrofight-legal/blob/main/DOWNLOAD_DISCLAIMER.md)
- [Third-Party Content Notice](https://github.com/Retrofight/retrofight-legal/blob/main/THIRD_PARTY_CONTENT.md)
- [Copyright Policy](https://github.com/Retrofight/retrofight-legal/blob/main/COPYRIGHT_POLICY.md)
- [Legal FAQ](https://github.com/Retrofight/retrofight-legal/blob/main/FAQ_LEGAL.md)
