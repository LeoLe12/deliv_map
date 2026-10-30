# Fase 1 — Gestione Turni e Ordini

> **Stato di avanzamento**: Fase Core di Sviluppo  
> **Durata stimata**: 5 - 7 giorni lavorativi  
> **Piattaforma target**: Flutter Desktop (Windows prioritario)  
> **Architettura**: Flutter Desktop UI ➔ Dart Shelf API (localhost:8080) ➔ Drift ORM / SQLite

---

## Panoramica della Fase 1

La Fase 1 costituisce il nucleo operativo essenziale di **DelivMap**. In questa fase viene implementata l'infrastruttura completa per consentire al ristoratore di:
1. Creare e monitorare sessioni di servizio (**Turni**, es. pranzo/cena).
2. Registrare rapidamente gli ordini di consegna con i relativi dati cliente (**Ordini** e **Clienti**), congelando i dati di recapito tramite **snapshot** immutabile.
3. Visualizzare l'elenco sequenziale delle consegne con ordinamento manuale tramite Drag & Drop desktop (`ReorderableListView`).
4. Aggiornare lo stato operativo reale delle consegne (`in_attesa`, `in_consegna`, `consegnato`).
5. Rilevare dinamicamente le consegne in ritardo con soglia di tolleranza di 5 minuti (configurabile e ignorabile dalle impostazioni).

L'architettura rispetta il principio **local-first**: la UI Flutter comunica esclusivamente tramite richieste HTTP REST con un server Dart `shelf` in ascolto su `http://127.0.0.1:8080`. Il server locale gestisce la logica di business e la persistenza transazionale su SQLite tramite **Drift**, operando all'interno di un Isolate di background.

---

## Decisioni di Revisione Applicate

1. **Schema Dati Canonico e Unificato**:
   - Eliminate le definizioni duplicate `*_table.dart`. Si utilizzano esclusivamente le tabelle canoniche definite in `lib/data/tables/` (`turni.dart`, `ordini.dart`, `clienti.dart`).
2. **Snapshot dell'Ordine (Immutabilità Storica)**:
   - La tabella `ordini` memorizza direttamente `nomeClienteConsegna`, `telefonoConsegna` e `indirizzoConsegna`. La tabella `clienti` funge da rubrica rapida per autocompletamento. Modifiche future ai clienti non alterano mai retroattivamente gli ordini passati.
3. **Macchina a Stati Pura ed Eliminazione di `in_ritardo` come Stato Scrivibile**:
   - Lo `StatoOrdine` accetta unicamente i tre stati reali del flusso: `in_attesa`, `in_consegna`, `consegnato`.
   - Il ritardo è un **flag booleano calcolato dinamicamente** confrontando l'orario richiesto con l'orario attuale.
   - **Tolleranza di 5 minuti configurabile**: l'indicatore di ritardo scatta solo se l'orario attuale supera la deadline di oltre 5 minuti (valore configurabile nelle impostazioni).
   - **Ignorabile da impostazioni**: un'opzione nelle impostazioni (`evidenzia_ritardi: bool`) consente di disattivare completamente la visualizzazione del ritardo.
4. **Preservazione della Type-Safety con `StatoOrdineConverter`**:
   - Nel DB SQLite viene scritta la stringa `snake_case`, mentre nel codice Dart si opera con l'enum tipizzato `StatoOrdine`.
5. **Formato Orario di Consegna**:
   - `orarioRichiesto` è memorizzato come stringa formattata `HH:mm` (es. `"20:30"`), validata via regex e combinata con la data del turno per i calcoli temporali.
6. **Inizializzazione Locale Italiano (`intl`)**:
   - Configurazione di `initializeDateFormatting('it_IT', null)` per la corretta formattazione in lingua italiana di date e valute senza eccezioni a runtime.
7. **Ottimizzazione Query Turni (Risoluzione N+1)**:
   - Il conteggio ordini e l'incasso totale di ciascun turno vengono aggregati con un'unica query SQL `LEFT JOIN` e `GROUP BY`.

---

## Schema Dati di Riferimento (Drift / SQLite)

Si utilizzano esclusivamente le tabelle canoniche definite in Fase 0 in `lib/data/tables/`, senza file duplicati `*_table.dart`:

```dart
// lib/data/tables/turni.dart
import 'package:drift/drift.dart';

@DataClassName('Turno')
class Turni extends Table {
  IntColumn get id => integer().autoIncrement()();
  DateTimeColumn get data => dateTime()();
  TextColumn get note => text().nullable()();
}
```

```dart
// lib/data/tables/clienti.dart
import 'package:drift/drift.dart';

/// Tabella anagrafica clienti (rubrica rapida per autocompletamento)
@DataClassName('Cliente')
@TableIndex(name: 'idx_clienti_telefono', columns: {#telefono})
class Clienti extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get nome => text().withLength(min: 1, max: 100)();
  TextColumn get telefono => text().withLength(min: 5, max: 25)();
  TextColumn get indirizzo => text()();
}
```

```dart
// lib/data/tables/ordini.dart
import 'package:drift/drift.dart';
import '../../models/stato_ordine.dart';
import '../converters/stato_ordine_converter.dart';
import 'clienti.dart';
import 'turni.dart';

@DataClassName('Ordine')
@TableIndex(name: 'idx_ordini_turno', columns: {#turnoId})
class Ordini extends Table {
  IntColumn get id => integer().autoIncrement()();

  /// Riferimento anagrafica opzionale (setNull se il cliente in rubrica viene rimosso)
  IntColumn get clienteId => integer()
      .named('cliente_id')
      .nullable()
      .references(Clienti, #id, onDelete: KeyAction.setNull)();

  /// Riferimento al turno operativo
  IntColumn get turnoId => integer()
      .named('turno_id')
      .references(Turni, #id, onDelete: KeyAction.cascade)();

  /// Snapshot anagrafico congelato al momento dell'ordine
  TextColumn get nomeClienteConsegna => text().named('nome_cliente_consegna').withLength(min: 1, max: 120)();
  TextColumn get telefonoConsegna => text().named('telefono_consegna').withLength(min: 3, max: 30)();
  TextColumn get indirizzoConsegna => text().named('indirizzo_consegna').withLength(min: 3, max: 250)();

  /// Importo monetario dell'ordine (in euro, es. 24.50)
  RealColumn get importo => real()();

  /// Orario richiesto dal cliente per la consegna nel formato HH:mm (es. '20:30')
  TextColumn get orarioRichiesto => text().named('orario_richiesto')();

  /// Stato di avanzamento (in_attesa, in_consegna, consegnato) con StatoOrdineConverter
  TextColumn get stato => text().map(const StatoOrdineConverter())();

  /// Posizione sequenziale nel turno (0, 1, 2, ...) per riordino manuale
  IntColumn get sequenza => integer().withDefault(const Constant(0))();

  /// Note aggiuntive (es. citofono, scala, piano)
  TextColumn get note => text().nullable()();
}
```

---
## 1.1 — DAO e Repository per Turni

Il DAO dei turni gestisce la persistenza del modello `Turni`, il calcolo degli aggregati di sintesi e le query reattive tramite `Stream` di Drift.

### Codice Completo: `TurniDao`

```dart
// lib/data/daos/turni_dao.dart
import 'package:drift/drift.dart';
import '../database.dart';
import '../tables/turni.dart';
import '../tables/ordini.dart';

part 'turni_dao.g.dart';

class TurnoConStatistiche {
  final Turno turno;
  final int totaleOrdini;
  final double incassoTotale;

  TurnoConStatistiche({
    required this.turno,
    required this.totaleOrdini,
    required this.incassoTotale,
  });
}

@DriftAccessor(tables: [Turni, Ordini])
class TurniDao extends DatabaseAccessor<AppDatabase> with _$TurniDaoMixin {
  TurniDao(AppDatabase db) : super(db);

  /// Crea un nuovo turno nel database
  Future<int> creaTurno(TurniCompanion turno) {
    return into(turni).insert(turno);
  }

  /// Recupera l'elenco paginato dei turni ordinati per data decrescente con filtri opzionali
  Future<List<Turno>> getTurni({
    int limit = 20,
    int offset = 0,
    DateTime? dal,
    DateTime? al,
  }) {
    final query = select(turni)
      ..orderBy([(t) => OrderingTerm.desc(t.data)])
      ..limit(limit, offset: offset);

    if (dal != null) {
      query.where((t) => t.data.isBiggerOrEqualValue(dal));
    }
    if (al != null) {
      query.where((t) => t.data.isSmallerOrEqualValue(al));
    }

    return query.get();
  }

  /// Recupera un turno tramite ID
  Future<Turno?> getTurnoById(int id) {
    return (select(turni)..where((t) => t.id.equals(id))).getSingleOrNull();
  }

  /// Aggiorna i dati del turno
  Future<bool> aggiornaTurno(Turno turno) {
    return update(turni).replace(turno);
  }

  /// Elimina un turno e, per effetto CASCADE, gli ordini associati
  Future<int> eliminaTurno(int id) {
    return (delete(turni)..where((t) => t.id.equals(id))).go();
  }

  /// Stream reattivo per osservare i turni recenti con paginazione
  Stream<List<Turno>> watchTurniRecenti({int limit = 20}) {
    return (select(turni)
          ..orderBy([(t) => OrderingTerm.desc(t.data)])
          ..limit(limit))
        .watch();
  }

  /// Recupera i turni compresi nella giornata odierna
  Future<List<Turno>> getTurniOggi() {
    final now = DateTime.now();
    final inizioGiorno = DateTime(now.year, now.month, now.day, 0, 0, 0);
    final fineGiorno = DateTime(now.year, now.month, now.day, 23, 59, 59);

    return (select(turni)
          ..where((t) => t.data.isBetweenValues(inizioGiorno, fineGiorno))
          ..orderBy([(t) => OrderingTerm.desc(t.data)]))
        .get();
  }

  /// Stream reattivo per i turni degli ultimi 7 giorni
  Stream<List<Turno>> watchTurniUltimi7Giorni() {
    final now = DateTime.now();
    final setteGiorniFa = now.subtract(const Duration(days: 7));
    final limiteIniziale = DateTime(setteGiorniFa.year, setteGiorniFa.month, setteGiorniFa.day, 0, 0, 0);

    return (select(turni)
          ..where((t) => t.data.isBiggerOrEqualValue(limiteIniziale))
          ..orderBy([(t) => OrderingTerm.desc(t.data)]))
        .watch();
  }

  /// Recupera statistiche aggregate di un turno con singola query LEFT JOIN (risoluzione N+1)
  Future<TurnoConStatistiche?> getStatisticheTurno(int turnoId) async {
    final countOrdini = ordini.id.count();
    final sumIncasso = ordini.importo.sum();
    final query = select(turni).join([
      leftOuterJoin(ordini, ordini.turnoId.equalsExp(turni.id)),
    ])
      ..where(turni.id.equals(turnoId))
      ..groupBy([turni.id]);

    // In Drift addColumns restituisce void: va invocato prima di await query.get()
    query.addColumns([countOrdini, sumIncasso]);
    final rows = await query.get();
    if (rows.isEmpty) return null;

    final row = rows.first;
    return TurnoConStatistiche(
      turno: row.readTable(turni),
      totaleOrdini: row.read(countOrdini) ?? 0,
      incassoTotale: row.read(sumIncasso) ?? 0.0,
    );
  }

  /// Recupera elenco turni con statistiche aggregate in un'unica query SQL (elimina N+1 su dashboard)
  Future<List<TurnoConStatistiche>> getTurniConStatistiche({
    int limit = 20,
    int offset = 0,
    DateTime? dal,
    DateTime? al,
  }) async {
    final countOrdini = ordini.id.count();
    final sumIncasso = ordini.importo.sum();
    final query = select(turni).join([
      leftOuterJoin(ordini, ordini.turnoId.equalsExp(turni.id)),
    ])
      ..orderBy([OrderingTerm.desc(turni.data)])
      ..groupBy([turni.id])
      ..limit(limit, offset: offset);

    if (dal != null) {
      query.where(turni.data.isBiggerOrEqualValue(dal));
    }
    if (al != null) {
      query.where(turni.data.isSmallerOrEqualValue(al));
    }

    query.addColumns([countOrdini, sumIncasso]);
    final rows = await query.get();

    return rows.map((row) {
      return TurnoConStatistiche(
        turno: row.readTable(turni),
        totaleOrdini: row.read(countOrdini) ?? 0,
        incassoTotale: row.read(sumIncasso) ?? 0.0,
      );
    }).toList();
  }
}
```

### Codice Completo: `TurniRepository`

