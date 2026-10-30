# Fase 0 — Decisioni di revisione e requisiti per la realizzazione

**Progetto:** DelivMap — Gestione consegne a domicilio per piccoli ristoranti **Argomento:** Decisioni prese in revisione del documento "Fase 0 — Scaffolding, Architettura e Database" e requisiti aggiornati per la realizzazione

## Decisioni confermate

- **database\_provider.dart rimosso** dall'albero `ui/providers/`: nessuna preferenza espressa dall'utente, quindi si procede con la scelta coerente all'architettura — `AppDatabase` vive solo dentro il server isolate, mai esposto direttamente alla UI
- **Fase 0 resta scaffolding puro**: le route CRUD (ordini/clienti/turni/impostazioni) e i relativi DTO restano fuori da questa fase e diventano contenuto della Fase 1
- **Istanza singola inclusa già in Fase 0**: va prevenuto l'avvio di due processi sulla stessa porta
- **Hashing password con bcrypt e salt**, al posto di SHA-256/HMAC puro

## Rilevamento istanza singola — nessuna dipendenza aggiuntiva necessaria

Non serve un pacchetto nuovo. Si può riusare il meccanismo già presente: se il bind sulla porta 8080 fallisce con `SocketException` all'avvio dell'Isolate, significa che un'altra istanza è già in esecuzione. In quel caso si mostra un dialog ("DelivMap è già aperto") e si chiude la seconda istanza, invece di lasciarla girare con l'API silenziosamente offline.

## Bug da correggere: colonna `stato` e l'enum `StatoOrdine`

La colonna `stato` in `ordini.dart` usa `textEnum<StatoOrdine>()()`, che di default serializza tramite `Enum.name` (es. `"inAttesa"`), non tramite il campo `codice` definito a mano nell'enum (es. `"in_attesa"`). Per far sì che la colonna usi davvero `codice`, serve aggiungere un `TypeConverter` dedicato — altrimenti il lavoro fatto su `codice`/`fromString` resta codice morto.

## Cosa serve per realizzare la Fase 0 (aggiornata)

### Ambiente di sviluppo

- Flutter SDK
- Visual Studio 2022 con il carico di lavoro "Desktop development with C++"
- VS Code con estensioni Flutter e Dart
- `flutter doctor -v` con tutte le voci relative a Windows desktop verdi

### Decisioni di design da applicare rispetto al documento originale

- Rimuovere `database_provider.dart` dall'albero/placeholder — `AppDatabase` istanziato solo dentro `runServerIsolate`
- Non creare le route CRUD (`ordini_route.dart`, `clienti_route.dart`, `turni_route.dart`, `impostazioni_route.dart`) né i DTO in questa fase — solo `/health` e lo scheletro del router — e aggiornare l'albero delle cartelle/placeholder perché sia coerente con questo scope (toglierli dall'elenco o segnarli esplicitamente come "Fase 1")
- Aggiungere la logica di rilevamento istanza singola nel bootstrap (`main.dart` / `server.dart`), basata sul fallimento del bind alla porta
- Sostituire `crypto` con un pacchetto di password hashing dedicato (es. `bcrypt` su pub.dev) e aggiungere un campo salt/cost nella tabella `Utenti` se il pacchetto scelto non lo gestisce internamente
- Aggiungere un `TypeConverter` per la colonna `stato`, così da usare davvero `codice` invece del default di `textEnum()`

### Pacchetti

Installare con `flutter pub add` invece di ricopiare le versioni fissate nel documento originale, già superate rispetto a pub.dev (es. drift è alla 2.35.x, non alla 2.20.0 indicata). Questo vale per tutta la lista di dipendenze del Task 0.2, bcrypt incluso.

### Altro

- Repository Git inizializzato prima del primo commit dello scaffolding
- Aggiornare i criteri di accettazione del documento originale per riflettere questi quattro cambiamenti, altrimenti i test di accettazione "passano" pur non rispettando le decisioni appena prese

## Prossimo passo

Da decidere: preparare la versione corretta e completa del documento Fase 0 con questi cambiamenti incorporati, oppure passare direttamente alla Fase 1 e sistemare questi dettagli durante l'implementazione.
