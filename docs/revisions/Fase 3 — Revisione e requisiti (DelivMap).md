# Fase 3 — Decisioni di revisione e requisiti per la realizzazione

**Progetto:** DelivMap — Gestione consegne a domicilio per piccoli ristoranti **Argomento:** Decisioni prese in revisione del documento «Fase 3 — Mappa e Calcolo Percorsi» e requisiti aggiornati per la realizzazione

## Problemi emersi dalla revisione

I punti 1–3 sono stati verificati sulle fonti Google (pagine Legacy, prezzi, termini di servizio); gli altri vengono dalla lettura del codice, non da una compilazione.

| # | Problema | Priorità |
| --- | --- | --- |
| 1 | Directions API (Legacy) non si può più abilitare nei nuovi progetti Google Cloud (dall'ottobre 2026). Con il BYOK ogni ristoratore crea un progetto nuovo, quindi `calcolaPercorso` fallirebbe con `REQUEST_DENIED`: serve la Routes API (`computeRoutes`) | Critica |
| 2 | Prezzi e free tier del piano non corrispondono: non c'è più il credito da 200 $, ogni SKU ha una soglia gratuita mensile propria (Geocoding 10.000 poi 5 $/1000; Compute Routes Essentials 10.000 poi 5 $/1000; Pro 5.000 poi 10 $/1000). I numeri «28.000 geocoding / 40.000 Directions» non tornano | Critica |
| 3 | I termini Google ammettono la cache di lat/lng di Geocoding e Directions per al massimo 30 giorni consecutivi; il piano le salva a tempo indeterminato in `clienti` e `ordini` | Critica (rischio accettato, vedi sotto) |
| 4 | `Clienti` e `Ordini` ridefinite con lo schema vecchio in file `*_table.dart` (niente snapshot, `clienteId` obbligatorio, `orarioRichiesto` come `DateTime`, `stato` con `in_ritardo`); `ClienteData`/`OrdineData` non esistono senza `@DataClassName`; l'`innerJoin` su `clienti` esclude gli ordini con `cliente_id` NULL; `OrdineDto` duplica `OrdineResponse` e va in errore su `cliente_id` NULL; `pubspec` con versioni vecchie e `sqlite3_flutter_libs` obsoleto | Critica |
| 5 | La UI chiama Google e legge la chiave direttamente (`mapsServiceProvider`, `settingsProvider`, mai definiti), contro l'architettura UI → API locale → Google; mancano le route `/api/v1/impostazioni` e `/api/v1/percorsi/calcola` | Critica |
| 6 | Parti descritte ma non specificate: coordinate di partenza del ristorante, `sosta_minuti`, geocoding dentro `POST /ordini`, notifica «WebSocket» inesistente, stato ed errore del geocoding non salvati, calcolo ETA, corrispondenza tratta ↔ ordine | Alta |
| 7 | Il percorso copre tutto il turno come un unico giro, consegnati inclusi, sommando i tempi: gli ETA delle ultime consegne non hanno senso | Alta |
| 8 | Cache coordinate indicizzata sul telefono e non sull'indirizzo: un cliente che ordina a un altro indirizzo riceve le coordinate vecchie | Alta |
| 9 | Geocoding: si accetta il primo risultato anche con `partial_match` o `location_type` `APPROXIMATE`, e senza `bounds` un indirizzo senza città è ambiguo | Alta |
| 10 | Due nozioni di ritardo mescolate: il marker «diventa `in_ritardo`» (stato reintrodotto); il ritardo stimato ignora `evidenzia_ritardi` e la tolleranza, usa `DateTime` e non gestisce la mezzanotte | Alta |
| 11 | `sealed class MapsApiException` non è istanziabile: i `throw const MapsApiException(...)` non compilano | Alta |
| 12 | Geocoding in background senza `try/catch`: un errore non gestito in un `Future` non atteso può terminare l'isolate del server | Alta |
| 13 | Numerazione: `sequenza` è 0-based, i marker partono da «0» mentre la lista mostra `sequenza + 1` | Media |
| 14 | `testApiKey` risponde `false` anche senza rete e prova solo Geocoding; la validazione `AIzaSy…` promessa non è implementata | Media |
| 15 | WebView: se `initialize()` fallisce c'è solo un `debugPrint` e lo spinner resta infinito; manca `gm_authFailure` e la gestione dell'errore di caricamento dello script; HTML scritto in un file temporaneo con la chiave in chiaro; `webview_windows` 0.4.0 ha il bug di perdita di focus tastiera; riferimento a Windows 7/8 non pertinente | Media |
| 16 | Pagina JS: nell'InfoWindow nome, indirizzo e note finiscono in HTML non escapato; un InfoWindow per marker, tutti aperti insieme; `fitBounds` a ogni aggiornamento (lo zoom si azzera); `oldWidget.ordini != widget.ordini` confronta l'identità della lista e rinvia i marker a ogni rebuild | Media |
| 17 | La chiave può comparire nei messaggi d'errore (`NetworkException('… $e')` contiene l'URL con la chiave) | Media |
| 18 | Dopo un riordino o un cambio di stato il percorso è obsoleto ma non c'è l'indicazione «da ricalcolare» | Media |
| 19 | «Nessun dato dei clienti esce dal PC» è fuorviante: gli indirizzi vengono inviati a Google per il geocoding | Media |
| 20 | `google.maps.Marker` è deprecato dal febbraio 2024; non ho trovato una data di rimozione. L'alternativa `AdvancedMarkerElement` richiede un Map ID | Bassa |
| 21 | La stima di 5–7 giorni non copre le parti non specificate né la migrazione a Routes API | Bassa |