```dart
// lib/data/repositories/turni_repository.dart
import '../daos/turni_dao.dart';
import '../database.dart';

class TurniRepository {
  final TurniDao _turniDao;

  TurniRepository(this._turniDao);

  Future<int> creaTurno({required DateTime data, String? note}) {
    return _turniDao.creaTurno(
      TurniCompanion.insert(
        data: data,
        note: Value(note),
      ),
    );
  }

  Future<List<Turno>> getTurni({
    int limit = 20,
    int offset = 0,
    DateTime? dal,
    DateTime? al,
  }) {
    return _turniDao.getTurni(limit: limit, offset: offset, dal: dal, al: al);
  }

  Future<Turno?> getTurnoById(int id) {
    return _turniDao.getTurnoById(id);
  }

  Future<bool> aggiornaTurno({
    required int id,
    required DateTime data,
    String? note,
  }) {
    return _turniDao.aggiornaTurno(
      Turno(id: id, data: data, note: note),
    );
  }

  Future<bool> eliminaTurno(int id) async {
    final rows = await _turniDao.eliminaTurno(id);
    return rows > 0;
  }

  Stream<List<Turno>> watchTurniRecenti({int limit = 20}) {
    return _turniDao.watchTurniRecenti(limit: limit);
  }

  Future<List<Turno>> getTurniOggi() {
    return _turniDao.getTurniOggi();
  }

  Stream<List<Turno>> watchTurniUltimi7Giorni() {
    return _turniDao.watchTurniUltimi7Giorni();
  }

  Future<TurnoConStatistiche?> getStatisticheTurno(int turnoId) {
    return _turniDao.getStatisticheTurno(turnoId);
  }

  Future<List<TurnoConStatistiche>> getTurniConStatistiche({
    int limit = 20,
    int offset = 0,
    DateTime? dal,
    DateTime? al,
  }) {
    return _turniDao.getTurniConStatistiche(limit: limit, offset: offset, dal: dal, al: al);
  }
}
```

---

## 1.2 — DAO e Repository per Ordini

Il DAO degli ordini deve gestire la relazione 1-a-N tra Clienti e Ordini. Durante la creazione dell'ordine, se il cliente esiste già (riconosciuto tramite numero di telefono indicizzato), viene riutilizzato o aggiornato; in caso contrario, viene inserita una nuova anagrafica cliente in un'unica transazione.

### Modello di Join: `OrdineConCliente`

```dart
// lib/data/models/ordine_con_cliente.dart
import '../database.dart';
import '../../models/ordine_dtos.dart';

class OrdineConCliente {
  final Ordine ordine;
  final Cliente cliente;

  OrdineConCliente({
    required this.ordine,
    required this.cliente,
  });

  /// Mappa l'entità SQLite nel DTO REST utilizzando i campi snapshot immutabili dell'ordine
  OrdineResponse toResponseDto({bool isInRitardo = false}) {
    return OrdineResponse(
      id: ordine.id,
      turnoId: ordine.turnoId,
      cliente: ClienteDto(
        id: cliente.id,
        nome: ordine.nomeClienteConsegna,
        telefono: ordine.telefonoConsegna,
        indirizzo: ordine.indirizzoConsegna,
      ),
      importo: ordine.importo,
      orarioRichiesto: ordine.orarioRichiesto,
      stato: ordine.stato.codice,
      sequenza: ordine.sequenza,
      note: ordine.note,
      isInRitardo: isInRitardo,
    );
  }
}
```

### Codice Completo: `OrdiniDao`

```dart
// lib/data/daos/ordini_dao.dart
import 'package:drift/drift.dart';
import '../database.dart';
import '../../models/stato_ordine.dart';
import '../tables/ordini.dart';
import '../tables/clienti.dart';
import '../tables/turni.dart';
import '../models/ordine_con_cliente.dart';

part 'ordini_dao.g.dart';

class ElementoSequenzaBatch {
  final int ordineId;
  final int nuovaSequenza;

  ElementoSequenzaBatch({required this.ordineId, required this.nuovaSequenza});
}

@DriftAccessor(tables: [Ordini, Clienti, Turni])
class OrdiniDao extends DatabaseAccessor<AppDatabase> with _$OrdiniDaoMixin {
  OrdiniDao(AppDatabase db) : super(db);

  /// Crea un ordine gestendo la lookup o creazione dell'anagrafica cliente in transazione
  Future<int> creaOrdineTransazionale({
    required int turnoId,
    required String nomeCliente,
    required String telefonoCliente,
    required String indirizzoCliente,
    required double importo,
    required String orarioRichiesto,
    String? note,
  }) {
    return transaction(() async {
      // 1. Verifica esistenza cliente per telefono
      final clienteEsistente = await (select(clienti)
            ..where((c) => c.telefono.equals(telefonoCliente.trim())))
          .getSingleOrNull();

      int clienteId;
      if (clienteEsistente != null) {
        clienteId = clienteEsistente.id;
        // Aggiorna nome o indirizzo se variati
        await update(clienti).replace(
          clienteEsistente.copyWith(
            nome: nomeCliente.trim(),
            indirizzo: indirizzoCliente.trim(),
          ),
        );
      } else {
        clienteId = await into(clienti).insert(
          ClientiCompanion.insert(
            nome: nomeCliente.trim(),
            telefono: telefonoCliente.trim(),
            indirizzo: indirizzoCliente.trim(),
          ),
        );
      }

      // 2. Calcola prossima sequenza disponibile nel turno
      final maxSequenzaQuery = selectOnly(ordini)
        ..addColumns([ordini.sequenza.max()])
        ..where(ordini.turnoId.equals(turnoId));
      final maxSequenza = await maxSequenzaQuery.map((row) => row.read(ordini.sequenza.max())).getSingleOrNull() ?? 0;
      final nuovaSequenza = maxSequenza + 1;

      // 3. Inserisce l'ordine con snapshot cliente immutabile e stato tipizzato
      return into(ordini).insert(
        OrdiniCompanion.insert(
          clienteId: Value(clienteId),
          turnoId: turnoId,
          nomeClienteConsegna: nomeCliente.trim(),
          telefonoConsegna: telefonoCliente.trim(),
          indirizzoConsegna: indirizzoCliente.trim(),
          importo: importo,
          orarioRichiesto: orarioRichiesto,
          stato: StatoOrdine.inAttesa,
          sequenza: Value(nuovaSequenza),
          note: Value(note),
        ),
      );
    });
  }

  /// Recupera tutti gli ordini di un turno con join cliente, ordinati per sequenza crescente
  Future<List<OrdineConCliente>> getOrdiniByTurno(int turnoId) {
    final query = select(ordini).join([
      innerJoin(clienti, clienti.id.equalsExp(ordini.clienteId)),
    ])
      ..where(ordini.turnoId.equals(turnoId))
      ..orderBy([OrderingTerm.asc(ordini.sequenza)]);

    return query.map((row) {
      return OrdineConCliente(
        ordine: row.readTable(ordini),
        cliente: row.readTable(clienti),
      );
    }).get();
  }

  /// Stream reattivo degli ordini di un turno
  Stream<List<OrdineConCliente>> watchOrdiniByTurno(int turnoId) {
    final query = select(ordini).join([
      innerJoin(clienti, clienti.id.equalsExp(ordini.clienteId)),
    ])
      ..where(ordini.turnoId.equals(turnoId))
      ..orderBy([OrderingTerm.asc(ordini.sequenza)]);

    return query.watch().map((rows) {
      return rows.map((row) {
        return OrdineConCliente(
          ordine: row.readTable(ordini),
          cliente: row.readTable(clienti),
        );
      }).toList();
    });
  }

  /// Recupera un singolo ordine tramite ID con dati cliente associati
  Future<OrdineConCliente?> getOrdineById(int id) {
    final query = select(ordini).join([
      innerJoin(clienti, clienti.id.equalsExp(ordini.clienteId)),
    ])..where(ordini.id.equals(id));

    return query.map((row) {
      return OrdineConCliente(
        ordine: row.readTable(ordini),
        cliente: row.readTable(clienti),
      );
    }).getSingleOrNull();
  }

  /// Aggiorna i dati anagrafici e gestionali di un ordine
  Future<void> aggiornaOrdineCompleto({
    required int ordineId,
    required String nomeCliente,
    required String telefonoCliente,
    required String indirizzoCliente,
    required double importo,
    required String orarioRichiesto,
    String? note,
  }) {
    return transaction(() async {
      final ordine = await (select(ordini)..where((o) => o.id.equals(ordineId))).getSingleOrNull();
      if (ordine == null) throw Exception('Ordine $ordineId non trovato');

      // Aggiorna anagrafica cliente
      await (update(clienti)..where((c) => c.id.equals(ordine.clienteId))).write(
        ClientiCompanion(
          nome: Value(nomeCliente.trim()),
          telefono: Value(telefonoCliente.trim()),
          indirizzo: Value(indirizzoCliente.trim()),
        ),
      );

      // Aggiorna ordine
      await (update(ordini)..where((o) => o.id.equals(ordineId))).write(
        OrdiniCompanion(
          importo: Value(importo),
          orarioRichiesto: Value(orarioRichiesto),
          note: Value(note),
        ),
      );
    });
  }

  /// Aggiorna lo stato di un ordine
  Future<bool> aggiornaStato(int ordineId, String nuovoStato) async {
    final righe = await (update(ordini)..where((o) => o.id.equals(ordineId))).write(
      OrdiniCompanion(
        stato: Value(nuovoStato),
      ),
    );
    return righe > 0;
  }

  /// Aggiorna la sequenza in blocco all'interno di una transazione SQLite
  Future<void> aggiornaSequenzaBatch(List<ElementoSequenzaBatch> items) {
    return transaction(() async {
      for (final item in items) {
        await (update(ordini)..where((o) => o.id.equals(item.ordineId))).write(
          OrdiniCompanion(
            sequenza: Value(item.nuovaSequenza),
          ),
        );
      }
    });
  }

  /// Elimina un ordine e ricalcola la sequenza degli ordini successivi
  Future<bool> eliminaOrdine(int ordineId) {
    return transaction(() async {
      final ordine = await (select(ordini)..where((o) => o.id.equals(ordineId))).getSingleOrNull();
      if (ordine == null) return false;

      final turnoId = ordine.turnoId;
      final sequenzaEliminata = ordine.sequenza;

      // 1. Elimina l'ordine
      await (delete(ordini)..where((o) => o.id.equals(ordineId))).go();

      // 2. Scala di 1 la sequenza degli ordini con sequenza superiore
      final ordiniSuccessivi = await (select(ordini)
            ..where((o) => o.turnoId.equals(turnoId) & o.sequenza.isBiggerThanValue(sequenzaEliminata))
            ..orderBy([(o) => OrderingTerm.asc(o.sequenza)]))
          .get();

      for (final o in ordiniSuccessivi) {
        await (update(ordini)..where((row) => row.id.equals(o.id))).write(
          OrdiniCompanion(
            sequenza: Value(o.sequenza - 1),
          ),
        );
      }

      return true;
    });
  }
}
```

### Codice Completo: `OrdiniRepository`

```dart
// lib/data/repositories/ordini_repository.dart
import '../daos/ordini_dao.dart';
import '../models/ordine_con_cliente.dart';

class OrdiniRepository {
  final OrdiniDao _ordiniDao;

  OrdiniRepository(this._ordiniDao);

  Future<int> creaOrdine({
    required int turnoId,
    required String nomeCliente,
    required String telefonoCliente,
    required String indirizzoCliente,
    required double importo,
    required String orarioRichiesto,
    String? note,
  }) {
    return _ordiniDao.creaOrdineTransazionale(
      turnoId: turnoId,
      nomeCliente: nomeCliente,
      telefonoCliente: telefonoCliente,
      indirizzoCliente: indirizzoCliente,
      importo: importo,
      orarioRichiesto: orarioRichiesto,
      note: note,
    );
  }

  Future<List<OrdineConCliente>> getOrdiniByTurno(int turnoId) {
    return _ordiniDao.getOrdiniByTurno(turnoId);
  }

  Stream<List<OrdineConCliente>> watchOrdiniByTurno(int turnoId) {
    return _ordiniDao.watchOrdiniByTurno(turnoId);
  }

  Future<OrdineConCliente?> getOrdineById(int id) {
    return _ordiniDao.getOrdineById(id);
  }

  Future<void> aggiornaOrdine({
    required int ordineId,
    required String nomeCliente,
    required String telefonoCliente,
    required String indirizzoCliente,
    required double importo,
    required String orarioRichiesto,
    String? note,
  }) {
    return _ordiniDao.aggiornaOrdineCompleto(
      ordineId: ordineId,
      nomeCliente: nomeCliente,
      telefonoCliente: telefonoCliente,
      indirizzoCliente: indirizzoCliente,
      importo: importo,
      orarioRichiesto: orarioRichiesto,
      note: note,
    );
  }

  Future<bool> aggiornaStato(int ordineId, String nuovoStato) {
    return _ordiniDao.aggiornaStato(ordineId, nuovoStato);
  }

  Future<void> aggiornaSequenzaBatch(List<ElementoSequenzaBatch> items) {
    return _ordiniDao.aggiornaSequenzaBatch(items);
  }

  Future<bool> eliminaOrdine(int ordineId) {
    return _ordiniDao.eliminaOrdine(ordineId);
  }
}
```

---

## 1.3 — API Routes per Turni (Dart Shelf)

