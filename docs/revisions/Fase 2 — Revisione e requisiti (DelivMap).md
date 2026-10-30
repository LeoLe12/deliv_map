# Fase 2 — Decisioni di revisione e requisiti per la realizzazione

**Progetto:** DelivMap — Gestione consegne a domicilio per piccoli ristoranti **Argomento:** Decisioni prese in revisione del documento «Fase 2 — Anagrafica Clienti» e requisiti aggiornati per la realizzazione

## Problemi emersi dalla revisione

| # | Problema | Priorità |
| --- | --- | --- |
| 1 | `Clienti` e `Ordini` ridefinite con lo schema vecchio (niente snapshot, `orarioRichiesto` come `DateTime`, `stato` senza converter, FK `restrict`, `turnoId` senza `cascade`) e in file duplicati `*_table.dart` | Critica |
| 2 | Due meccanismi per creare il cliente: upsert lato server in `POST /ordini` (Fase 1) e creazione lato client nel form (Fase 2). Il form non chiama mai `POST /ordini`, la checkbox «Salva in rubrica» non ha effetto, un telefono già presente blocca l'ordine con 409 | Critica |
| 3 | Blocco dell'eliminazione per qualsiasi ordine, in contrasto con lo snapshot | Critica |
| 4 | Debounce dell'autocompletamento sbagliato: `family` con la query come argomento, quindi una richiesta per tasto oppure nessun risultato | Alta |
| 5 | `FocusNode()` creato dentro `build` | Alta |
| 6 | Il cliente selezionato non si scollega se si modificano nome o telefono: l'ordine può finire sul cliente sbagliato | Alta |
| 7 | `OrdineSinteticoResponse` non compila (`DateTime.parse('20:30')`, `StatoOrdine` assegnato a `String`); storico ordinato per ora del giorno e senza data del turno | Alta |
| 8 | N+1 su `countOrdiniByCliente` | Media |
| 9 | Telefono non normalizzato: `unique()` non evita i duplicati con formati diversi | Media |
| 10 | Lista limitata a 100 clienti senza paginazione nella UI | Media |
| 11 | Contratto API incoerente con Fase 1 (`error` vs `errore`, camelCase vs snake\_case, `DELETE` 200 vs 204), `ApiException` duplicata, `localhost` al posto di `127.0.0.1`, nessun timeout | Media |
| 12 | `ClientiRoutes` non montato in `ApiRouter`; DAO usato senza repository | Media |
| 13 | Lo spinner sostituisce la lista a ogni tasto (`skipLoadingOnReload`) | Bassa |
| 14 | Affermazioni non misurate: «< 10 ms grazie all'indice», «60 s → 10 s», «piena conformità GDPR»; `.unique()` rende ridondante `idx_clienti_telefono` | Bassa |
| 15 | `OrdineFormDialog` duplica il form di Fase 1 e manca di orario e validazione importo | Bassa |
| 16 | Lunghezze del telefono incoerenti tra le fasi (5–25, 3–30, 5–20) e `CHECK` generati da `withLength` | Bassa |

## Decisioni confermate

- **Cliente solo lato server.** Creazione e aggiornamento del cliente avvengono solo dentro `POST /ordini`, in un'unica transazione.
- **Eliminazione cliente.** Un cliente con ordini attivi (`stato != consegnato`) non si elimina. Per sbloccarlo basta segnare gli ordini come consegnati e poi eliminare. Con soli ordini passati l'eliminazione è consentita: il riferimento al cliente diventa NULL (`setNull`) e lo snapshot resta nell'ordine. Il messaggio 409 deve dirlo chiaramente.
- **Telefono normalizzato.** Il dato si salva in forma compatta; la formattazione con gli spazi è solo rendering lato UI. Il formato compatto è modificabile in corso d'opera, a patto che la logica resti in un'unica funzione: dopo la distribuzione un cambio richiede una migrazione che rinormalizza la colonna.
- **Contratto API come Fase 1.** Chiavi snake\_case, errori con chiave `errore`, `DELETE` con 204, un solo `ApiClient` condiviso.
- **Snapshot e retention: rimandato.** La gestione dei dati personali che restano nello snapshot degli ordini viene affrontata più avanti.

## Regola di normalizzazione del telefono (proposta)

1. Rimuovere spazi, punti, trattini e parentesi.
2. Un prefisso `00` diventa `+`.
3. Il prefisso `+39` viene tolto: i numeri italiani si salvano in formato nazionale. Lo zero iniziale dei fissi resta (`+39 06 1234567` diventa `061234567`).
4. Gli altri prefissi internazionali restano con il `+` (`+44…`).
5. Validazione dopo la normalizzazione: solo cifre (con `+` iniziale ammesso), da 6 a 15 cifre (massimo E.164).
6. Stessa funzione per `POST/PUT /clienti`, upsert da `POST /ordini`, snapshot `telefono_consegna` e ricerca. Nella ricerca, se `q` contiene solo caratteri da telefono, si normalizza `q` e si cerca sulla colonna compatta; il prefisso si tronca solo se `q` inizia con `+` o `00`.
7. Visualizzazione: funzione `formattaTelefono()` solo lato UI (cellulare italiano di 10 cifre che inizia con 3 → `333 123 4567`, altri numeri invariati).

## Cosa serve per realizzare la Fase 2

