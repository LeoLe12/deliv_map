# Piano di sviluppo — App gestione consegne a domicilio per piccoli ristoranti

*Documento vivo: da aggiornare man mano che le decisioni vengono prese o riviste.*

## 1. Obiettivo del progetto

Applicazione pensata per piccoli ristoranti che gestiscono da soli le consegne a domicilio (senza piattaforme come Glovo/Deliveroo/Just Eat), per aiutarli a organizzare meglio le consegne tramite mappa, scheduling orario, calcolo tempistiche/fattibilità e anagrafica cliente (nome, telefono, importo).

## 2. Target e modello di business

- **Target**: piccoli imprenditori della ristorazione, 1-3 fattorini, oggi gestiscono le consegne a telefono/carta.
- **Modello di business**: vendita del software (licenza) più eventuali aggiornamenti. I costi operativi (es. API mappe) sono a carico del ristoratore che usa l'app.
- **Differenziazione rispetto ai competitor esistenti** (Routific, OptimoRoute, Track-POD, Onfleet, Shipday, Velocissimo, Deliverart): estrema semplicità, nessuna integrazione POS obbligatoria, costo di ingresso basso, zero training per l'uso quotidiano.

## 3. Scope MVP (v1)

### Attori nel v1

Un solo attore: **il gestore/titolare del locale**, che usa l'app su un computer nel ristorante. Nessuna app fattorino nel v1 — le consegne restano assegnate e comunicate come oggi (verbalmente/telefonicamente), ma organizzate tramite l'app.

### Funzionalità incluse nel v1

- Login gestore
- Inserimento nuovo ordine: nome cliente, telefono, indirizzo, importo, note, orario richiesto
- Visualizzazione mappa con i marker degli ordini del turno
- Calcolo del tempo di percorrenza tra i punti di consegna (anche solo in sequenza, non necessariamente ottimizzato)
- Sequenziamento manuale degli ordini (riordino lista, es. drag & drop)
- Stato dell'ordine: in attesa / in consegna / consegnato / in ritardo
- Impostazioni: chiave API mappe (BYOK, inserita dal ristoratore), zona di consegna, orari del locale
- Backup locale dei dati (export manuale o automatico)

### Funzionalità rimandate a v2/v3

- App mobile per il fattorino (che referenzia lo stesso database/API, via LAN)
- Ottimizzazione automatica del percorso (algoritmo VRP con finestre orarie, es. Google OR-Tools)
- Multi-fattorino con bilanciamento automatico del carico
- Tracking GPS in tempo reale
- Notifiche automatiche al cliente finale (SMS/WhatsApp)
- Storico cliente e statistiche (tempo medio di consegna, incassi per fascia oraria)
- Eventuale sincronizzazione cloud multi-sede

## 4. Architettura tecnica

### Principio guida: local-first

Nel v1 tutto gira su un solo PC, nel locale del ristorante. Nessun dato cliente transita su server esterni — il ristorante resta l'unico titolare del trattamento dati (vedi sezione 5). Questo semplifica di molto anche il modello di business: nessuna infrastruttura cloud da mantenere, nessun costo di hosting a carico del venditore del software.

### Stack tecnologico proposto

| Livello | Scelta | Motivazione |
|---|---|---|
| Interfaccia | Flutter Desktop | Un solo codebase, riusabile in futuro anche per mobile |
| API/logica | Dart (pacchetto `shelf`), server locale su `localhost` | Stesso linguaggio di UI e futura app fattorino |
| Database | SQLite | Zero configurazione, file singolo, backup semplice |
| Accesso dati | Repository pattern (Dart) | Disaccoppia UI e futuro fattorino dalla scelta di storage |
| Mappa e percorsi | Google Maps API, chiave inserita dal ristoratore (BYOK) | Coerente col modello "costi a carico dell'utente"; nessun bisogno di billing multi-tenant lato venditore |

### Struttura del backend locale (tre livelli)

1. **Interfaccia utente** (Flutter Desktop) — solo presentazione, non accede mai direttamente al database
2. **API locale** (server Dart `shelf`, in ascolto su `localhost`) — contiene tutta la logica di business (creazione ordine, sequenziamento, validazione orari)
3. **Database** (SQLite, singolo file locale) — accesso mediato da repository dedicati

Punto di estensione futuro: quando arriverà l'app fattorino, il server smette di ascoltare solo `localhost` e inizia ad ascoltare anche sull'IP della rete locale (LAN) del ristorante — stessa logica, stessi endpoint, nessuna riscrittura.

### Modello dati essenziale (v1)

- **Ristorante**: id, nome, indirizzo, zona di consegna, chiave API mappe
- **Utente**: id, ruolo (gestore), credenziali
- **Cliente**: nome, telefono, indirizzo
- **Ordine**: cliente, importo, orario richiesto, stato, posizione nella sequenza, note
- **Turno**: data, ordini associati

## 5. Considerazioni privacy/GDPR

- Il ristorante è l'unico **titolare del trattamento** dei dati dei propri clienti (nome, telefono, indirizzo, importo ordine) — l'architettura local-first evita che il venditore del software diventi responsabile del trattamento (data processor), a differenza di un'ipotesi cloud/SaaS multi-tenant.
- Base giuridica: esecuzione del servizio richiesto dal cliente che ordina (art. 6.1.b GDPR) — non serve consenso esplicito per il trattamento minimo necessario.
- Da garantire lato ristorante (con supporto dell'app dove possibile): informativa privacy minima, minimizzazione dei dati raccolti, policy di retention (per quanto tempo tenere gli ordini), sicurezza di accesso al PC (password, aggiornamenti), possibilità di cancellare i dati di un cliente su richiesta.
- Se in futuro si passasse a un backend cloud condiviso, andrebbero rivalutati: DPA con ogni ristorante cliente, hosting in UE, gestione data breach, registro dei trattamenti.

## 6. Roadmap

### V1 — MVP locale (gestore)
Vedi sezione 3. Obiettivo: validare che i ristoratori target adottano davvero lo strumento al posto di telefono/carta.

### V2 — Ottimizzazione e app fattorino
- App mobile fattorino (stesso backend, esposto via LAN)
- Autenticazione/pairing fattorino-gestore (es. PIN generato dal gestore)
- Ottimizzazione automatica del percorso (OR-Tools o motore equivalente)
- Multi-fattorino con bilanciamento del carico

### V3 — Eventuale evoluzione cloud
- Tracking GPS in tempo reale, notifiche cliente, statistiche
- Da valutare solo se emerge domanda reale per gestione multi-sede o accesso da remoto

## 7. Decisioni aperte / da validare

- [ ] Formato esatto del backup automatico (frequenza, destinazione)
- [ ] Meccanismo di pairing fattorino-gestore in LAN (v2)
- [ ] Se offrire, oltre a BYOK, anche un'opzione "chiave gestita" a pagamento per chi non vuole configurare Google Cloud da solo
- [ ] Prezzo e struttura della licenza (una tantum vs abbonamento aggiornamenti)