Le rotte REST sono esposte dal pacchetto `shelf_router` e restituiscono payload JSON standardizzati con codici HTTP conformi.

### Struttura Middleware di Risposta JSON
Ogni risposta include l'header `content-type: application/json; charset=utf-8`.

### Codice Completo: `TurniApi`

```dart
// lib/api/routes/turni_routes.dart
import 'dart:convert';
import 'package:shelf/shelf.dart';
import 'package:shelf_router/shelf_router.dart';
import '../../data/repositories/turni_repository.dart';
import '../../data/repositories/ordini_repository.dart';
import '../../models/turno_dtos.dart';

class TurniApi {
  final TurniRepository turniRepo;
  final OrdiniRepository ordiniRepo;

  TurniApi({required this.turniRepo, required this.ordiniRepo});

  Router get router {
    final app = Router();

    // GET /api/v1/turni - Elenco turni con filtri e paginazione
    app.get('/', (Request request) async {
      try {
        final queryParams = request.url.queryParameters;
        final filtro = queryParams['filtro'];
        final limit = int.tryParse(queryParams['limit'] ?? '20') ?? 20;
        final offset = int.tryParse(queryParams['offset'] ?? '0') ?? 0;

        DateTime? dal;
        DateTime? al;

        if (filtro == 'oggi') {
          final now = DateTime.now();
          dal = DateTime(now.year, now.month, now.day, 0, 0, 0);
          al = DateTime(now.year, now.month, now.day, 23, 59, 59);
        } else if (filtro == 'settimana') {
          final now = DateTime.now();
          final start = now.subtract(const Duration(days: 7));
          dal = DateTime(start.year, start.month, start.day, 0, 0, 0);
          al = DateTime(now.year, now.month, now.day, 23, 59, 59);
        } else {
          if (queryParams.containsKey('dal')) {
            dal = DateTime.tryParse(queryParams['dal']!);
          }
          if (queryParams.containsKey('al')) {
            al = DateTime.tryParse(queryParams['al']!);
          }
        }

        final turniList = await turniRepo.getTurni(
          limit: limit,
          offset: offset,
          dal: dal,
          al: al,
        );

        final List<Map<String, dynamic>> responseList = [];
        for (final t in turniList) {
          final stats = await turniRepo.getStatisticheTurno(t.id);
          responseList.add(TurnoResponse(
            id: t.id,
            data: t.data,
            note: t.note,
            totaleOrdini: stats?.totaleOrdini ?? 0,
            incassoTotale: stats?.incassoTotale ?? 0.0,
          ).toJson());
        }

        return Response.ok(
          jsonEncode({'turni': responseList}),
          headers: {'content-type': 'application/json'},
        );
      } catch (e) {
        return Response.internalServerError(
          body: jsonEncode({'errore': 'Impossibile recuperare i turni', 'dettaglio': e.toString()}),
          headers: {'content-type': 'application/json'},
        );
      }
    });

    // POST /api/v1/turni - Creazione nuovo turno
    app.post('/', (Request request) async {
      try {
        final bodyString = await request.readAsString();
        if (bodyString.isEmpty) {
          return Response(400, body: jsonEncode({'errore': 'Body JSON mancante'}), headers: {'content-type': 'application/json'});
        }

        final Map<String, dynamic> json = jsonDecode(bodyString);
        final turnoReq = TurnoRequest.fromJson(json);

        final idGenerato = await turniRepo.creaTurno(
          data: turnoReq.data,
          note: turnoReq.note,
        );

        final creato = await turniRepo.getTurnoById(idGenerato);
        if (creato == null) {
          return Response.internalServerError(
            body: jsonEncode({'errore': 'Errore durante la creazione del turno'}),
            headers: {'content-type': 'application/json'},
          );
        }

        final responseDto = TurnoResponse(
          id: creato.id,
          data: creato.data,
          note: creato.note,
          totaleOrdini: 0,
          incassoTotale: 0.0,
        );

        return Response(
          201,
          body: jsonEncode(responseDto.toJson()),
          headers: {'content-type': 'application/json'},
        );
      } on FormatException catch (e) {
        return Response(400, body: jsonEncode({'errore': 'JSON non valido', 'dettaglio': e.message}), headers: {'content-type': 'application/json'});
      } catch (e) {
        return Response.internalServerError(
          body: jsonEncode({'errore': 'Errore interno del server', 'dettaglio': e.toString()}),
          headers: {'content-type': 'application/json'},
        );
      }
    });

    // GET /api/v1/turni/<id> - Dettaglio turno con lista ordini
    app.get('/<id|[0-9]+>', (Request request, String id) async {
      try {
        final turnoId = int.parse(id);
        final turno = await turniRepo.getTurnoById(turnoId);
        if (turno == null) {
          return Response.notFound(
            jsonEncode({'errore': 'Turno $turnoId non trovato'}),
            headers: {'content-type': 'application/json'},
          );
        }

        final stats = await turniRepo.getStatisticheTurno(turnoId);
        final ordiniConCliente = await ordiniRepo.getOrdiniByTurno(turnoId);

        final ordiniResponseList = ordiniConCliente.map((item) {
          return item.toResponseDto();
        }).toList();

        final responseDto = TurnoDettaglioResponse(
          id: turno.id,
          data: turno.data,
          note: turno.note,
          totaleOrdini: stats?.totaleOrdini ?? 0,
          incassoTotale: stats?.incassoTotale ?? 0.0,
          ordini: ordiniResponseList,
        );

        return Response.ok(
          jsonEncode(responseDto.toJson()),
          headers: {'content-type': 'application/json'},
        );
      } catch (e) {
        return Response.internalServerError(
          body: jsonEncode({'errore': 'Errore recupero dettaglio turno', 'dettaglio': e.toString()}),
          headers: {'content-type': 'application/json'},
        );
      }
    });

    // PUT /api/v1/turni/<id> - Aggiorna turno
    app.put('/<id|[0-9]+>', (Request request, String id) async {
      try {
        final turnoId = int.parse(id);
        final bodyString = await request.readAsString();
        final Map<String, dynamic> json = jsonDecode(bodyString);
        final turnoReq = TurnoRequest.fromJson(json);

        final esistente = await turniRepo.getTurnoById(turnoId);
        if (esistente == null) {
          return Response.notFound(
            jsonEncode({'errore': 'Turno non trovato'}),
            headers: {'content-type': 'application/json'},
          );
        }

        final success = await turniRepo.aggiornaTurno(
          id: turnoId,
          data: turnoReq.data,
          note: turnoReq.note,
        );

        if (!success) {
          return Response.internalServerError(
            body: jsonEncode({'errore': 'Impossibile aggiornare il turno'}),
            headers: {'content-type': 'application/json'},
          );
        }

        final aggiornato = await turniRepo.getTurnoById(turnoId);
        final stats = await turniRepo.getStatisticheTurno(turnoId);

        return Response.ok(
          jsonEncode(TurnoResponse(
            id: aggiornato!.id,
            data: aggiornato.data,
            note: aggiornato.note,
            totaleOrdini: stats?.totaleOrdini ?? 0,
            incassoTotale: stats?.incassoTotale ?? 0.0,
          ).toJson()),
          headers: {'content-type': 'application/json'},
        );
      } catch (e) {
        return Response(400, body: jsonEncode({'errore': 'Dati richiesta non validi', 'dettaglio': e.toString()}), headers: {'content-type': 'application/json'});
      }
    });

    // DELETE /api/v1/turni/<id> - Elimina turno
    app.delete('/<id|[0-9]+>', (Request request, String id) async {
      try {
        final turnoId = int.parse(id);
        final eliminato = await turniRepo.eliminaTurno(turnoId);
        if (!eliminato) {
          return Response.notFound(
            jsonEncode({'errore': 'Turno non trovato o già eliminato'}),
            headers: {'content-type': 'application/json'},
          );
        }

        return Response(204, headers: {'content-type': 'application/json'});
      } catch (e) {
        return Response.internalServerError(
          body: jsonEncode({'errore': 'Errore eliminazione turno', 'dettaglio': e.toString()}),
          headers: {'content-type': 'application/json'},
        );
      }
    });

    return app;
  }
}
```

---

## 1.4 — API Routes per Ordini (Dart Shelf)

Le rotte degli ordini applicano una validazione rigorosa dei parametri di input e gestiscono il riordino batch sequenziale in transazione.

### Codice Completo: `OrdiniApi`