### Schema (nessuna ridefinizione)

- Un solo file `clienti.dart`: partire da quello di Fase 0/1 e aggiungere `createdAt` e `updatedAt`.
- Eliminare `clienti_table.dart` e `ordini_table.dart` della Fase 2: `Ordini` resta quello di Fase 1 (snapshot, `HH:mm`, `StatoOrdineConverter`, `clienteId` nullable con `setNull`, `turnoId` con `cascade`).
- Telefono `unique()` sul valore normalizzato; togliere `@TableIndex idx_clienti_telefono`, perché `unique` crea già l'indice.
- Costanti condivise per le lunghezze di nome, indirizzo e telefono (telefono: 16 caratteri al massimo, cioè 15 cifre più il `+`), usate da tabelle, snapshot e validazione.
- Se esiste già un `delivmap.sqlite`: cancellarlo in sviluppo, oppure alzare `schemaVersion` e scrivere la migrazione per le due colonne nuove.

### Ordine ↔ cliente (solo lato server)

- `OrdineRequest` guadagna `cliente_id` (opzionale) e `salva_in_rubrica` (default `true`).
- `creaOrdineTransazionale`, in un'unica transazione: con `cliente_id` usa quel cliente (422 se non esiste); senza `cliente_id` e con `salva_in_rubrica` fa l'upsert per telefono normalizzato; altrimenti `clienteId = null`.
- Proposta sull'aggiornamento della rubrica: con `cliente_id` selezionato non si sovrascrivono i dati della rubrica, che si modificano solo con `PUT /clienti`.
- Il form non chiama più `POST /clienti`. La checkbox diventa `salva_in_rubrica` e il cliente si scollega se nome o telefono divergono dai valori selezionati.
- Il task 2.7 si riduce: niente `LEFT JOIN` e niente `watch…`; il nome mostrato viene dallo snapshot e il badge «registrato» è `cliente_id != null`.

### DAO, repository e API

- `ClientiDao`: CRUD e ricerca; lista con conteggio ordini in un'unica query `LEFT JOIN` + `GROUP BY`; `getOrdiniByCliente` con join su `Turni` ordinato per `turni.data desc, orarioRichiesto desc`; `countOrdiniAttiviByCliente` (`stato != consegnato`). Rimuovere i `watch*`, inutili con l'architettura HTTP.
- `ClientiRepository` come per turni e ordini; `ClientiRoutes` rinominato `ClientiApi` e montato in `ApiRouter` con `v1.mount('/clienti', …)`, con DAO e repository creati dentro l'isolate.
- `DELETE /clienti/<id>`: 409 `ORDERS_ATTACHED` solo se ci sono ordini attivi, con numero di ordini e turni coinvolti e l'indicazione di segnarli come consegnati; altrimenti elimina e risponde 204.
- Contratto come Fase 1: chiavi snake\_case (`created_at`, `totale_ordini`, `turno_id`), errori con chiave `errore`.
- Validazioni: `limit` della ricerca tra 1 e 50, query minima di 2 caratteri, escape di `%` e `_` nel `LIKE`.

### Flutter

- DTO con `json_serializable`. `OrdineSinteticoResponse`: `orario_richiesto` come testo, `data_turno` come data, `stato` come codice.
- `ClientiService` riusa `ApiClient` (`127.0.0.1`, timeout, `SocketException` gestita), senza una seconda `ApiException`.
- Autocompletamento: debounce con `Timer` nel widget, che diventa `StatefulWidget` con proprio `FocusNode` liberato con `dispose`; chiamata diretta al service; eliminare la `family` `clienteAutocompleteProvider`.
- Lista: `skipLoadingOnReload: true`; `ref.invalidate` al posto di `state = loading` dopo create, update e delete; paginazione reale con `total_count`.
- Nel form ordine di Fase 1 (`FormNuovoOrdineDialog`) sostituire i campi nome e telefono con `ClienteAutocompleteField`, mantenendo orario e validazione dell'importo. Niente secondo dialog.

### Da correggere nel documento originale

- Criterio CA-2.2: togliere «< 10 ms grazie all'indice». Criterio misurabile proposto: ricerca su 5.000 clienti di prova sotto i 100 ms, misurata.
- Togliere «piena conformità GDPR» e «da 60 s a 10 s», oppure dichiararli obiettivi non misurati.
- Test del debounce: 10 caratteri digitati a meno di 300 ms l'uno dall'altro producono una sola richiesta, contata con un `http.Client` finto.
- Test da aggiungere:
  - eliminazione con un ordine `in_attesa` → 409;
  - eliminazione con soli ordini `consegnato` → 204, `cliente_id` NULL e snapshot intatto;
  - ordine con `cliente_id` inesistente → 422;
  - stesso telefono in formati diversi (`+39 333 123 4567`, `0039 333 1234567`, `333-123.4567`) → stesso cliente e 409 sul duplicato;
  - `(06) 1234567` e `+39 06 1234567` → `061234567`;
  - ricerca `333 12` → trova `3331234567`.

## Rimandato

- **Snapshot e retention.** Lo snapshot conserva nome, telefono e indirizzo anche dopo l'eliminazione del cliente. La policy di retention (per esempio cancellazione degli ordini oltre N mesi) è rimandata; per i dettagli legali conviene sentire un consulente.

## Prossimo passo

Revisione della Fase 3.
