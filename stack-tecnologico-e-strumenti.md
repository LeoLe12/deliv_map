# Stack tecnologico e strumenti necessari
 
*Riferimento operativo per l'ambiente di sviluppo. Complementare al piano di sviluppo (`piano-sviluppo-app-consegne.md`).*
 
## 1. Ambiente di sviluppo
 
- **Flutter SDK** (versione stabile più recente) — include già il Dart SDK
- **IDE**: VS Code (con estensioni Flutter e Dart) oppure Android Studio
- **Git** per il versionamento del codice
- **Toolchain nativa per il target desktop**, necessaria per compilare Flutter Desktop:
  - Windows: Visual Studio (componenti "Sviluppo desktop con C++")
  - macOS: Xcode
  - Linux: pacchetti di build standard (`clang`, `cmake`, `ninja-build`, `pkg-config`, `libgtk-3-dev`)
## 2. Librerie/pacchetti Dart e Flutter principali
 
| Pacchetto | A cosa serve |
|---|---|
| `drift` (basato su `sqflite_common_ffi`) | ORM e gestione schema/migrazioni su SQLite — vedi sezione 3 |
| `shelf` + `shelf_router` | Server API locale in Dart |
| `google_maps_flutter` (o WebView verso Maps JavaScript API) | Visualizzazione mappa |
| `http` | Chiamate alle API esterne (Directions, Geocoding) |
| `path_provider` | Individuare la cartella dati dell'app dove salvare il database |
| `provider` o `riverpod` (a scelta) | Gestione dello stato dell'app |
 
## 3. Database locale: storage, schema e migrazioni
 
### Perché Drift (non sqflite "nudo")
 
Drift è un ORM Dart costruito sopra SQLite, pensato apposta per Flutter. Rispetto a scrivere SQL a mano con `sqflite_common_ffi` direttamente:
 
- Tabelle definite come classi Dart tipizzate — errori di schema individuati in fase di compilazione, non a runtime
- Gestione strutturata delle **migrazioni**, utile perché lo schema del v1 crescerà nel v2 (tabella fattorini, assegnazioni)
- Query reattive (stream): la UI si aggiorna da sola quando cambia lo stato di un ordine, senza ricaricare manualmente
### Dove vive il file
 
Il database è un singolo file SQLite, salvato nella cartella dati dell'applicazione (tramite `path_provider`, es. `getApplicationSupportDirectory()`) — non nella cartella di installazione del programma, così il file sopravvive agli aggiornamenti del software.
 
### Schema v1 (tabelle principali)
 
| Tabella | Campi principali | Note |
|---|---|---|
| `ristoranti` | id, nome, indirizzo, zona_consegna, maps_api_key | Riga singola nel v1 (un solo ristorante per installazione) |
| `utenti` | id, nome_utente, password_hash, ruolo | Ruolo = "gestore" nel v1 |
| `clienti` | id, nome, telefono, indirizzo | Indicizzato su telefono per ricerca rapida |
| `ordini` | id, cliente_id (FK), importo, orario_richiesto, stato, sequenza, note, turno_id (FK) | `stato`: in_attesa / in_consegna / consegnato / in_ritardo |
| `turni` | id, data, note | Raggruppa gli ordini di una sessione di consegne |
 
### Migrazioni
 
Drift versiona lo schema con un intero (`schemaVersion`). Ogni cambiamento futuro (es. tabella `fattorini` nel v2) diventa una migrazione incrementale, eseguita in automatico all'avvio se il file trovato ha una versione precedente — nessun rischio di perdere i dati esistenti quando il software viene aggiornato.
 
## 4. Servizi esterni da configurare
 
- Account **Google Cloud Platform**
- Attivazione **Google Maps Platform**: Maps SDK, Directions API, Geocoding API
- Generazione della **chiave API**, con restrizioni (per app/dominio) per evitare usi impropri
- Attivazione della fatturazione Google Cloud (richiesta anche per restare nel free tier)
## 5. Strumenti di supporto al progetto
 
- Repository **Git** (GitHub o GitLab) per versionamento e backup del codice
- Uno strumento leggero di gestione task/roadmap (board Trello/Notion, oppure semplicemente il file di roadmap già creato)
- Opzionale: **Figma** per abbozzare le schermate principali prima di scriverle in Flutter
## 6. Distribuzione (v1)
 
- Build Flutter Desktop per Windows/macOS/Linux (in base a dove operano i ristoranti target)
- Eventuale installer per facilitare l'installazione presso il locale (es. Inno Setup su Windows)
- Meccanismo semplice di controllo aggiornamenti, coerente col modello di business "licenza + aggiornamenti"