```dart
// lib/api/routes/ordini_routes.dart
import 'dart:convert';
import 'package:shelf/shelf.dart';
import 'package:shelf_router/shelf_router.dart';
import '../../data/repositories/ordini_repository.dart';
import '../../data/repositories/turni_repository.dart';
import '../../data/daos/ordini_dao.dart';
import '../../models/ordine_dtos.dart';

class OrdiniApi {
  final OrdiniRepository ordiniRepo;
  final TurniRepository turniRepo;

  OrdiniApi({required this.ordiniRepo, required this.turniRepo});

  static final List<String> statiValidi = StatoOrdine.values.map((e) => e.codice).toList();

  Router get router {
    final app = Router();

    // GET /api/v1/ordini?turno_id=... - Lista ordini per turno
    app.get('/', (Request request) async {
      try {
        final turnoIdStr = request.url.queryParameters['turno_id'];
        if (turnoIdStr == null || int.tryParse(turnoIdStr) == null) {
          return Response(400,
              body: jsonEncode({'errore': 'Parametro obbligatorio `turno_id` mancante o non valido'}),
              headers: {'content-type': 'application/json'});
        }

        final turnoId = int.parse(turnoIdStr);
        final ordini = await ordiniRepo.getOrdiniByTurno(turnoId);
        final dtos = ordini.map((o) => o.toResponseDto()).toList();

        return Response.ok(
          jsonEncode({'ordini': dtos.map((e) => e.toJson()).toList()}),
          headers: {'content-type': 'application/json'},
        );
      } catch (e) {
        return Response.internalServerError(
          body: jsonEncode({'errore': 'Errore recupero ordini', 'dettaglio': e.toString()}),
          headers: {'content-type': 'application/json'},
        );
      }
    });

    // POST /api/v1/ordini - Crea ordine
    app.post('/', (Request request) async {
      try {
        final body = await request.readAsString();
        if (body.isEmpty) {
          return Response(400, body: jsonEncode({'errore': 'Corpo richiesta vuoto'}), headers: {'content-type': 'application/json'});
        }

        final json = jsonDecode(body) as Map<String, dynamic>;
        final req = OrdineRequest.fromJson(json);

        // Validazioni di business
        final errori = <String>[];
        if (req.nomeCliente.trim().length < 2) errori.add('nomeCliente deve contenere almeno 2 caratteri');
        if (req.indirizzoCliente.trim().length < 4) errori.add('indirizzoCliente non valido o troppo corto');
        if (req.importo <= 0) errori.add('importo deve essere strettamente maggiore di 0');
        final orarioRegExp = RegExp(r'^([0-1]?[0-9]|2[0-3]):[0-5][0-9]$');
        if (!orarioRegExp.hasMatch(req.orarioRichiesto)) {
          errori.add('orarioRichiesto deve essere nel formato HH:mm (es. 20:30)');
        }

        // Verifica esistenza turno
        final turno = await turniRepo.getTurnoById(req.turnoId);
        if (turno == null) errori.add('Turno con id ${req.turnoId} inesistente');

        if (errori.isNotEmpty) {
          return Response(422,
              body: jsonEncode({'errore': 'Validazione fallita', 'dettagli': errori}),
              headers: {'content-type': 'application/json'});
        }

        final ordineId = await ordiniRepo.creaOrdine(
          turnoId: req.turnoId,
          nomeCliente: req.nomeCliente,
          telefonoCliente: req.telefonoCliente,
          indirizzoCliente: req.indirizzoCliente,
          importo: req.importo,
          orarioRichiesto: req.orarioRichiesto,
          note: req.note,
        );

        final ordineCreato = await ordiniRepo.getOrdineById(ordineId);
        return Response(
          201,
          body: jsonEncode(ordineCreato!.toResponseDto().toJson()),
          headers: {'content-type': 'application/json'},
        );
      } on FormatException catch (e) {
        return Response(400, body: jsonEncode({'errore': 'JSON non valido', 'dettaglio': e.message}), headers: {'content-type': 'application/json'});
      } catch (e) {
        return Response.internalServerError(
          body: jsonEncode({'errore': 'Errore creazione ordine', 'dettaglio': e.toString()}),
          headers: {'content-type': 'application/json'},
        );
      }
    });

    // PUT /api/v1/ordini/sequenza - Aggiornamento batch della sequenza consegne
    app.put('/sequenza', (Request request) async {
      try {
        final body = await request.readAsString();
        final json = jsonDecode(body) as Map<String, dynamic>;

        if (!json.containsKey('sequenze') || json['sequenze'] is! List) {
          return Response(400,
              body: jsonEncode({'errore': 'Atteso campo array `sequenze`'}),
              headers: {'content-type': 'application/json'});
        }

        final listaRaw = json['sequenze'] as List<dynamic>;
        final itemsBatch = <ElementoSequenzaBatch>[];

        for (final item in listaRaw) {
          if (item is! Map<String, dynamic> || !item.containsKey('id') || !item.containsKey('sequenza')) {
            return Response(400,
                body: jsonEncode({'errore': 'Ogni elemento deve contenere `id` e `sequenza`'}),
                headers: {'content-type': 'application/json'});
          }
          itemsBatch.add(
            ElementoSequenzaBatch(
              ordineId: item['id'] as int,
              nuovaSequenza: item['sequenza'] as int,
            ),
          );
        }

        await ordiniRepo.aggiornaSequenzaBatch(itemsBatch);

        return Response.ok(
          jsonEncode({'messaggio': 'Sequenza aggiornata con successo', 'conteggio': itemsBatch.length}),
          headers: {'content-type': 'application/json'},
        );
      } catch (e) {
        return Response.internalServerError(
          body: jsonEncode({'errore': 'Errore aggiornamento sequenza batch', 'dettaglio': e.toString()}),
          headers: {'content-type': 'application/json'},
        );
      }
    });

    // PATCH /api/v1/ordini/<id>/stato - Modifica stato ordine
    app.patch('/<id|[0-9]+>/stato', (Request request, String id) async {
      try {
        final ordineId = int.parse(id);
        final body = await request.readAsString();
        final json = jsonDecode(body) as Map<String, dynamic>;

        final nuovoStato = json['stato'] as String?;
        if (nuovoStato == null || !statiValidi.contains(nuovoStato)) {
          return Response(400,
              body: jsonEncode({'errore': 'Stato non valido. Valori ammessi: $statiValidi'}),
              headers: {'content-type': 'application/json'});
        }

        final success = await ordiniRepo.aggiornaStato(ordineId, nuovoStato);
        if (!success) {
          return Response.notFound(
            jsonEncode({'errore': 'Ordine non trovato'}),
            headers: {'content-type': 'application/json'},
          );
        }

        final aggiornato = await ordiniRepo.getOrdineById(ordineId);
        return Response.ok(
          jsonEncode(aggiornato!.toResponseDto().toJson()),
          headers: {'content-type': 'application/json'},
        );
      } catch (e) {
        return Response(400, body: jsonEncode({'errore': 'Richiesta non valida', 'dettaglio': e.toString()}), headers: {'content-type': 'application/json'});
      }
    });

    // PATCH /api/v1/ordini/<id> - Modifica anagrafica e dettagli ordine
    app.patch('/<id|[0-9]+>', (Request request, String id) async {
      try {
        final ordineId = int.parse(id);
        final esistente = await ordiniRepo.getOrdineById(ordineId);
        if (esistente == null) {
          return Response.notFound(
            jsonEncode({'errore': 'Ordine non trovato'}),
            headers: {'content-type': 'application/json'},
          );
        }

        final body = await request.readAsString();
        final json = jsonDecode(body) as Map<String, dynamic>;

        final nomeCliente = json['nomeCliente'] ?? esistente.cliente.nome;
        final telefonoCliente = json['telefonoCliente'] ?? esistente.cliente.telefono;
        final indirizzoCliente = json['indirizzoCliente'] ?? esistente.cliente.indirizzo;
        final importo = (json['importo'] as num?)?.toDouble() ?? esistente.ordine.importo;
        final orarioRichiesto = json['orarioRichiesto'] ?? esistente.ordine.orarioRichiesto;
        final note = json.containsKey('note') ? json['note'] as String? : esistente.ordine.note;

        if (importo <= 0) {
          return Response(422, body: jsonEncode({'errore': 'Importo deve essere maggiore di 0'}), headers: {'content-type': 'application/json'});
        }

        await ordiniRepo.aggiornaOrdine(
          ordineId: ordineId,
          nomeCliente: nomeCliente,
          telefonoCliente: telefonoCliente,
          indirizzoCliente: indirizzoCliente,
          importo: importo,
          orarioRichiesto: orarioRichiesto,
          note: note,
        );

        final aggiornato = await ordiniRepo.getOrdineById(ordineId);
        return Response.ok(
          jsonEncode(aggiornato!.toResponseDto().toJson()),
          headers: {'content-type': 'application/json'},
        );
      } catch (e) {
        return Response.internalServerError(
          body: jsonEncode({'errore': 'Errore modifica ordine', 'dettaglio': e.toString()}),
          headers: {'content-type': 'application/json'},
        );
      }
    });

    // DELETE /api/v1/ordini/<id> - Elimina ordine
    app.delete('/<id|[0-9]+>', (Request request, String id) async {
      try {
        final ordineId = int.parse(id);
        final eliminato = await ordiniRepo.eliminaOrdine(ordineId);
        if (!eliminato) {
          return Response.notFound(
            jsonEncode({'errore': 'Ordine non trovato'}),
            headers: {'content-type': 'application/json'},
          );
        }

        return Response(204, headers: {'content-type': 'application/json'});
      } catch (e) {
        return Response.internalServerError(
          body: jsonEncode({'errore': 'Errore eliminazione ordine', 'dettaglio': e.toString()}),
          headers: {'content-type': 'application/json'},
        );
      }
    });

    return app;
  }
}
```

### 1.4.3 Configurazione Router Principale (`lib/api/routes/api_router.dart`)

```dart
import 'package:shelf/shelf.dart';
import 'package:shelf_router/shelf_router.dart';

import '../../data/daos/ordini_dao.dart';
import '../../data/daos/turni_dao.dart';
import '../../data/database.dart';
import '../../data/repositories/ordini_repository.dart';
import '../../data/repositories/turni_repository.dart';
import 'health_route.dart';
import 'ordini_api.dart';
import 'turni_api.dart';

class ApiRouter {
  final AppDatabase db;

  ApiRouter(this.db);

  Router get router {
    final router = Router();
    final healthRoute = HealthRoute();

    // Rotte Health dirette (senza mount annidati per evitare /health/health)
    router.get('/health', healthRoute.health);
    router.get('/api/v1/health', healthRoute.health);
    router.get('/ping', healthRoute.ping);

    // Inizializzazione DAO e Repository nell'Isolate Shelf
    final turniDao = TurniDao(db);
    final ordiniDao = OrdiniDao(db);
    final turniRepo = TurniRepository(turniDao);
    final ordiniRepo = OrdiniRepository(ordiniDao);

    // Montaggio rotte modulari con prefisso /api/v1
    final turniApi = TurniApi(turniRepo: turniRepo, ordiniRepo: ordiniRepo);
    final ordiniApi = OrdiniApi(ordiniRepo: ordiniRepo, turniRepo: turniRepo);

    router.mount('/api/v1/turni', turniApi.router);
    router.mount('/api/v1/ordini', ordiniApi.router);

    return router;
  }

  Handler get handler => router.call;
}
```

---

## 1.5 — Modelli DTO

I DTO incapsulano il contratto di comunicazione REST tra il frontend Flutter e il server Shelf. Sono annotati per la serializzazione con `json_serializable` e includono metodi `fromJson` e `toJson`.

### DTO per Turni: `lib/models/turno_dtos.dart`

```dart
import 'package:json_annotation/json_annotation.dart';
import 'ordine_dtos.dart';

part 'turno_dtos.g.dart';

@JsonSerializable()
class TurnoRequest {
  final DateTime data;
  final String? note;

  TurnoRequest({
    required this.data,
    this.note,
  });

  factory TurnoRequest.fromJson(Map<String, dynamic> json) => _$TurnoRequestFromJson(json);
  Map<String, dynamic> toJson() => _$TurnoRequestToJson(this);
}

@JsonSerializable()
class TurnoResponse {
  final int id;
  final DateTime data;
  final String? note;
  @JsonKey(name: 'totale_ordini')
  final int totaleOrdini;
  @JsonKey(name: 'incasso_totale')
  final double incassoTotale;

  TurnoResponse({
    required this.id,
    required this.data,
    this.note,
    this.totaleOrdini = 0,
    this.incassoTotale = 0.0,
  });

  factory TurnoResponse.fromJson(Map<String, dynamic> json) => _$TurnoResponseFromJson(json);
  Map<String, dynamic> toJson() => _$TurnoResponseToJson(this);
}

@JsonSerializable()
class TurnoDettaglioResponse {
  final int id;
  final DateTime data;
  final String? note;
  @JsonKey(name: 'totale_ordini')
  final int totaleOrdini;
  @JsonKey(name: 'incasso_totale')
  final double incassoTotale;
  final List<OrdineResponse> ordini;

  TurnoDettaglioResponse({
    required this.id,
    required this.data,
    this.note,
    required this.totaleOrdini,
    required this.incassoTotale,
    required this.ordini,
  });

  factory TurnoDettaglioResponse.fromJson(Map<String, dynamic> json) => _$TurnoDettaglioResponseFromJson(json);
  Map<String, dynamic> toJson() => _$TurnoDettaglioResponseToJson(this);
}
```

### DTO per Ordini: `lib/models/ordine_dtos.dart`

```dart
import 'package:json_annotation/json_annotation.dart';

part 'ordine_dtos.g.dart';

@JsonSerializable()
class ClienteDto {
  final int id;
  final String nome;
  final String telefono;
  final String indirizzo;

  ClienteDto({
    required this.id,
    required this.nome,
    required this.telefono,
    required this.indirizzo,
  });

  factory ClienteDto.fromJson(Map<String, dynamic> json) => _$ClienteDtoFromJson(json);
  Map<String, dynamic> toJson() => _$ClienteDtoToJson(this);
}

@JsonSerializable()
class OrdineRequest {
  @JsonKey(name: 'turno_id')
  final int turnoId;
  @JsonKey(name: 'nome_cliente')
  final String nomeCliente;
  @JsonKey(name: 'telefono_cliente')
  final String telefonoCliente;
  @JsonKey(name: 'indirizzo_cliente')
  final String indirizzoCliente;
  final double importo;
  @JsonKey(name: 'orario_richiesto')
  final String orarioRichiesto; // formato HH:mm
  final String? note;

  OrdineRequest({
    required this.turnoId,
    required this.nomeCliente,
    required this.telefonoCliente,
    required this.indirizzoCliente,
    required this.importo,
    required this.orarioRichiesto,
    this.note,
  });

  factory OrdineRequest.fromJson(Map<String, dynamic> json) => _$OrdineRequestFromJson(json);
  Map<String, dynamic> toJson() => _$OrdineRequestToJson(this);
}

@JsonSerializable()
class OrdineResponse {
  final int id;
  @JsonKey(name: 'turno_id')
  final int turnoId;
  final ClienteDto cliente;
  final double importo;
  @JsonKey(name: 'orario_richiesto')
  final String orarioRichiesto;
  final String stato;
  final int sequenza;
  final String? note;
  @JsonKey(name: 'is_in_ritardo')
  final bool isInRitardo;

  OrdineResponse({
    required this.id,
    required this.turnoId,
    required this.cliente,
    required this.importo,
    required this.orarioRichiesto,
    required this.stato,
    required this.sequenza,
    this.note,
    this.isInRitardo = false,
  });

  factory OrdineResponse.fromJson(Map<String, dynamic> json) => _$OrdineResponseFromJson(json);
  Map<String, dynamic> toJson() => _$OrdineResponseToJson(this);

  /// Calcola se l'ordine è in ritardo rispetto alla deadline con tolleranza configurabile.
  /// Gestisce i turni a cavallo della mezzanotte (se l'orario richiesto precede l'inizio turno, aggiunge 1 giorno).
  /// Il parametro [oraCorrente] permette unit test completamente deterministici senza dipendere da DateTime.now().
  bool calcolaRitardo(
    DateTime dataTurno, {
    int tolleranzaMinuti = 5,
    bool evidenziaRitardi = true,
    DateTime? oraCorrente,
  }) {
    if (!evidenziaRitardi) return false;
    if (stato == 'consegnato') return false;
    try {
      final parts = orarioRichiesto.split(':');
      final ore = int.parse(parts[0]);
      final minuti = int.parse(parts[1]);
      var orarioPrevisto = DateTime(dataTurno.year, dataTurno.month, dataTurno.day, ore, minuti);
      
      // Se l'orario richiesto è precedente all'orario di inizio turno, è nel giorno successivo (cavallo mezzanotte)
      if (orarioPrevisto.isBefore(dataTurno)) {
        orarioPrevisto = orarioPrevisto.add(const Duration(days: 1));
      }
      
      final deadline = orarioPrevisto.add(Duration(minutes: tolleranzaMinuti));
      final adesso = oraCorrente ?? DateTime.now();
      return adesso.isAfter(deadline);
    } catch (_) {
      return false;
    }
  }
}

@JsonSerializable()
class SequenzaItemDto {
  final int id;
  final int sequenza;

  SequenzaItemDto({required this.id, required this.sequenza});

  factory SequenzaItemDto.fromJson(Map<String, dynamic> json) => _$SequenzaItemDtoFromJson(json);
  Map<String, dynamic> toJson() => _$SequenzaItemDtoToJson(this);
}

extension OrdineConClienteMapper on OrdineConCliente {
  OrdineResponse toResponseDto() {
    return OrdineResponse(
      id: ordine.id,
      turnoId: ordine.turnoId,
      cliente: ClienteDto(
        id: cliente.id,
        nome: ordine.nomeClienteConsegna,
        telefono: ordine.telefonoConsegna,
        indirizzo: ordine.indirizzoConsegna,
      ),
      importo: ordine.importo,
      orarioRichiesto: ordine.orarioRichiesto,
      stato: ordine.stato.codice,
      sequenza: ordine.sequenza,
      note: ordine.note,
    );
  }
}
```