## Decisioni confermate

- **Routes API al posto di Directions.** Tempi di percorrenza **senza traffico** nell'MVP (SKU Essentials); l'ETA è dichiarato «senza traffico» nella UI.
- **WebView:** fork mantenuto `webview_flutter_windows`, che richiede Flutter 3.44+ e Dart 3.12+ e corregge la perdita di focus tastiera. L'API va verificata sul README del fork: gli snippet del piano usano quella di `webview_windows`.
- **Coordinate persistenti come da piano.** Scelta confermata, con il rischio rispetto al limite di 30 giorni dei termini Google accettato (vedi Rischi accettati).
- **Percorsi su sottogruppi scelti dal ristoratore.** Sulla mappa compaiono gli ordini non consegnati con l'orario richiesto; il ristoratore li seleziona creando sottogruppi. Per consegne con orario simile il percorso viene ottimizzato in base alla distanza.

## Cosa serve per realizzare la Fase 3

### Schema (partendo da quello canonico di Fase 0–2)

- Un solo `clienti.dart` e un solo `ordini.dart`, senza i file `*_table.dart` della Fase 3; restano `@DataClassName`, snapshot, `HH:mm`, `StatoOrdineConverter` e `setNull`.
- `clienti`: aggiungere `latitudine`, `longitudine` e `indirizzo_geocodificato`. Le coordinate in cache valgono solo se l'indirizzo attuale coincide con `indirizzo_geocodificato`.
- `ordini`: aggiungere `latitudine`, `longitudine`, `geocoding_stato` (con converter: `da_fare`, `ok`, `approssimato`, `fallito`), `geocoding_errore`, `geocoded_at` e `place_id`.
- `ristoranti`: aggiungere `latitudine`, `longitudine`, `sosta_minuti` (default 3) e la soglia «orario simile» in minuti (default proposto: 15). `maps_api_key`, `evidenzia_ritardi` e `tolleranza_ritardo_minuti` esistono già.
- Tabella `maps_usage` (mese, servizio, conteggio) per il contatore d'uso.
- Nulla è ancora distribuito: fondere tutto nello schema v1 e cancellare il database di sviluppo, oppure scrivere la migrazione con `if (from < 2)` (non `from == 1`).

### Server (isolate)

