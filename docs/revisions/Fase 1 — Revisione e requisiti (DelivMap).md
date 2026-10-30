# Fase 1 — Decisioni di revisione e requisiti per la realizzazione

**Progetto:** DelivMap — Gestione consegne a domicilio per piccoli ristoranti **Argomento:** Decisioni prese in revisione del documento "Fase 1 — Gestione Turni e Ordini" e requisiti aggiornati per la realizzazione

## Problemi emersi dalla revisione

| # | Problema | Priorità |
| --- | --- | --- |
| 1 | Le tabelle `Turni`, `Clienti`, `Ordini` sono ridefinite in Fase 1 con classi omonime e campi diversi rispetto a Fase 0 (`orarioRichiesto`, `stato`, vincoli di lunghezza): conflitto di nomi, il progetto non compila | Critica |
| 2 | Lo storico degli ordini non è affidabile: l'indirizzo viene letto sempre dal record `Clienti` corrente, che `creaOrdineTransazionale` e `aggiornaOrdineCompleto` sovrascrivono a ogni nuovo ordine dello stesso telefono | Critica |
| 3 | `in_ritardo` è sia un valore accettato da `PATCH /ordini/<id>/stato` sia un'etichetta calcolata lato client da `calcolaRitardo()` | Alta |
| 4 | Rollback ottimistico di Riverpod: in `cambiaStato` (e `riordina`) l'errore sovrascrive subito il ripristino dei dati precedenti | Media |
| 5 | `json_annotation` / `json_serializable` usati dai DTO ma assenti dal `pubspec.yaml` di Fase 0 | Media |
| 6 | `TurniApi` e `OrdiniApi` non sono montate nel `LocalApiServer` di Fase 0 | Media |
| 7 | `DateFormat(..., 'it_IT')` richiede `initializeDateFormatting('it_IT')`, assente dal bootstrap: crash al primo rendering della dashboard | Media |
| 8 | `GET /api/v1/turni` fa una query di statistiche per ogni turno (N+1) | Bassa |

## Decisioni confermate

- **Schema canonico: versione Fase 1.** `orarioRichiesto` come testo `HH:mm`, `stato` come testo libero con default `in_attesa`, validato lato API.
- **Snapshot su `Ordini`.** La tabella guadagna un proprio indirizzo e nome di consegna, scritti alla creazione dell'ordine. `Clienti` resta solo per il riconoscimento rapido via telefono.
- **`in_ritardo` resta scrivibile** via API, come nel documento originale. Rischio noto: un client diverso dalla UI attuale (un test, la futura app fattorino) che invia `PATCH stato=in_ritardo` sovrascrive lo stato reale del flusso (es. "in consegna"), perdendo quell'informazione.
- **Rollback ottimistico Riverpod: rimandato.** Non blocca l'uso base; da riprendere in una fase successiva (pattern previsto: `AsyncValue.error(...).copyWithPrevious(...)`).

## Effetto collaterale sulla Fase 0

Con lo schema Fase 1 la colonna `stato` non usa più `textEnum<StatoOrdine>()`, quindi il bug del `TypeConverter` segnalato in revisione Fase 0 decade. L'enum `StatoOrdine` resta utile come riferimento Dart per etichette e validazione.

## Cosa serve per realizzare la Fase 1

### Schema `Ordini` (file unico `lib/data/tables/ordini.dart`)

Eliminare i file duplicati `*_table.dart` del documento originale e mantenere la convenzione di nomi di Fase 0.

- Campi esistenti: `id`, `clienteId` (FK), `turnoId` (FK), `importo`, `orarioRichiesto` (testo `HH:mm`), `stato` (testo, default `in_attesa`), `sequenza`, `note`
- Campi nuovi: `indirizzoConsegna` e `nomeClienteConsegna`, scritti alla creazione e indipendenti dal record `Clienti`

### Correzioni ai DAO e alle API

- `creaOrdineTransazionale` e `aggiornaOrdineCompleto` scrivono `indirizzoConsegna` e `nomeClienteConsegna` sull'ordine. L'aggiornamento del record `Clienti` condiviso resta una scrittura separata, così un ordine passato non cambia più retroattivamente.
- `OrdiniApi.statiValidi` derivato da `StatoOrdine.values.map((e) => e.codice)`, non da una lista scritta a mano, per evitare disallineamenti nel tempo.
- Le risposte (`toResponseDto`) leggono indirizzo e nome dallo snapshot dell'ordine, non dal cliente.

### Dipendenze da aggiungere al `pubspec.yaml`

- `json_annotation` (runtime)
- `json_serializable` (dev, per il codegen dei DTO)
- Installare con `flutter pub add`, senza ricopiare versioni fisse

### Collegamento al server di Fase 0

Montare `TurniApi` e `OrdiniApi` nel router di `LocalApiServer`. `AppDatabase`, DAO, repository e le due API vanno istanziati dentro `runServerIsolate()`, mai nella UI, coerentemente con la decisione che il database vive solo nel server isolate.

### Bootstrap (`main.dart`)

`initializeDateFormatting('it_IT', null)` prima del primo uso di `DateFormat` con locale italiano.

### Facoltativo, non bloccante

Sostituire il ciclo N+1 di `GET /api/v1/turni` con un'unica query aggregata `JOIN` + `GROUP BY` per le statistiche dei turni.

## Criteri di accettazione da aggiornare

- Aggiungere un criterio sullo snapshot: due ordini con lo stesso telefono e indirizzi diversi mantengono ciascuno il proprio indirizzo di consegna.
- Aggiungere un criterio sulla coerenza di `statiValidi` con `StatoOrdine`.
- Verificare che il progetto compili con un solo set di definizioni tabella.

## Prossimo passo

Da decidere: preparare la versione corretta del documento Fase 1 con questi cambiamenti incorporati, oppure passare alla revisione della Fase 2.