---

## 1.6 — Service Layer (Client HTTP lato Flutter)

Il Service Layer effettua le chiamate HTTP al demone Shelf locale (`127.0.0.1:8080`), gestisce i timeout e incapsula la gestione delle eccezioni.

### Client HTTP di Base con Gestione Errori

```dart
// lib/services/api_client.dart
import 'dart:convert';
import 'dart:io';
import 'package:http/http.dart' as http;

class ApiException implements Exception {
  final int statusCode;
  final String message;
  final dynamic details;

  ApiException({required this.statusCode, required this.message, this.details});

  @override
  String toString() => 'ApiException (HTTP $statusCode): $message ${details ?? ""}';
}

class ApiClient {
  final String baseUrl;
  final String? authToken;
  final http.Client _httpClient;

  ApiClient({
    this.baseUrl = 'http://127.0.0.1:8080/api/v1',
    this.authToken,
    http.Client? client,
  }) : _httpClient = client ?? http.Client();

  Map<String, String> get _headers => {
        'Content-Type': 'application/json; charset=utf-8',
        'Accept': 'application/json',
        if (authToken != null) 'Authorization': 'Bearer $authToken',
      };

  Future<dynamic> get(String path, {Map<String, String>? queryParams}) async {
    try {
      final uri = Uri.parse('$baseUrl$path').replace(queryParameters: queryParams);
      final response = await _httpClient.get(uri, headers: _headers).timeout(const Duration(seconds: 10));
      return _handleResponse(response);
    } on SocketException {
      throw ApiException(statusCode: 503, message: 'Server locale non raggiungibile su $baseUrl');
    }
  }

  Future<dynamic> post(String path, {dynamic body}) async {
    try {
      final uri = Uri.parse('$baseUrl$path');
      final response = await _httpClient
          .post(uri, headers: _headers, body: body != null ? jsonEncode(body) : null)
          .timeout(const Duration(seconds: 10));
      return _handleResponse(response);
    } on SocketException {
      throw ApiException(statusCode: 503, message: 'Server locale non raggiungibile su $baseUrl');
    }
  }

  Future<dynamic> put(String path, {dynamic body}) async {
    try {
      final uri = Uri.parse('$baseUrl$path');
      final response = await _httpClient
          .put(uri, headers: _headers, body: body != null ? jsonEncode(body) : null)
          .timeout(const Duration(seconds: 10));
      return _handleResponse(response);
    } on SocketException {
      throw ApiException(statusCode: 503, message: 'Server locale non raggiungibile su $baseUrl');
    }
  }

  Future<dynamic> patch(String path, {dynamic body}) async {
    try {
      final uri = Uri.parse('$baseUrl$path');
      final response = await _httpClient
          .patch(uri, headers: _headers, body: body != null ? jsonEncode(body) : null)
          .timeout(const Duration(seconds: 10));
      return _handleResponse(response);
    } on SocketException {
      throw ApiException(statusCode: 503, message: 'Server locale non raggiungibile su $baseUrl');
    }
  }

  Future<dynamic> delete(String path) async {
    try {
      final uri = Uri.parse('$baseUrl$path');
      final response = await _httpClient.delete(uri, headers: _headers).timeout(const Duration(seconds: 10));
      return _handleResponse(response);
    } on SocketException {
      throw ApiException(statusCode: 503, message: 'Server locale non raggiungibile su $baseUrl');
    }
  }

  dynamic _handleResponse(http.Response res) {
    if (res.statusCode == 204) return null;

    dynamic jsonBody;
    try {
      jsonBody = jsonDecode(res.body);
    } catch (_) {
      jsonBody = res.body;
    }

    if (res.statusCode >= 200 && res.statusCode < 300) {
      return jsonBody;
    } else {
      final errorMsg = jsonBody is Map ? (jsonBody['errore'] ?? 'Errore HTTP ${res.statusCode}') : 'Errore ${res.statusCode}';
      throw ApiException(statusCode: res.statusCode, message: errorMsg.toString(), details: jsonBody);
    }
  }
}
```

### Codice Completo: `TurniService`

```dart
// lib/services/turni_service.dart
import '../models/turno_dtos.dart';
import 'api_client.dart';

class TurniService {
  final ApiClient api;

  TurniService(this.api);

  Future<List<TurnoResponse>> getTurni({String? filtro, int limit = 50, int offset = 0}) async {
    final params = <String, String>{
      'limit': limit.toString(),
      'offset': offset.toString(),
    };
    if (filtro != null) params['filtro'] = filtro;

    final data = await api.get('/turni', queryParams: params);
    final list = data['turni'] as List<dynamic>;
    return list.map((item) => TurnoResponse.fromJson(item as Map<String, dynamic>)).toList();
  }

  Future<TurnoDettaglioResponse> getTurnoById(int id) async {
    final data = await api.get('/turni/$id');
    return TurnoDettaglioResponse.fromJson(data as Map<String, dynamic>);
  }

  Future<TurnoResponse> creaTurno({required DateTime data, String? note}) async {
    final req = TurnoRequest(data: data, note: note);
    final res = await api.post('/turni', body: req.toJson());
    return TurnoResponse.fromJson(res as Map<String, dynamic>);
  }

  Future<TurnoResponse> aggiornaTurno(int id, {required DateTime data, String? note}) async {
    final req = TurnoRequest(data: data, note: note);
    final res = await api.put('/turni/$id', body: req.toJson());
    return TurnoResponse.fromJson(res as Map<String, dynamic>);
  }

  Future<void> eliminaTurno(int id) async {
    await api.delete('/turni/$id');
  }
}
```

### Codice Completo: `OrdiniService`

```dart
// lib/services/ordini_service.dart
import '../models/ordine_dtos.dart';
import 'api_client.dart';

class OrdiniService {
  final ApiClient api;

  OrdiniService(this.api);

  Future<List<OrdineResponse>> getOrdiniByTurno(int turnoId) async {
    final data = await api.get('/ordini', queryParams: {'turno_id': turnoId.toString()});
    final list = data['ordini'] as List<dynamic>;
    return list.map((item) => OrdineResponse.fromJson(item as Map<String, dynamic>)).toList();
  }

  Future<OrdineResponse> creaOrdine(OrdineRequest request) async {
    final data = await api.post('/ordini', body: request.toJson());
    return OrdineResponse.fromJson(data as Map<String, dynamic>);
  }

  Future<OrdineResponse> aggiornaStato(int ordineId, String nuovoStato) async {
    final data = await api.patch('/ordini/$ordineId/stato', body: {'stato': nuovoStato});
    return OrdineResponse.fromJson(data as Map<String, dynamic>);
  }

  Future<void> aggiornaSequenzaBatch(List<SequenzaItemDto> items) async {
    await api.put('/ordini/sequenza', body: {
      'sequenze': items.map((e) => e.toJson()).toList(),
    });
  }

  Future<OrdineResponse> aggiornaDettagliOrdine(int ordineId, Map<String, dynamic> updates) async {
    final data = await api.patch('/ordini/$ordineId', body: updates);
    return OrdineResponse.fromJson(data as Map<String, dynamic>);
  }

  Future<void> eliminaOrdine(int ordineId) async {
    await api.delete('/ordini/$ordineId');
  }
}
```

---

## 1.7 — Riverpod Providers

La gestione dello stato adotta **Riverpod 2** con `AsyncNotifier` per garantire reattività, gestione ottimistica della UI e separazione tra stato e presentazione.

```dart
// lib/ui/providers/turni_providers.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../main.dart';
import '../../services/api_client.dart';
import '../../services/turni_service.dart';
import '../../services/ordini_service.dart';
import '../../models/turno_dtos.dart';
import '../../models/ordine_dtos.dart';

// Provider HTTP Client di base autenticato localmente con il token casuale d'avvio
final apiClientProvider = Provider<ApiClient>((ref) {
  return ApiClient(authToken: appLocalAuthToken);
});

// Provider dei servizi
final turniServiceProvider = Provider<TurniService>((ref) {
  return TurniService(ref.watch(apiClientProvider));
});

final ordiniServiceProvider = Provider<OrdiniService>((ref) {
  return OrdiniService(ref.watch(apiClientProvider));
});

// Filtro selezionato per la lista turni (oggi, settimana, tutti)
final filtroTurniProvider = StateProvider<String>((ref) => 'oggi');

/// Stream ticker temporale che emette ogni 30 secondi per aggiornare dinamicamente
/// i badge di ritardo degli ordini sulla dashboard senza richiedere il ricaricamento manuale.
final orarioCorrenteProvider = StreamProvider.autoDispose<DateTime>((ref) async* {
  yield DateTime.now();
  yield* Stream.periodic(const Duration(seconds: 30), (_) => DateTime.now());
});

// AsyncNotifier per la gestione della lista dei turni
class TurniListNotifier extends AsyncNotifier<List<TurnoResponse>> {
  @override
  Future<List<TurnoResponse>> build() async {
    final filtro = ref.watch(filtroTurniProvider);
    final service = ref.watch(turniServiceProvider);
    return service.getTurni(filtro: filtro == 'tutti' ? null : filtro);
  }

  Future<void> ricarica() async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() async {
      final filtro = ref.read(filtroTurniProvider);
      return ref.read(turniServiceProvider).getTurni(filtro: filtro == 'tutti' ? null : filtro);
    });
  }

  Future<TurnoResponse> aggiungiTurno({required DateTime data, String? note}) async {
    final nuovoTurno = await ref.read(turniServiceProvider).creaTurno(data: data, note: note);
    // Aggiornamento locale dello stato
    state.whenData((turni) {
      state = AsyncValue.data([nuovoTurno, ...turni]);
    });
    return nuovoTurno;
  }

  Future<void> eliminaTurno(int id) async {
    await ref.read(turniServiceProvider).eliminaTurno(id);
    state.whenData((turni) {
      state = AsyncValue.data(turni.where((t) => t.id != id).toList());
    });
  }
}

final turniListProvider = AsyncNotifierProvider<TurniListNotifier, List<TurnoResponse>>(
  TurniListNotifier.new,
);

// Provider Future per il dettaglio del singolo turno
final turnoDettaglioProvider = FutureProvider.family<TurnoDettaglioResponse, int>((ref, turnoId) async {
  final service = ref.watch(turniServiceProvider);
  return service.getTurnoById(turnoId);
});

// AsyncNotifier Family per la lista e riordino ordini di uno specifico turno
class OrdiniTurnoNotifier extends FamilyAsyncNotifier<List<OrdineResponse>, int> {
  @override
  Future<List<OrdineResponse>> build(int arg) async {
    final service = ref.watch(ordiniServiceProvider);
    return service.getOrdiniByTurno(arg);
  }

  /// Cambio rapido di stato con aggiornamento ottimistico
  Future<void> cambiaStato(int ordineId, String nuovoStato) async {
    final statoPrecedente = state.value ?? [];
    
    // Aggiornamento ottimistico
    state = AsyncValue.data(
      statoPrecedente.map((o) => o.id == ordineId ? OrdineResponse(
        id: o.id,
        turnoId: o.turnoId,
        cliente: o.cliente,
        importo: o.importo,
        orarioRichiesto: o.orarioRichiesto,
        stato: nuovoStato,
        sequenza: o.sequenza,
        note: o.note,
      ) : o).toList(),
    );

    try {
      final aggiornato = await ref.read(ordiniServiceProvider).aggiornaStato(ordineId, nuovoStato);
      ref.invalidate(turnoDettaglioProvider(arg));
      state = AsyncValue.data(
        (state.value ?? []).map((o) => o.id == ordineId ? aggiornato : o).toList(),
      );
    } catch (e, stack) {
      // Rollback
      state = AsyncValue.data(statoPrecedente);
      state = AsyncValue.error(e, stack);
    }
  }

  /// Riordino drag & drop della sequenza con aggiornamento batch
  Future<void> riordina(int oldIndex, int newIndex) async {
    final listaCorrente = List<OrdineResponse>.from(state.value ?? []);
    if (oldIndex < newIndex) {
      newIndex -= 1;
    }
    final itemSpostato = listaCorrente.removeAt(oldIndex);
    listaCorrente.insert(newIndex, itemSpostato);

    // Ricalcola numeri di sequenza progressivi
    final List<SequenzaItemDto> payloadBatch = [];
    final List<OrdineResponse> listaRicalcolata = [];

    for (int i = 0; i < listaCorrente.length; i++) {
      final ordine = listaCorrente[i];
      final nuovaSeq = i + 1;
      listaRicalcolata.add(
        OrdineResponse(
          id: ordine.id,
          turnoId: ordine.turnoId,
          cliente: ordine.cliente,
          importo: ordine.importo,
          orarioRichiesto: ordine.orarioRichiesto,
          stato: ordine.stato,
          sequenza: nuovaSeq,
          note: ordine.note,
        ),
      );
      payloadBatch.add(SequenzaItemDto(id: ordine.id, sequenza: nuovaSeq));
    }

    // Applica subito alla UI
    state = AsyncValue.data(listaRicalcolata);

    try {
      await ref.read(ordiniServiceProvider).aggiornaSequenzaBatch(payloadBatch);
    } catch (e, st) {
      ref.invalidateSelf();
      state = AsyncValue.error(e, st);
    }
  }

  Future<void> aggiungiOrdine(OrdineRequest req) async {
    final nuovo = await ref.read(ordiniServiceProvider).creaOrdine(req);
    ref.invalidate(turnoDettaglioProvider(arg));
    state.whenData((ordini) {
      state = AsyncValue.data([...ordini, nuovo]);
    });
  }

  Future<void> eliminaOrdine(int ordineId) async {
    await ref.read(ordiniServiceProvider).eliminaOrdine(ordineId);
    ref.invalidate(turnoDettaglioProvider(arg));
    ref.invalidateSelf();
  }
}

final ordiniTurnoProvider = AsyncNotifierProvider.family<OrdiniTurnoNotifier, List<OrdineResponse>, int>(
  OrdiniTurnoNotifier.new,
);
```