- `MapsService` vive solo nel server. La UI non chiama Google e non legge la chiave, salvo per caricare la mappa (vedi Flutter).
- **Routes API:** `POST routes.googleapis.com/directions/v2:computeRoutes`, chiave nell'header `X-Goog-Api-Key`, `X-Goog-FieldMask` obbligatorio (`routes.duration`, `routes.distanceMeters`, `routes.polyline.encodedPolyline`, `routes.legs.duration`, `routes.legs.distanceMeters`), `intermediates`, `travelMode: DRIVE`. Le durate sono stringhe tipo `420s`. Errori nel formato JSON standard Google.
- **Limiti:** Essentials copre fino a 10 tappe intermedie; oltre 10 si paga di più; il massimo è 25.
- **Route nuove:** `GET/PUT /api/v1/impostazioni`; `POST /api/v1/impostazioni/maps/verifica`; `POST /api/v1/ordini/<id>/geocodifica` (riprova o correzione indirizzo); `POST /api/v1/percorsi/calcola`; `GET /api/v1/impostazioni/mappa` (chiave e coordinate del ristorante per la WebView, con il token locale se adottato).
- **`POST /percorsi/calcola`:** riceve `ordine_ids`, ora di partenza e flag di ottimizzazione; restituisce l'ordine proposto, le tappe (`ordine_id`, ordine consigliato, ETA, durata e distanza della tratta, ritardo stimato), la polyline e i totali. Non salva nulla: tratte, ETA e polyline sono transitori, in linea con il divieto di cache dei risultati dei percorsi.
- **`POST /ordini`:** salva subito con `geocoding_stato = da_fare` (o riusa la cache) e geocodifica in background con `try/catch` completo. La UI interroga ogni pochi secondi finché ci sono ordini `da_fare`.
- **Geocoding:** `region=it`, `components=country:IT` e `bounds` attorno al ristorante. `ok` solo per `ROOFTOP` o `RANGE_INTERPOLATED` senza `partial_match`; altrimenti `approssimato`, con avviso e conferma della posizione.
- **Cambio indirizzo** (ordine o rubrica): coordinate e stato si azzerano e il geocoding riparte.
- **Eccezioni:** classe base non `sealed`. `testApiKey` restituisce un esito tipizzato (`ok`, `chiave_non_valida`, `api_non_abilitata`, `rete`) e prova Geocoding e Routes. La chiave non compare mai nei messaggi d'errore. `LatLng` valida con eccezioni, non con `assert`.

### Sottogruppi e ottimizzazione (ambito nuovo rispetto al piano)

- Sulla mappa compaiono gli ordini non consegnati, con l'orario richiesto nel marker o nel tooltip. Il ristoratore li seleziona e li raggruppa in sottogruppi (A, B…), ciascuno con colore e percorso propri; l'ora di partenza è per sottogruppo.
- **Ordine proposto:** ordinare gli ordini del sottogruppo per orario richiesto; raggruppare quelli a distanza oraria ≤ soglia; i raggruppamenti si susseguono in ordine cronologico e dentro ciascuno si ordina per distanza.
- **Come calcolare l'ordine ottimale:**
  - Opzione A: `optimizeWaypointOrder` di `computeRoutes`, una sola richiesta. Non ho trovato nella documentazione se faccia scattare lo SKU Pro: da verificare prima di adottarla.
  - Opzione B: solver locale (tutte le permutazioni fino a 8 tappe, euristica oltre) su distanze in linea d'aria, poi un solo `computeRoutes` per gli ETA. Costa meno ma stima male dove ci sono fiumi, ferrovie e sensi unici.
- L'ottimizzazione per distanza ignora l'orario dentro il gruppo: l'ETA va controllato dopo e i ritardi segnalati, non scartati.
- L'ordine proposto è un suggerimento con pulsante «Applica», che aggiorna `sequenza`: il drag & drop manuale di Fase 1 resta valido.
- Dopo un riordino o un cambio di stato il percorso è marcato «da ricalcolare».
- **ETA:** parte dall'ora di partenza scelta e somma tempi di tratta e sosta. Orario `HH:mm` combinato con la data del turno, con +1 giorno se l'orario è precedente all'inizio turno; orologio passato come parametro, per renderlo testabile.
- **Ritardo:** tenere separati `is_ritardo` (effettivo, Fase 1) e `ritardo_stimato_min`, che rispetta `evidenzia_ritardi` e la tolleranza. `in_ritardo` non torna come stato del marker: stato e flag si mostrano insieme (colore e anello o badge).