---

## 1.8 — UI — Lista Turni (Dashboard)

La dashboard iniziale per Windows Desktop consente di monitorare lo storico e lo stato dei turni giornalieri con statistiche aggregate.

```dart
// lib/ui/screens/turni_dashboard_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:intl/intl.dart';
import '../providers/turni_providers.dart';
import '../widgets/crea_turno_dialog.dart';
import 'turno_dettaglio_screen.dart';

class TurniDashboardScreen extends ConsumerWidget {
  const TurniDashboardScreen({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final turniAsync = ref.watch(turniListProvider);
    final filtroAttivo = ref.watch(filtroTurniProvider);

    return Scaffold(
      backgroundColor: const Color(0xFFF8F9FA),
      appBar: AppBar(
        title: const Text('DelivMap — Gestione Turni', style: TextStyle(fontWeight: FontWeight.bold)),
        backgroundColor: Colors.white,
        foregroundColor: const Color(0xFF1E293B),
        elevation: 1,
        actions: [
          // Segmented button filtro
          Padding(
            padding: const EdgeInsets.symmetric(vertical: 8.0),
            child: SegmentedButton<String>(
              segments: const [
                ButtonSegment(value: 'oggi', label: Text('Oggi')),
                ButtonSegment(value: 'settimana', label: Text('Ultimi 7 Giorni')),
                ButtonSegment(value: 'tutti', label: Text('Tutti')),
              ],
              selected: {filtroAttivo},
              onSelectionChanged: (newSelection) {
                ref.read(filtroTurniProvider.notifier).state = newSelection.first;
              },
            ),
          ),
          const SizedBox(width: 16),
          IconButton(
            tooltip: 'Ricarica Turni',
            icon: const Icon(Icons.refresh),
            onPressed: () => ref.read(turniListProvider.notifier).ricarica(),
          ),
          const SizedBox(width: 16),
        ],
      ),
      floatingActionButton: FloatingActionButton.extended(
        backgroundColor: const Color(0xFF2563EB),
        foregroundColor: Colors.white,
        icon: const Icon(Icons.add),
        label: const Text('Nuovo Turno'),
        onPressed: () => _mostraDialogNuovoTurno(context, ref),
      ),
      body: turniAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (err, _) => Center(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              const Icon(Icons.cloud_off, size: 64, color: Colors.redAccent),
              const SizedBox(height: 16),
              Text('Errore nel caricamento dei turni: $err', style: const TextStyle(color: Colors.red)),
              const SizedBox(height: 12),
              ElevatedButton(
                onPressed: () => ref.read(turniListProvider.notifier).ricarica(),
                child: const Text('Riprova'),
              )
            ],
          ),
        ),
        data: (turni) {
          if (turni.isEmpty) {
            return Center(
              child: Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  Icon(Icons.calendar_today_outlined, size: 80, color: Colors.grey.shade400),
                  const SizedBox(height: 16),
                  Text(
                    'Nessun turno trovato per la selezione corrente.',
                    style: TextStyle(fontSize: 18, color: Colors.grey.shade700),
                  ),
                  const SizedBox(height: 12),
                  ElevatedButton.icon(
                    icon: const Icon(Icons.add),
                    label: const Text('Crea il tuo primo turno'),
                    onPressed: () => _mostraDialogNuovoTurno(context, ref),
                  )
                ],
              ),
            );
          }

          return RefreshIndicator(
            onRefresh: () => ref.read(turniListProvider.notifier).ricarica(),
            child: ListView.separated(
              padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 20),
              itemCount: turni.length,
              separatorBuilder: (_, __) => const SizedBox(height: 12),
              itemBuilder: (ctx, index) {
                final turno = turni[index];
                return _TurnoCard(
                  turno: turno,
                  onTap: () {
                    Navigator.of(context).push(
                      MaterialPageRoute(
                        builder: (_) => TurnoDettaglioScreen(turnoId: turno.id),
                      ),
                    );
                  },
                  onDelete: () async {
                    final conferma = await showDialog<bool>(
                      context: context,
                      builder: (dCtx) => AlertDialog(
                        title: const Text('Conferma eliminazione'),
                        content: Text('Vuoi eliminare il turno del ${DateFormat('dd/MM/yyyy HH:mm').format(turno.data)} e tutti i suoi ordini?'),
                        actions: [
                          TextButton(onPressed: () => Navigator.pop(dCtx, false), child: const Text('Annulla')),
                          ElevatedButton(
                            style: ElevatedButton.styleFrom(backgroundColor: Colors.red),
                            onPressed: () => Navigator.pop(dCtx, true),
                            child: const Text('Elimina', style: TextStyle(color: Colors.white)),
                          ),
                        ],
                      ),
                    );
                    if (conferma == true) {
                      ref.read(turniListProvider.notifier).eliminaTurno(turno.id);
                    }
                  },
                );
              },
            ),
          );
        },
      ),
    );
  }

  void _mostraDialogNuovoTurno(BuildContext context, WidgetRef ref) {
    showDialog(
      context: context,
      barrierDismissible: false,
      builder: (_) => const CreaTurnoDialog(),
    );
  }
}

class _TurnoCard extends StatelessWidget {
  final dynamic turno;
  final VoidCallback onTap;
  final VoidCallback onDelete;

  const _TurnoCard({
    required this.turno,
    required this.onTap,
    required this.onDelete,
  });

  @override
  Widget build(BuildContext context) {
    final dateFormat = DateFormat('EEEE d MMMM yyyy — HH:mm', 'it_IT');
    final formattedDate = dateFormat.format(turno.data);

    return Card(
      elevation: 1.5,
      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
      child: InkWell(
        borderRadius: BorderRadius.circular(12),
        onTap: onTap,
        child: Padding(
          padding: const EdgeInsets.all(20.0),
          child: Row(
            children: [
              Container(
                width: 56,
                height: 56,
                decoration: BoxDecoration(
                  color: const Color(0xFFEFF6FF),
                  borderRadius: BorderRadius.circular(10),
                ),
                child: const Icon(Icons.delivery_dining, color: Color(0xFF2563EB), size: 32),
              ),
              const SizedBox(width: 20),
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      formattedDate,
                      style: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold, color: Color(0xFF0F172A)),
                    ),
                    const SizedBox(height: 6),
                    Text(
                      turno.note?.isNotEmpty == true ? turno.note! : 'Nessuna nota aggiuntiva',
                      style: TextStyle(fontSize: 14, color: Colors.grey.shade600),
                    ),
                  ],
                ),
              ),
              Column(
                crossAxisAlignment: CrossAxisAlignment.end,
                children: [
                  Row(
                    children: [
                      Container(
                        padding: const EdgeInsets.symmetric(horizontal: 10, vertical: 4),
                        decoration: BoxDecoration(
                          color: const Color(0xFFF1F5F9),
                          borderRadius: BorderRadius.circular(8),
                        ),
                        child: Text(
                          '${turno.totaleOrdini} ordini',
                          style: const TextStyle(fontWeight: FontWeight.w600, color: Color(0xFF334155)),
                        ),
                      ),
                      const SizedBox(width: 8),
                      Container(
                        padding: const EdgeInsets.symmetric(horizontal: 10, vertical: 4),
                        decoration: BoxDecoration(
                          color: const Color(0xFFDCFCE7),
                          borderRadius: BorderRadius.circular(8),
                        ),
                        child: Text(
                          '€ ${turno.incassoTotale.toStringAsFixed(2)}',
                          style: const TextStyle(fontWeight: FontWeight.bold, color: Color(0xFF15803D)),
                        ),
                      ),
                    ],
                  ),
                ],
              ),
              const SizedBox(width: 12),
              IconButton(
                icon: const Icon(Icons.delete_outline, color: Colors.redAccent),
                onPressed: onDelete,
                tooltip: 'Elimina turno',
              ),
              const Icon(Icons.chevron_right, color: Colors.grey),
            ],
          ),
        ),
      ),
    );
  }
}
```

### Dialog di Creazione: `lib/ui/widgets/crea_turno_dialog.dart`

```dart
// lib/ui/widgets/crea_turno_dialog.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:intl/intl.dart';
import '../providers/turni_providers.dart';

class CreaTurnoDialog extends ConsumerStatefulWidget {
  const CreaTurnoDialog({Key? key}) : super(key: key);

  @override
  ConsumerState<CreaTurnoDialog> createState() => _CreaTurnoDialogState();
}

class _CreaTurnoDialogState extends ConsumerState<CreaTurnoDialog> {
  DateTime _dataSelezionata = DateTime.now();
  TimeOfDay _orarioSelezionato = TimeOfDay.now();
  final _noteController = TextEditingController();
  bool _isSaving = false;

  @override
  void dispose() {
    _noteController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final dateString = DateFormat('dd/MM/yyyy').format(_dataSelezionata);
    final timeString = _orarioSelezionato.format(context);

    return AlertDialog(
      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
      title: const Text('Nuovo Turno di Consegna', style: TextStyle(fontWeight: FontWeight.bold)),
      content: SizedBox(
        width: 450,
        child: Column(
          mainAxisSize: MainAxisSize.min,
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            const Text('Data e Orario Inizio Turno', style: TextStyle(fontWeight: FontWeight.w600)),
            const SizedBox(height: 8),
            Row(
              children: [
                Expanded(
                  child: OutlinedButton.icon(
                    icon: const Icon(Icons.calendar_month),
                    label: Text(dateString),
                    onPressed: () async {
                      final picked = await showDatePicker(
                        context: context,
                        initialDate: _dataSelezionata,
                        firstDate: DateTime(2025),
                        lastDate: DateTime(2035),
                      );
                      if (picked != null) setState(() => _dataSelezionata = picked);
                    },
                  ),
                ),
                const SizedBox(width: 12),
                Expanded(
                  child: OutlinedButton.icon(
                    icon: const Icon(Icons.access_time),
                    label: Text(timeString),
                    onPressed: () async {
                      final picked = await showTimePicker(
                        context: context,
                        initialTime: _orarioSelezionato,
                      );
                      if (picked != null) setState(() => _orarioSelezionato = picked);
                    },
                  ),
                ),
              ],
            ),
            const SizedBox(height: 16),
            const Text('Note Sessione (opzionale)', style: TextStyle(fontWeight: FontWeight.w600)),
            const SizedBox(height: 8),
            TextField(
              controller: _noteController,
              decoration: const InputDecoration(
                hintText: 'Es. Pranzo Sabato, pioggia intensa, 2 rider disponibili',
                border: OutlineInputBorder(),
              ),
              maxLines: 2,
            ),
          ],
        ),
      ),
      actions: [
        TextButton(
          onPressed: _isSaving ? null : () => Navigator.pop(context),
          child: const Text('Annulla'),
        ),
        ElevatedButton(
          style: ElevatedButton.styleFrom(backgroundColor: const Color(0xFF2563EB)),
          onPressed: _isSaving ? null : _salvaTurno,
          child: _isSaving
              ? const SizedBox(width: 18, height: 18, child: CircularProgressIndicator(strokeWidth: 2, color: Colors.white))
              : const Text('Crea Turno', style: TextStyle(color: Colors.white)),
        ),
      ],
    );
  }

  Future<void> _salvaTurno() async {
    setState(() => _isSaving = true);
    final dataFinale = DateTime(
      _dataSelezionata.year,
      _dataSelezionata.month,
      _dataSelezionata.day,
      _orarioSelezionato.hour,
      _orarioSelezionato.minute,
    );

    try {
      await ref.read(turniListProvider.notifier).aggiungiTurno(
            data: dataFinale,
            note: _noteController.text.trim().isEmpty ? null : _noteController.text.trim(),
          );
      if (mounted) Navigator.pop(context);
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Errore creazione turno: $e'), backgroundColor: Colors.red),
        );
      }
    } finally {
      if (mounted) setState(() => _isSaving = false);
    }
  }
}
```

---

## 1.9 — UI — Dettaglio Turno con ReorderableListView

La schermata di dettaglio del turno gestisce la sequenza operativa delle consegne:
- Permette di riordinare gli ordini con drag & drop reattivo (`ReorderableListView.builder`).
- Mostra badge di stato e calcola in tempo reale lo stato `in_ritardo` se l'orario richiesto è trascorso senza che l'ordine sia `consegnato`.
- Consente la rapida transizione di stato con un menu contestuale rapido.

```dart
// lib/ui/screens/turno_dettaglio_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:intl/intl.dart';
import '../providers/turni_providers.dart';
import '../models/ordine_dtos.dart';
import '../widgets/form_nuovo_ordine_dialog.dart';

class TurnoDettaglioScreen extends ConsumerWidget {
  final int turnoId;

  const TurnoDettaglioScreen({Key? key, required this.turnoId}) : super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final turnoDettaglioAsync = ref.watch(turnoDettaglioProvider(turnoId));
    final ordiniAsync = ref.watch(ordiniTurnoProvider(turnoId));

    return Scaffold(
      backgroundColor: const Color(0xFFF1F5F9),
      appBar: AppBar(
        title: const Text('Dettaglio Consegne Turno', style: TextStyle(fontWeight: FontWeight.bold)),
        backgroundColor: Colors.white,
        foregroundColor: const Color(0xFF1E293B),
        elevation: 1,
        actions: [
          IconButton(
            tooltip: 'Ricarica',
            icon: const Icon(Icons.refresh),
            onPressed: () {
              ref.invalidate(turnoDettaglioProvider(turnoId));
              ref.invalidate(ordiniTurnoProvider(turnoId));
            },
          ),
          const SizedBox(width: 12),
        ],
      ),
      floatingActionButton: FloatingActionButton.extended(
        backgroundColor: const Color(0xFF2563EB),
        foregroundColor: Colors.white,
        icon: const Icon(Icons.add_shopping_cart),
        label: const Text('Aggiungi Ordine'),
        onPressed: () {
          showDialog(
            context: context,
            barrierDismissible: false,
            builder: (_) => FormNuovoOrdineDialog(turnoId: turnoId),
          );
        },
      ),
      body: turnoDettaglioAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (err, _) => Center(child: Text('Errore: $err', style: const TextStyle(color: Colors.red))),
        data: (turno) {
          return Column(
            children: [
              // Header Statistiche Turno
              _TurnoHeader(turno: turno),

              // Lista Ordini Riordinabile con calcolo dinamico ritardi in tempo reale
              Expanded(
                child: ordiniAsync.when(
                  loading: () => const Center(child: CircularProgressIndicator()),
                  error: (err, _) => Center(child: Text('Errore caricamento ordini: $err')),
                  data: (ordini) {
                    final adesso = ref.watch(orarioCorrenteProvider).valueOrNull ?? DateTime.now();

                    if (ordini.isEmpty) {
                      return Center(
                        child: Column(
                          mainAxisSize: MainAxisSize.min,
                          children: [
                            Icon(Icons.inventory_2_outlined, size: 72, color: Colors.grey.shade400),
                            const SizedBox(height: 12),
                            const Text(
                              'Nessun ordine presente in questo turno.',
                              style: TextStyle(fontSize: 16, color: Color(0xFF64748B)),
                            ),
                          ],
                        ),
                      );
                    }

                    return ReorderableListView.builder(
                      padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 16),
                      itemCount: ordini.length,
                      onReorder: (oldIdx, newIdx) {
                        ref.read(ordiniTurnoProvider(turnoId).notifier).riordina(oldIdx, newIdx);
                      },
                      itemBuilder: (ctx, index) {
                        final ordine = ordini[index];
                        final isRitardo = ordine.calcolaRitardo(turno.data, oraCorrente: adesso);

                        return _OrdineCardItem(
                          key: ValueKey(ordine.id),
                          ordine: ordine,
                          dataTurno: turno.data,
                          isRitardo: isRitardo,
                          onStatoChanged: (nuovoStato) {
                            ref.read(ordiniTurnoProvider(turnoId).notifier).cambiaStato(ordine.id, nuovoStato);
                          },
                          onDelete: () {
                            ref.read(ordiniTurnoProvider(turnoId).notifier).eliminaOrdine(ordine.id);
                          },
                        );
                      },
                    );
                  },
                ),
              ),
            ],
          );
        },
      ),
    );
  }
}

class _TurnoHeader extends StatelessWidget {
  final dynamic turno;

  const _TurnoHeader({required this.turno});

  @override
  Widget build(BuildContext context) {
    final dateStr = DateFormat('EEEE d MMMM yyyy — HH:mm', 'it_IT').format(turno.data);

    return Container(
      padding: const EdgeInsets.all(20),
      color: Colors.white,
      child: Row(
        children: [
          Expanded(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(dateStr, style: const TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
                if (turno.note != null && turno.note.toString().isNotEmpty) ...[
                  const SizedBox(height: 4),
                  Text('Note: ${turno.note}', style: TextStyle(color: Colors.grey.shade600, fontSize: 14)),
                ],
              ],
            ),
          ),
          _KpiBadge(titolo: 'Totale Ordini', valore: '${turno.totaleOrdini}', icon: Icons.receipt_long),
          const SizedBox(width: 16),
          _KpiBadge(
            titolo: 'Incasso Previsto',
            valore: '€ ${turno.incassoTotale.toStringAsFixed(2)}',
            icon: Icons.euro,
            colore: const Color(0xFF16A34A),
          ),
        ],
      ),
    );
  }
}

class _KpiBadge extends StatelessWidget {
  final String titolo;
  final String valore;
  final IconData icon;
  final Color colore;

  const _KpiBadge({
    required this.titolo,
    required this.valore,
    required this.icon,
    this.colore = const Color(0xFF2563EB),
  });

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 10),
      decoration: BoxDecoration(
        color: colore.withOpacity(0.08),
        borderRadius: BorderRadius.circular(10),
        border: Border.all(color: colore.withOpacity(0.2)),
      ),
      child: Row(
        children: [
          Icon(icon, color: colore, size: 24),
          const SizedBox(width: 10),
          Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(titolo, style: TextStyle(fontSize: 11, color: Colors.grey.shade600, fontWeight: FontWeight.w600)),
              Text(valore, style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold, color: colore)),
            ],
          )
        ],
      ),
    );
  }
}

class _OrdineCardItem extends StatelessWidget {
  final OrdineResponse ordine;
  final DateTime dataTurno;
  final bool isRitardo;
  final ValueChanged<String> onStatoChanged;
  final VoidCallback onDelete;

  const _OrdineCardItem({
    required Key key,
    required this.ordine,
    required this.dataTurno,
    required this.isRitardo,
    required this.onStatoChanged,
    required this.onDelete,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Container(
      margin: const EdgeInsets.only(bottom: 12),
      decoration: BoxDecoration(
        color: Colors.white,
        borderRadius: BorderRadius.circular(12),
        border: Border.all(
          color: isRitardo ? Colors.red.shade400 : Colors.grey.shade200,
          width: isRitardo ? 2 : 1,
        ),
        boxShadow: const [
          BoxShadow(color: Color(0x08000000), blurRadius: 4, offset: Offset(0, 2)),
        ],
      ),
      child: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Row(
          children: [
            // Numero di Sequenza (con maniglia di drag)
            Container(
              width: 40,
              height: 40,
              decoration: BoxDecoration(
                color: const Color(0xFF1E293B),
                borderRadius: BorderRadius.circular(8),
              ),
              alignment: Alignment.center,
              child: Text(
                '#${ordine.sequenza}',
                style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold, fontSize: 16),
              ),
            ),
            const SizedBox(width: 16),

            // Dati Cliente & Indirizzo
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Row(
                    children: [
                      Text(
                        ordine.cliente.nome,
                        style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 16, color: Color(0xFF0F172A)),
                      ),
                      const SizedBox(width: 12),
                      Icon(Icons.phone, size: 14, color: Colors.grey.shade600),
                      const SizedBox(width: 4),
                      Text(ordine.cliente.telefono, style: TextStyle(color: Colors.grey.shade700, fontSize: 13)),
                    ],
                  ),
                  const SizedBox(height: 4),
                  Row(
                    children: [
                      const Icon(Icons.location_on, size: 16, color: Color(0xFF64748B)),
                      const SizedBox(width: 4),
                      Expanded(
                        child: Text(
                          ordine.cliente.indirizzo,
                          style: const TextStyle(color: Color(0xFF334155), fontSize: 14),
                          overflow: TextOverflow.ellipsis,
                        ),
                      ),
                    ],
                  ),
                  if (ordine.note != null && ordine.note!.isNotEmpty) ...[
                    const SizedBox(height: 4),
                    Text('Nota: ${ordine.note!}', style: const TextStyle(fontSize: 12, fontStyle: FontStyle.italic, color: Color(0xFF64748B))),
                  ]
                ],
              ),
            ),

            // Orario e Importo
            Column(
              crossAxisAlignment: CrossAxisAlignment.end,
              children: [
                Row(
                  children: [
                    const Icon(Icons.schedule, size: 16, color: Color(0xFF64748B)),
                    const SizedBox(width: 4),
                    Text(
                      ordine.orarioRichiesto,
                      style: TextStyle(
                        fontSize: 15,
                        fontWeight: FontWeight.bold,
                        color: isRitardo ? Colors.red : const Color(0xFF0F172A),
                      ),
                    ),
                  ],
                ),
                const SizedBox(height: 4),
                Text(
                  '€ ${ordine.importo.toStringAsFixed(2)}',
                  style: const TextStyle(fontSize: 15, fontWeight: FontWeight.bold, color: Color(0xFF15803D)),
                ),
              ],
            ),
            const SizedBox(width: 16),

            // Badge Stato & Selettore Rapido
            _BadgeStato(stato: ordine.stato, isRitardo: isRitardo),
            const SizedBox(width: 8),

            PopupMenuButton<String>(
              icon: const Icon(Icons.more_vert),
              tooltip: 'Azioni ordine',
              onSelected: (valore) {
                if (valore == 'elimina') {
                  onDelete();
                } else {
                  onStatoChanged(valore);
                }
              },
              itemBuilder: (ctx) => [
                const PopupMenuItem(value: 'in_attesa', child: Text('Segna: In Attesa')),
                const PopupMenuItem(value: 'in_consegna', child: Text('Segna: In Consegna')),
                const PopupMenuItem(value: 'consegnato', child: Text('Segna: Consegnato')),
                const PopupMenuDivider(),
                const PopupMenuItem(
                  value: 'elimina',
                  child: Text('Elimina Ordine', style: TextStyle(color: Colors.red)),
                ),
              ],
            ),

            const SizedBox(width: 8),
            const Icon(Icons.drag_indicator, color: Color(0xFF94A3B8)),
          ],
        ),
      ),
    );
  }
}

class _BadgeStato extends StatelessWidget {
  final String stato;
  final bool isRitardo;

  const _BadgeStato({required this.stato, required this.isRitardo});

  @override
  Widget build(BuildContext context) {
    if (isRitardo && stato != 'consegnato') {
      return Container(
        padding: const EdgeInsets.symmetric(horizontal: 10, vertical: 6),
        decoration: BoxDecoration(color: Colors.red.shade100, borderRadius: BorderRadius.circular(8)),
        child: const Text('IN RITARDO', style: TextStyle(color: Colors.red, fontWeight: FontWeight.bold, fontSize: 12)),
      );
    }

    Color bgColor;
    Color fgColor;
    String label;

    switch (stato) {
      case 'in_consegna':
        bgColor = const Color(0xFFDBEAFE);
        fgColor = const Color(0xFF1D4ED8);
        label = 'IN CONSEGNA';
        break;
      case 'consegnato':
        bgColor = const Color(0xFFDCFCE7);
        fgColor = const Color(0xFF15803D);
        label = 'CONSEGNATO';
        break;
      case 'in_attesa':
      default:
        bgColor = const Color(0xFFFEF3C7);
        fgColor = const Color(0xFFB45309);
        label = 'IN ATTESA';
        break;
    }

    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 10, vertical: 6),
      decoration: BoxDecoration(color: bgColor, borderRadius: BorderRadius.circular(8)),
      child: Text(label, style: TextStyle(color: fgColor, fontWeight: FontWeight.bold, fontSize: 12)),
    );
  }
}
```

---

## 1.10 — UI — Form Nuovo Ordine con Validazione