### Flutter

- **Pacchetto:** `webview_flutter_windows` (con `flutter pub add`), vincoli Flutter 3.44+ e Dart 3.12+; widget riscritto sul README del fork.
- **Mappa:**
  - HTML dall'asset, con la chiave presa da `GET /impostazioni/mappa` e caricato con `loadStringContent`, senza file temporaneo.
  - Fallimento di `initialize()` con messaggio esplicito (runtime WebView2 assente; l'installer include il bootstrapper Evergreen).
  - `gm_authFailure` ed errore di caricamento script rimandati a Flutter, che mostra il banner di AC-3.12.
  - InfoWindow unico con testo escapato (`textContent`).
  - `fitBounds` solo al primo caricamento e col pulsante «Centra»; marker aggiornati per differenza.
  - Numerazione `sequenza + 1`.
  - `google.maps.Marker` va bene per l'MVP.
- **Selezione:** stato nel provider, sincronizzato tra click sul marker e checkbox in lista; layout diviso lista | mappa nel dettaglio turno, con evidenziazione della riga.
- **Impostazioni:** chiave mascherata con avviso non bloccante sul formato e pulsante «Verifica»; indirizzo del ristorante con conferma sulla mappa; sosta; soglia «orario simile»; ritardi (tolleranza ed `evidenzia_ritardi` di Fase 1); contatore d'uso mensile.
- **Costi:** togliere il dialog di conferma a ogni calcolo (un calcolo costa 0,005 $ a 5 $/1000), sostituito dal contatore e da un tetto giornaliero.
- **DTO:** estendere `OrdineResponse` con coordinate e stato geocoding, senza `OrdineDto`; un solo `ApiClient`; niente `innerJoin`, il nome viene dallo snapshot.

### Guida alla chiave (testo per il ristoratore)

- API da abilitare: Geocoding, Routes e Maps JavaScript.
- Restrizioni applicazione: nessuna (le chiavi con referrer non funzionano con le chiamate REST dal server). Restrizione sulle tre API, quota giornaliera nella console Google e budget alert.

### Da correggere nel documento originale

- Il credito da 200 $ e i numeri «28.000 / 40.000» vanno sostituiti con le soglie per SKU.
- «Nessun dato dei clienti esce dal PC» va corretto: gli indirizzi vanno a Google; l'informativa del ristorante deve citarlo.
- Togliere il riferimento a Windows 7/8.
- Aggiornare AC-3.5 (Routes API) e AC-3.11 (ritardo stimato) e aggiungere test, con `http.Client` finto:
  - un secondo ordine allo stesso indirizzo non chiama Google, con indirizzo diverso sì;
  - geocoding `approssimato` e `ZERO_RESULTS`;
  - Routes: corpo e header corretti, parsing di `420s`, errore sopra 25 tappe e avviso sopra 10;
  - ETA a cavallo di mezzanotte;
  - ottimizzazione: ordine atteso su 5 punti noti;
  - la chiave non compare nei messaggi d'errore;
  - errore di `initialize()` della WebView.

## Rischi accettati

- **Coordinate persistenti oltre i 30 giorni** rispetto ai termini Google: scelta confermata. Non sono un legale e i termini possono cambiare. `geocoded_at` e `place_id` rendono banale passare più avanti a una scadenza a 30 giorni.
- ETA senza traffico.
- API del fork WebView da verificare; Flutter 3.44+ obbligatorio.
- L'ottimizzazione per distanza non considera l'orario dentro il gruppo.
- La stima di 5–7 giorni va rivista al rialzo: conviene dividere in **3a** (geocoding, impostazioni, mappa) e **3b** (percorsi e ottimizzazione).

## Decisioni aperte

- [ ] I sottogruppi restano una selezione di lavoro in memoria (proposta di partenza) oppure vanno salvati dopo il riavvio e resi assegnabili ai fattorini? Nel secondo caso serve una tabella dedicata.
- [ ] Ottimizzazione: `optimizeWaypointOrder` (verificare lo SKU) oppure solver locale.
- [ ] Valore della soglia «orario simile» (proposto: 15 minuti).

## Prossimo passo

Revisione della Fase 4.