Il form per l'inserimento rapido degli ordini su desktop include validazione su tutti i campi obbligatori, time-picker per la fascia oraria di consegna e feedback immediato.

```dart
// lib/ui/widgets/form_nuovo_ordine_dialog.dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../providers/turni_providers.dart';
import '../../models/ordine_dtos.dart';

class FormNuovoOrdineDialog extends ConsumerStatefulWidget {
  final int turnoId;

  const FormNuovoOrdineDialog({Key? key, required this.turnoId}) : super(key: key);

  @override
  ConsumerState<FormNuovoOrdineDialog> createState() => _FormNuovoOrdineDialogState();
}

class _FormNuovoOrdineDialogState extends ConsumerState<FormNuovoOrdineDialog> {
  final _formKey = GlobalKey<FormState>();

  final _nomeController = TextEditingController();
  final _telefonoController = TextEditingController();
  final _indirizzoController = TextEditingController();
  final _importoController = TextEditingController();
  final _noteController = TextEditingController();

  TimeOfDay _orarioSelezionato = const TimeOfDay(hour: 20, minute: 0);
  bool _isSaving = false;

  @override
  void dispose() {
    _nomeController.dispose();
    _telefonoController.dispose();
    _indirizzoController.dispose();
    _importoController.dispose();
    _noteController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final orarioFormatted = '${_orarioSelezionato.hour.toString().padLeft(2, '0')}:${_orarioSelezionato.minute.toString().padLeft(2, '0')}';

    return AlertDialog(
      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
      title: const Text('Nuovo Ordine a Domicilio', style: TextStyle(fontWeight: FontWeight.bold)),
      content: SizedBox(
        width: 520,
        child: Form(
          key: _formKey,
          child: SingleChildScrollView(
            child: Column(
              mainAxisSize: MainAxisSize.min,
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                // Nome cliente
                TextFormField(
                  controller: _nomeController,
                  decoration: const InputDecoration(
                    labelText: 'Nome Cliente *',
                    prefixIcon: Icon(Icons.person),
                    border: OutlineInputBorder(),
                  ),
                  validator: (value) {
                    if (value == null || value.trim().length < 2) {
                      return 'Inserisci un nome valido (almeno 2 caratteri)';
                    }
                    return null;
                  },
                ),
                const SizedBox(height: 14),

                // Telefono
                TextFormField(
                  controller: _telefonoController,
                  keyboardType: TextInputType.phone,
                  decoration: const InputDecoration(
                    labelText: 'Telefono Recapito *',
                    prefixIcon: Icon(Icons.phone),
                    border: OutlineInputBorder(),
                  ),
                  validator: (value) {
                    if (value == null || value.trim().length < 5) {
                      return 'Inserisci un recapito telefonico valido';
                    }
                    return null;
                  },
                ),
                const SizedBox(height: 14),

                // Indirizzo
                TextFormField(
                  controller: _indirizzoController,
                  decoration: const InputDecoration(
                    labelText: 'Indirizzo Completo (Via/Piazza, Civico) *',
                    prefixIcon: Icon(Icons.home),
                    border: OutlineInputBorder(),
                  ),
                  validator: (value) {
                    if (value == null || value.trim().length < 4) {
                      return 'Inserisci via e numero civico';
                    }
                    return null;
                  },
                ),
                const SizedBox(height: 14),

                // Riga Importo e Orario
                Row(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Expanded(
                      child: TextFormField(
                        controller: _importoController,
                        keyboardType: const TextInputType.numberWithOptions(decimal: true),
                        inputFormatters: [
                          FilteringTextInputFormatter.allow(RegExp(r'^\d+[\.,]?\d{0,2}')),
                        ],
                        decoration: const InputDecoration(
                          labelText: 'Importo Totale (€) *',
                          prefixIcon: Icon(Icons.euro),
                          border: OutlineInputBorder(),
                        ),
                        validator: (value) {
                          if (value == null || value.isEmpty) return 'Obbligatorio';
                          final norm = value.replaceAll(',', '.');
                          final d = double.tryParse(norm);
                          if (d == null || d <= 0) return 'Importo > 0';
                          return null;
                        },
                      ),
                    ),
                    const SizedBox(width: 12),
                    Expanded(
                      child: OutlinedButton.icon(
                        style: OutlinedButton.styleFrom(
                          padding: const EdgeInsets.symmetric(vertical: 16),
                          side: BorderSide(color: Colors.grey.shade400),
                        ),
                        icon: const Icon(Icons.access_time),
                        label: Text('Orario: $orarioFormatted', style: const TextStyle(fontSize: 15)),
                        onPressed: () async {
                          final picked = await showTimePicker(
                            context: context,
                            initialTime: _orarioSelezionato,
                          );
                          if (picked != null) {
                            setState(() => _orarioSelezionato = picked);
                          }
                        },
                      ),
                    ),
                  ],
                ),
                const SizedBox(height: 14),

                // Note Consegna
                TextFormField(
                  controller: _noteController,
                  decoration: const InputDecoration(
                    labelText: 'Note Consegna (Citofono, Piano, Allergie)',
                    prefixIcon: Icon(Icons.notes),
                    border: OutlineInputBorder(),
                  ),
                  maxLines: 2,
                ),
              ],
            ),
          ),
        ),
      ),
      actions: [
        TextButton(
          onPressed: _isSaving ? null : () => Navigator.pop(context),
          child: const Text('Annulla'),
        ),
        ElevatedButton(
          style: ElevatedButton.styleFrom(
            backgroundColor: const Color(0xFF2563EB),
            padding: const EdgeInsets.symmetric(horizontal: 20, vertical: 12),
          ),
          onPressed: _isSaving ? null : _inviaOrdine,
          child: _isSaving
              ? const SizedBox(width: 20, height: 20, child: CircularProgressIndicator(color: Colors.white, strokeWidth: 2))
              : const Text('Salva Ordine', style: TextStyle(color: Colors.white, fontWeight: FontWeight.bold)),
        ),
      ],
    );
  }

  Future<void> _inviaOrdine() async {
    if (!_formKey.currentState!.validate()) return;

    setState(() => _isSaving = true);
    final orarioFormatted = '${_orarioSelezionato.hour.toString().padLeft(2, '0')}:${_orarioSelezionato.minute.toString().padLeft(2, '0')}';
    final importo = double.parse(_importoController.text.replaceAll(',', '.'));

    final request = OrdineRequest(
      turnoId: widget.turnoId,
      nomeCliente: _nomeController.text.trim(),
      telefonoCliente: _telefonoController.text.trim(),
      indirizzoCliente: _indirizzoController.text.trim(),
      importo: importo,
      orarioRichiesto: orarioFormatted,
      note: _noteController.text.trim().isEmpty ? null : _noteController.text.trim(),
    );

    try {
      await ref.read(ordiniTurnoProvider(widget.turnoId).notifier).aggiungiOrdine(request);
      if (mounted) {
        Navigator.pop(context);
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(
            content: Text('Ordine inserito con successo!'),
            backgroundColor: Color(0xFF15803D),
          ),
        );
      }
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(
            content: Text('Errore durante il salvataggio: $e'),
            backgroundColor: Colors.red,
          ),
        );
      }
    } finally {
      if (mounted) setState(() => _isSaving = false);
    }
  }
}
```

---

## Criteri di Accettazione

Per considerare la Fase 1 completata e pronta per il merge nel ramo principale, tutte le seguenti condizioni devono essere verificate con esito positivo:

| ID | Modulo | Criterio di Accettazione |
|---|---|---|
| **AC-1.1** | Turni DAO | La creazione del turno salva data e note; i turni vengono restituiti ordinati per data discendente con paginazione e filtri data funzionanti (`oggi`, `ultimi 7 giorni`). |
| **AC-1.2** | Ordini DAO | L'ordine congela uno snapshot immutabile (`nomeClienteConsegna`, `telefonoConsegna`, `indirizzoConsegna`). La sequenza iniziale è calcolata come `max(sequenza) + 1`. |
| **AC-1.3** | Ordini DAO | L'eliminazione di un ordine scala di 1 la sequenza degli ordini successivi all'interno della stessa transazione. |
| **AC-1.4** | API Turni | `POST /api/v1/turni` restituisce `201 Created` con payload conforme; `GET /api/v1/turni` risponde entro 100ms e supporta parametri di filtro. |
| **AC-1.5** | API Ordini | `POST /api/v1/ordini` rifiuta importi `<= 0`, formati orari non conformi a `HH:mm` e riferimenti a turni non esistenti, restituendo codici `400` o `422`. |
| **AC-1.6** | Batch Sequenza | `PUT /api/v1/ordini/sequenza` riceve un array di coppie `{id, sequenza}` e applica gli aggiornamenti in una transazione SQLite isolata, restituendo `200 OK`. |
| **AC-1.7** | Gestione Errori HTTP | In caso di assenza del server locale o errore 500/503, la UI non va in crash ma mostra messaggi contestuali e consente il retry. |
| **AC-1.8** | Reattività UI | Lo spostamento di un elemento nella `ReorderableListView` aggiorna immediatamente la numerazione visuale `#1, #2...` ed effettua la sincronizzazione asincrona. |
| **AC-1.9** | Stati & Ritardi | La macchina a stati accetta solo `in_attesa`, `in_consegna`, `consegnato`. Il ritardo è calcolato dinamicamente con 5 minuti di tolleranza ed è disattivabile dalle impostazioni. |
| **AC-1.10** | Autonomia Locale | Tutte le funzionalità della Fase 1 operano senza connessione Internet attiva. |

---

## Checklist di Testing

### 1. Test Unitari (DAO & Modelli)
- [ ] **TurniDao**: Creazione di 3 turni in date diverse; verifica filtro per turni odierni.
- [ ] **TurniDao**: Calcolo corretto degli aggregati `totaleOrdini` e `incassoTotale` per un turno con ordini misti.
- [ ] **OrdiniDao (Deduplicazione Cliente)**: Inserimento di due ordini con lo stesso recapito telefonico; verifica che la tabella `clienti` contenga un unico record associato a entrambi gli ordini.
- [ ] **OrdiniDao (Transazione Sequenza)**: Inserimento di 4 ordini; eliminazione dell'ordine #2; verifica che gli ordini precedentemente marcati come #3 e #4 abbiano ora sequenza rispettivamente #2 e #3.
- [ ] **OrdiniDao (Batch Update)**: Inversione sequenza ordini tramite transazione; verifica consistenza ordinamento.
- [ ] **DTO Serializzazione**: Test `fromJson` e `toJson` per tutti i DTO con caratteri speciali e valori nulli opzionali.

### 2. Test di Integrazione API (Shelf Server)
- [ ] Avvio server Shelf su porta di test (`127.0.0.1:8081`).
- [ ] Chiamata `POST /api/v1/turni` con body valido: status 201 e header `Content-Type: application/json`.
- [ ] Chiamata `POST /api/v1/ordini` con importo negativo (`-5.00`): status 422 con array di errori esplicativi.
- [ ] Chiamata `POST /api/v1/ordini` con orario errato (`25:70`): status 422.
- [ ] Chiamata `PUT /api/v1/ordini/sequenza` con array vuoto o malformato: status 400.
- [ ] Chiamata `DELETE /api/v1/turni/:id` su ID inesistente: status 404.

### 3. Test Widget & State Management (Riverpod)
- [ ] **TurniDashboardScreen**: Verifica rendering corretto in stato di caricamento (`CircularProgressIndicator`), vuoto (`Nessun turno trovato`) e popolato.
- [ ] **CreaTurnoDialog**: Validazione selezione data/ora e disabilitazione pulsante durante il salvataggio (`_isSaving`).
- [ ] **TurnoDettaglioScreen**: Spostamento di una riga nella `ReorderableListView`; verifica che il notifier invii il payload batch corretto al servizio mockato.
- [ ] **Rilevamento Ritardo**: Creazione di un ordine con orario passato di 15 minuti; verifica comparsa badge rosso `IN RITARDO`; transizione a stato `consegnato` e verifica scomparsa badge di ritardo.
- [ ] **FormNuovoOrdineDialog**: Test validatori Form (nome vuoto, telefono corto, importo 0, caratteri alfabetici nel campo importo).

### 4. Test End-to-End su Windows Desktop
- [ ] Avvio applicativo con `flutter run -d windows`.
- [ ] Creazione del turno della serata corrente.
- [ ] Inserimento consecutivo di 3 ordini (es. Cliente Mario Rossi € 22.00 ore 20:00, Luigi Bianchi € 35.50 ore 20:15, Anna Verdi € 18.00 ore 20:30).
- [ ] Trascinamento di Anna Verdi dalla terza alla prima posizione; verifica aggiornamento dei numeri di sequenza (#1 Anna Verdi, #2 Mario Rossi, #3 Luigi Bianchi).
- [ ] Cambio di stato di Anna Verdi da `in_attesa` a `in_consegna` e successivamente a `consegnato`.
- [ ] Chiusura forzata dell'applicazione e riavvio: verifica completa della persistenza di turni, anagrafiche clienti, ordini e sequenze ordinate su SQLite.
