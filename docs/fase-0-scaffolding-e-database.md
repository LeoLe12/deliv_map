# Fase 0 — Scaffolding, Architettura e Database

**Progetto:** DelivMap — Gestione consegne a domicilio per piccoli ristoranti  
**Fase:** 0 (Scaffolding, Architettura e Database Locale)  
**Target:** Flutter Desktop (Windows 10/11 x64 primary target)  
**Durata stimata:** 2-3 giorni lavorativi  
**Stato:** Pianificazione esecutiva (Aggiornata post-revisione)  

---

## Panoramica della Fase 0

La Fase 0 stabilisce le fondamenta architetturali, lo scheletro dell'applicazione, la persistenza relazionale e il canale di comunicazione locale tra l'interfaccia utente Flutter e il backend embedded.

Seguendo il paradigma **local-first**, DelivMap non si appoggia a server cloud esterni per i dati operativi. Tutto risiede ed opera sulla workstation del punto vendita mediante un'architettura a tre livelli rigorosamente disaccoppiata:

1. **Frontend Flutter Desktop**: Interfaccia grafica reattiva compilata nativamente in C++/DirectX su Windows, basata su Material 3 e Riverpod.
2. **Server API Shelf su Isolate dedicato**: Processo server HTTP/1.1 in ascolto su `http://127.0.0.1:8080`, isolato dal thread grafico per evitare blocchi dell'interfaccia durante elaborazioni pesanti.
3. **Database Relazionale Drift/SQLite**: Motore relazionale SQLite con tabelle tipizzate, foreign key, indici e transazioni atomiche gestito tramite Drift ORM, memorizzato nella directory dati applicativa dell'utente (`%APPDATA%` / `Application Support`).

> [!IMPORTANT]
> **Regola architetturale aurea (Nessun accesso diretto DB dalla UI):**
> L'istanza di `AppDatabase` risiede ed è gestita **esclusivamente all'interno dell'Isolate del server Shelf**. L'interfaccia grafica non apre né istanzia mai il database SQLite: comunica con esso unicamente tramite chiamate HTTP locali su `localhost:8080`. Questo garantisce assenza di lock sul file SQLite ed evita riscritture quando nella Fase 2/v2 l'app fattorini comunicherà via rete LAN con la medesima API.

---

## Decisioni di Revisione Applicate

In seguito alla revisione tecnica del piano, sono state integrate le seguenti direttive vincolanti:

1. **Eliminazione di `database_provider.dart`**: Nessuna esposizione del database relazionale al layer dei provider UI.
2. **Scaffolding puro (nessun CRUD prematuro)**: Le route CRUD applicative (ordini, clienti, turni, impostazioni), i rispettivi DAO e i DTO completi appartengono alla **Fase 1** e **Fase 2**. In Fase 0 vengono predisposti unicamente l'endpoint di diagnostica `/health` e il montaggio base del router `/api/v1`.
3. **Rilevamento istanza singola (Single Instance)**: Gestione nativa del doppio avvio dell'applicazione intercettando il fallimento del socket bind su porta 8080 (`SocketException`).
4. **Hashing password con `bcrypt`**: Adozione di `bcrypt` in luogo di hash deboli. Il campo `password_hash` conterrà la stringa modulare bcrypt standard a 60 caratteri (che include già internamente cost factor a 12, salt a 128 bit e hash).
5. **Correzione serializzazione enum `StatoOrdine`**: Sostituzione di `textEnum` con un `TypeConverter` dedicato (`StatoOrdineConverter`), garantendo che nel database SQLite vengano scritte le stringhe snake_case (`in_attesa`, `in_consegna`, `consegnato`, `in_ritardo`) e non i nomi camelCase dell'enum Dart.
6. **Gestione dipendenze elastica**: Installazione delle librerie tramite `flutter pub add` per consentire al package manager di risolvere le versioni stabili più recenti compatibili con l'SDK installato.

---

## Task 0.1 — Creazione Progetto Flutter e Struttura Cartelle

### 1.1 Inizializzazione del Progetto con Flutter CLI

Il progetto viene inizializzato con il nome `deliv_map`, l'organizzazione `com.delivmap` e abilitando il target desktop Windows.

Eseguire i seguenti comandi dalla cartella genitore dei progetti:

```powershell
# Verifica e abilitazione del supporto desktop Windows nell'ambiente Flutter
flutter config --enable-windows-desktop

# Verifica che i toolchain nativi siano installati (Visual Studio con "Desktop development with C++")
flutter doctor -v

# Creazione del progetto con Windows come target desktop prioritario
flutter create --org com.delivmap --project-name deliv_map --platforms=windows .
```

> [!IMPORTANT]
> Su Windows è obbligatorio disporre di **Visual Studio 2022 Community** (o Professional/Enterprise) con il carico di lavoro *"Desktop development with C++"* (Sviluppo per desktop con C++) installato, inclusi MSVC v143, Windows 10/11 SDK e C++ CMake tools for Windows.

### 1.2 Albero delle Directory di Progetto (`lib/`)

L'albero dei file della Fase 0 riflette lo scaffolding puro senza file CRUD prematuri:

```text
lib/
├── main.dart                          # Entrypoint applicazione, bootstrap Isolate server, single-instance check e Window manager
├── api/                               # Backend locale Shelf HTTP
│   ├── server.dart                    # Configurazione e avvio server Shelf su localhost:8080 in Isolate
│   ├── routes/                        # Router modulari
│   │   ├── api_router.dart            # Router principale montato su /api/v1
│   │   └── health_route.dart          # Endpoint /health e diagnostica
│   └── middleware/                    # Middleware Shelf
│       ├── error_middleware.dart      # Intercettazione eccezioni e risposte JSON strutturate
│       ├── logging_middleware.dart    # Logging strutturato delle richieste HTTP su stdout
│       └── cors_middleware.dart       # Gestione header CORS per richieste locali
├── data/                              # Livello di persistenza locale (Drift / SQLite)
│   ├── database.dart                  # Classe AppDatabase, schema version = 1 e LazyDatabase
│   ├── converters/                    # TypeConverter personalizzati Drift
│   │   └── stato_ordine_converter.dart# Mappatura StatoOrdine <-> String snake_case
│   └── tables/                        # Definizioni tabelle Drift v1
│       ├── ristoranti.dart            # Tabella 'ristoranti' (configurazione locale / BYOK Maps)
│       ├── utenti.dart                # Tabella 'utenti' (gestore / credenziali con password_hash bcrypt)
│       ├── clienti.dart               # Tabella 'clienti' (anagrafica con indice su telefono)
│       ├── turni.dart                 # Tabella 'turni' (sessioni di consegna)
│       └── ordini.dart                # Tabella 'ordini' (ordini con FK, sequenza e StatoOrdineConverter)
├── models/                            # DTOs e modelli condivisi
│   ├── api_response.dart              # Wrapper unificato per risposte HTTP JSON
│   └── stato_ordine.dart              # Enum degli stati di avanzamento consegna con codice snake_case
├── services/                          # Servizi infrastrutturali di dominio (Fase 3-5)
│   └── storage_service.dart           # Risoluzione percorsi sicuri per il database su %APPDATA%
└── ui/                                # Livello di presentazione Flutter Desktop
    ├── app.dart                       # Root widget MaterialApp.router e configurazione tema
    ├── router.dart                    # Definizione GoRouter con StatefulShellRoute desktop
    ├── providers/                     # Notifier Riverpod (gestione stato UI)
    │   ├── theme_provider.dart        # Gestione Dark/Light mode
    │   └── server_status_provider.dart# Monitoraggio liveness server Shelf
    ├── screens/                       # Schermate shell dell'applicazione
    │   ├── shell_screen.dart          # Shell principale con NavigationRail laterale
    │   ├── dashboard_screen.dart      # Placeholder vista dashboard
    │   ├── turni_screen.dart          # Placeholder vista turni (sviluppata in Fase 1)
    │   ├── clienti_screen.dart        # Placeholder vista clienti (sviluppata in Fase 2)
    │   ├── mappa_screen.dart          # Placeholder vista mappa (sviluppata in Fase 3)
    │   └── impostazioni_screen.dart   # Placeholder vista impostazioni (sviluppata in Fase 4)
    └── widgets/                       # Widget riutilizzabili
        ├── single_instance_dialog.dart# Dialog per avviso seconda istanza già aperta
        └── server_indicator.dart      # Indicatore visivo stato server locale
```

### 1.3 Script di Generazione Automatica delle Cartelle

Eseguire lo script PowerShell seguente per generare l'intera alberatura:

```powershell
$base = "c:\Users\39366\StudioProjects\DelivMap"

$dirs = @(
    "$base\lib\api\routes",
    "$base\lib\api\middleware",
    "$base\lib\data\tables",
    "$base\lib\data\converters",
    "$base\lib\models",
    "$base\lib\services",
    "$base\lib\ui\providers",
    "$base\lib\ui\screens",
    "$base\lib\ui\widgets"
)

foreach ($dir in $dirs) {
    if (-not (Test-Path $dir)) {
        New-Item -ItemType Directory -Path $dir -Force | Out-Null
    }
}

Write-Host "Cartelle di architettura generate con successo." -ForegroundColor Green
```

---

## Task 0.2 — Configurazione delle Dipendenze

Invece di fissare versioni statiche obsolete, si raccomanda l'installazione tramite il comando `flutter pub add`, permettendo l'allineamento con le versioni più recenti e stabili compatibili con l'SDK in uso.

### 2.1 Comandi di Installazione Dipendenze

Eseguire da terminale:

```powershell
# Dipendenze applicative principali
flutter pub add drift
flutter pub add sqlite3
flutter pub add path_provider
flutter pub add path
flutter pub add shelf
flutter pub add shelf_router
flutter pub add flutter_riverpod
flutter pub add go_router
flutter pub add window_manager
flutter pub add bcrypt
flutter pub add http
flutter pub add json_annotation
flutter pub add intl

# Dipendenze di sviluppo (compilatori e generatori di codice)
flutter pub add --dev drift_dev
flutter pub add --dev build_runner
flutter pub add --dev json_serializable
```

### 2.2 Quadro Sinottico delle Librerie

| Dipendenza | Categoria | Scopo Architetturale |
|---|---|---|
| `drift` | Database ORM | Schema tipizzato, query SQL compilate e migrazioni incrementali |
| `sqlite3` | Database Core | Binding Dart diretti all'API C di SQLite con build hook nativo (sostituisce `sqlite3_flutter_libs`) |
| `path_provider` | System Storage | Individuazione directory sicura per SQLite (`getApplicationSupportDirectory`) nel main isolate |
| `path` | System Utilities | Manipolazione percorsi filesystem cross-platform |
| `shelf` + `shelf_router` | API Server | Motore server HTTP RESTful locale su localhost |
| `flutter_riverpod` | State Management | Gestione stato reattivo UI e dependency injection |
| `go_router` | Routing | Routing dichiarativo con supporto per NavigationRail desktop |
| `window_manager` | Desktop Integration | Gestione dimensioni minime, titolo e centratura finestra Windows |
| `bcrypt` | Crittografia | Hashing sicuro delle password utenti con salt a 128 bit e cost factor 12 (`BCrypt.gensalt(12)`) |
| `http` | HTTP Client | Comunicazione tra UI Flutter e Server Shelf locale |
| `json_annotation` | Serializzazione | Decoratori JSON per DTO |
| `intl` | Localizzazione | Formattazione date e valute in lingua italiana (`it_IT`) |

---

## Task 0.3 — Tabelle Drift e Database Locale

### 3.1 Enum `StatoOrdine` e relativo `TypeConverter`

Creare il file `lib/models/stato_ordine.dart`:

```dart
/// Enum per gli stati operativi di avanzamento di una consegna (macchina a stati pura).
/// NOTA: Il ritardo non è uno stato del flusso ma un flag temporale calcolato con tolleranza (default 5 min).
enum StatoOrdine {
  inAttesa('in_attesa', 'In Attesa'),
  inConsegna('in_consegna', 'In Consegna'),
  consegnato('consegnato', 'Consegnato');

  final String codice;
  final String label;

  const StatoOrdine(this.codice, this.label);

  static StatoOrdine fromString(String codice) {
    return StatoOrdine.values.firstWhere(
      (e) => e.codice == codice,
      orElse: () => throw ArgumentError('Stato sconosciuto: $codice'),
    );
  }
}
```

Creare il converter `lib/data/converters/stato_ordine_converter.dart`:

```dart
import 'package:drift/drift.dart';
import '../../models/stato_ordine.dart';

/// TypeConverter per garantire che Drift serializzi e deserializzi
/// l'enum StatoOrdine usando esattamente la stringa snake_case 'codice' nel DB SQLite.
class StatoOrdineConverter extends TypeConverter<StatoOrdine, String> {
  const StatoOrdineConverter();

  @override
  StatoOrdine fromSql(String fromDb) {
    return StatoOrdine.fromString(fromDb);
  }

  @override
  String toSql(StatoOrdine value) {
    return value.codice;
  }
}
```

### 3.2 Tabelle Drift v1

#### A. Tabella Ristoranti (`lib/data/tables/ristoranti.dart`)

```dart
import 'package:drift/drift.dart';

/// Tabella per la configurazione del ristorante e le impostazioni geografiche/API.
@DataClassName('Ristorante')
class Ristoranti extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get nome => text().withLength(min: 1, max: 120)();
  TextColumn get indirizzo => text()();
  
  /// Poligono GeoJSON o raggio descrittivo della zona di consegna
  TextColumn get zonaConsegna => text().named('zona_consegna')();
  
  /// Chiave Google Maps Platform inserita dal titolare (modello BYOK)
  TextColumn get mapsApiKey => text().named('maps_api_key').nullable()();

  /// Se true, evidenzia visivamente gli ordini in ritardo rispetto all'orario richiesto + tolleranza
  BoolColumn get evidenziaRitardi => boolean().named('evidenzia_ritardi').withDefault(const Constant(true))();

  /// Tolleranza in minuti prima di far scattare la segnalazione di ritardo (default: 5 minuti)
  IntColumn get tolleranzaRitardoMinuti => integer().named('tolleranza_ritardo_minuti').withDefault(const Constant(5))();
}
```

#### B. Tabella Utenti (`lib/data/tables/utenti.dart`)

> [!NOTE]
> La colonna `passwordHash` memorizza la stringa modulare generata da `bcrypt`. Il formato standard (es. `$2a$12$e8N8...`) contiene già algoritmi, cost factor, salt e hash nei suoi 60 caratteri: non occorrono colonne separate per salt o cost.

```dart
import 'package:drift/drift.dart';

/// Tabella degli utenti con accesso all'applicazione desktop.
@DataClassName('Utente')
class Utenti extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get nomeUtente => text().named('nome_utente').unique().withLength(min: 3, max: 50)();
  
  /// Stringa modulare completa generata da bcrypt (cost factor 12 + salt 128 bit + hash)
  TextColumn get passwordHash => text().named('password_hash')();
  
  TextColumn get ruolo => text().withDefault(const Constant('gestore'))();
}
```

#### C. Tabella Clienti (`lib/data/tables/clienti.dart`)

```dart
import 'package:drift/drift.dart';

/// Tabella anagrafica clienti con indicizzazione sul recapito telefonico (rubrica rapida).
@DataClassName('Cliente')
@TableIndex(name: 'idx_clienti_telefono', columns: {#telefono})
class Clienti extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get nome => text().withLength(min: 1, max: 100)();
  TextColumn get telefono => text().withLength(min: 5, max: 25)();
  TextColumn get indirizzo => text()();
}
```

#### D. Tabella Turni (`lib/data/tables/turni.dart`)

```dart
import 'package:drift/drift.dart';

/// Tabella che raggruppa le sessioni operative di consegna (es. pranzo / cena).
@DataClassName('Turno')
class Turni extends Table {
  IntColumn get id => integer().autoIncrement()();
  DateTimeColumn get data => dateTime()();
  TextColumn get note => text().nullable()();
}
```

#### E. Tabella Ordini (`lib/data/tables/ordini.dart`)

```dart
import 'package:drift/drift.dart';
import '../../models/stato_ordine.dart';
import '../converters/stato_ordine_converter.dart';
import 'clienti.dart';
import 'turni.dart';

/// Tabella principale delle consegne con snapshot anagrafico e sequenziamento.
@DataClassName('Ordine')
@TableIndex(name: 'idx_ordini_turno', columns: {#turnoId})
class Ordini extends Table {
  IntColumn get id => integer().autoIncrement()();

  /// Riferimento alla rubrica clienti (nullable: l'ordine resta intatto anche se il cliente viene cancellato)
  IntColumn get clienteId => integer()
      .named('cliente_id')
      .nullable()
      .references(Clienti, #id, onDelete: KeyAction.setNull)();

  /// Riferimento al turno operativo
  IntColumn get turnoId => integer()
      .named('turno_id')
      .references(Turni, #id, onDelete: KeyAction.cascade)();

  /// Nome del cliente congelato come snapshot al momento dell'ordine
  TextColumn get nomeClienteConsegna => text().named('nome_cliente_consegna').withLength(min: 1, max: 120)();

  /// Telefono di consegna congelato come snapshot
  TextColumn get telefonoConsegna => text().named('telefono_consegna').withLength(min: 3, max: 30)();

  /// Indirizzo effettivo di recapito congelato come snapshot
  TextColumn get indirizzoConsegna => text().named('indirizzo_consegna').withLength(min: 3, max: 250)();

  /// Importo monetario dell'ordine (in euro, es. 24.50)
  RealColumn get importo => real()();

  /// Orario richiesto dal cliente per la consegna nel formato HH:mm (es. '20:30')
  TextColumn get orarioRichiesto => text().named('orario_richiesto')();

  /// Stato ordine serializzato via StatoOrdineConverter (es. 'in_attesa', 'in_consegna', 'consegnato')
  TextColumn get stato => text().map(const StatoOrdineConverter())();

  /// Posizione ordinale nella sequenza di consegna (0, 1, 2, ...) per il turno
  IntColumn get sequenza => integer().withDefault(const Constant(0))();

  /// Note aggiuntive (es. citofono guasto, scala B, piano 3)
  TextColumn get note => text().nullable()();
}
```

### 3.3 File del Database (`lib/data/database.dart`)

Configurazione di `AppDatabase` con connessione nativa SQLite eseguita in background:

```dart
import 'dart:io';

import 'package:drift/drift.dart';
import 'package:drift/native.dart';
import 'package:path/path.dart' as p;
import 'package:path_provider/path_provider.dart';

import '../models/stato_ordine.dart';
import 'converters/stato_ordine_converter.dart';
import 'tables/clienti.dart';
import 'tables/ordini.dart';
import 'tables/ristoranti.dart';
import 'tables/turni.dart';
import 'tables/utenti.dart';

part 'database.g.dart';

/// Classe centrale del database Drift per DelivMap.
/// Istanza esclusiva dell'Isolate del server Shelf.
@DriftDatabase(tables: [
  Ristoranti,
  Utenti,
  Clienti,
  Turni,
  Ordini,
])
class AppDatabase extends _$AppDatabase {
  final String dbPath;

  AppDatabase({required this.dbPath, QueryExecutor? executor})
      : super(executor ?? _openConnection(dbPath));

  @override
  int get schemaVersion => 1;

  @override
  MigrationStrategy get migration {
    return MigrationStrategy(
      beforeOpen: (details) async {
        // Abilita i vincoli di integrità referenziale (Foreign Keys) su SQLite
        await customStatement('PRAGMA foreign_keys = ON;');
        // Modalità WAL per massime prestazioni in lettura e scrittura concorrente
        await customStatement('PRAGMA journal_mode = WAL;');
      },
      onCreate: (Migrator m) async {
        await m.createAll();
        // Inserimento configurazione iniziale del ristorante di default
        await into(ristoranti).insert(
          RistorantiCompanion.insert(
            nome: 'Il Mio Ristorante',
            indirizzo: 'Via Roma 1, Roma',
            zonaConsegna: '5',
          ),
        );
      },
    );
  }
}

/// Costruttore LazyDatabase che riceve il percorso calcolato dal main isolate
LazyDatabase _openConnection(String dbPath) {
  return LazyDatabase(() async {
    final file = File(dbPath);

    // Creazione automatica directory se non esistente
    if (!await file.parent.exists()) {
      await file.parent.create(recursive: true);
    }

    return NativeDatabase.createInBackground(file);
  });
}
```

---

## Task 0.4 — Generazione del Codice Drift

Eseguire il comando di generazione codice per creare il file `database.g.dart`:

```powershell
dart run build_runner build --delete-conflicting-outputs
```

Verificare che `lib/data/database.g.dart` sia stato generato correttamente e contenga le classi companion (`OrdiniCompanion`, `ClientiCompanion`, ecc.) e la mappatura con `StatoOrdineConverter`.

---

## Task 0.5 — Server Shelf su Isolate e Rilevamento Istanza Singola

### 5.1 Router Principale (`lib/api/routes/api_router.dart`) e Health Route (`lib/api/routes/health_route.dart`)

Creare `lib/api/routes/health_route.dart`:

```dart
import 'dart:convert';
import 'package:shelf/shelf.dart';
import 'package:shelf_router/shelf_router.dart';

class HealthRoute {
  Response health(Request request) {
    final data = {
      'status': 'ok',
      'app': 'DelivMap',
      'version': '1.0.0',
      'timestamp': DateTime.now().toUtc().toIso8601String(),
      'mode': 'local-first',
    };
    return Response.ok(
      jsonEncode(data),
      headers: {'Content-Type': 'application/json; charset=utf-8'},
    );
  }

  Response ping(Request request) {
    return Response.ok(
      jsonEncode({'message': 'pong'}),
      headers: {'Content-Type': 'application/json; charset=utf-8'},
    );
  }
}
```

Creare `lib/api/routes/api_router.dart`:

```dart
import 'package:shelf/shelf.dart';
import 'package:shelf_router/shelf_router.dart';
import '../../data/database.dart';
import 'health_route.dart';

class ApiRouter {
  final AppDatabase db;

  ApiRouter(this.db);

  Router get router {
    final router = Router();
    final healthRoute = HealthRoute();

    // Health check diretti (senza mount annidati per evitare /health/health)
    router.get('/health', healthRoute.health);
    router.get('/api/v1/health', healthRoute.health);
    router.get('/ping', healthRoute.ping);

    // Le rotte applicative (turni, ordini, clienti) verranno montate qui nella Fase 1 e 2
    // router.mount('/api/v1/turni', TurniApi(db.turniDao).router);
    // router.mount('/api/v1/ordini', OrdiniApi(db.ordiniDao).router);

    return router;
  }

  Handler get handler => router.call;
}
```

### 5.2 Server Shelf (`lib/api/server.dart`) con Rilevamento Porta Occupata

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';
import 'dart:isolate';

import 'package:shelf/shelf.dart';
import 'package:shelf/shelf_io.dart' as io;

import '../data/database.dart';
import 'routes/api_router.dart';

class LocalApiServer {
  final int port;
  final String dbPath;
  final String authToken;
  HttpServer? _server;
  AppDatabase? _database;

  LocalApiServer({
    this.port = 8080,
    required this.dbPath,
    required this.authToken,
  });

  AppDatabase get db => _database!;

  Future<HttpServer> start() async {
    // Il database vive e viene istanziato esclusivamente qui dentro
    _database = AppDatabase(dbPath: dbPath);

    final apiRouter = ApiRouter(_database!);

    final pipeline = const Pipeline()
        .addMiddleware(_authMiddleware(authToken))
        .addMiddleware(_jsonHeaderMiddleware())
        .addHandler(apiRouter.handler);

    // Bind su localhost: se la porta è già occupata, genera SocketException
    _server = await io.serve(pipeline, InternetAddress.loopbackIPv4, port);
    return _server!;
  }

  Future<void> stop() async {
    await _server?.close(force: true);
    await _database?.close();
  }

  static Middleware _jsonHeaderMiddleware() {
    return (Handler innerHandler) {
      return (Request request) async {
        final response = await innerHandler(request);
        if (!response.headers.containsKey('Content-Type')) {
          return response.change(headers: {
            'Content-Type': 'application/json; charset=utf-8',
          });
        }
        return response;
      };
    };
  }

  /// Middleware di sicurezza locale: blocca richieste non autorizzate da browser o pagine web aperte in locale.
  /// Le route di health check (/health, /api/v1/health, /ping) sono pubbliche per la diagnostica d'avvio.
  static Middleware _authMiddleware(String expectedToken) {
    return (Handler innerHandler) {
      return (Request request) async {
        final normalizedPath = request.url.path;
        if (normalizedPath == 'health' || 
            normalizedPath == 'api/v1/health' || 
            normalizedPath == 'ping') {
          return await innerHandler(request);
        }

        final authHeader = request.headers['Authorization'];
        if (authHeader == null || authHeader != 'Bearer $expectedToken') {
          return Response.forbidden(
            jsonEncode({'error': 'UNAUTHORIZED', 'message': 'Richiesta locale non autorizzata'}),
            headers: {'Content-Type': 'application/json; charset=utf-8'},
          );
        }

        return await innerHandler(request);
      };
    };
  }
}

/// Messaggi inviati dall'Isolate al thread UI
enum ServerStatusType {
  ready,
  portInUse,
  error,
}

class ServerIsolateConfig {
  final SendPort statusPort;
  final int port;
  final String dbPath;
  final String authToken;

  ServerIsolateConfig({
    required this.statusPort,
    required this.port,
    required this.dbPath,
    required this.authToken,
  });
}

/// Entrypoint Isolate per il server Shelf
void runServerIsolate(ServerIsolateConfig config) async {
  final commandPort = ReceivePort();

  try {
    final server = LocalApiServer(
      port: config.port,
      dbPath: config.dbPath,
      authToken: config.authToken,
    );
    final httpServer = await server.start();

    config.statusPort.send({
      'status': ServerStatusType.ready.name,
      'port': httpServer.port,
      'commandPort': commandPort.sendPort,
    });

    await for (final message in commandPort) {
      if (message == 'stop') {
        await server.stop();
        commandPort.close();
        break;
      }
    }
  } on SocketException catch (e) {
    // Intercettazione dell'errore porta occupata (WSAEADDRINUSE su Windows)
    config.statusPort.send({
      'status': ServerStatusType.portInUse.name,
      'message': 'Porta ${config.port} già occupata.',
      'error': e.toString(),
    });
    commandPort.close();
  } catch (e, stack) {
    config.statusPort.send({
      'status': ServerStatusType.error.name,
      'error': e.toString(),
      'stack': stack.toString(),
    });
    commandPort.close();
  }
}
```

### 5.3 Widget di Avviso Istanza Singola o Conflitto Porta (`lib/ui/widgets/single_instance_dialog.dart`)

```dart
import 'dart:io';
import 'package:flutter/material.dart';

/// Applicazione minimale visualizzata se l'avvio rileva un'altra istanza o la porta occupata
class SingleInstanceAlertApp extends StatelessWidget {
  final bool isDelivMapRunning;

  const SingleInstanceAlertApp({
    super.key,
    required this.isDelivMapRunning,
  });

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      theme: ThemeData(useMaterial3: true, colorSchemeSeed: Colors.amber),
      home: Scaffold(
        body: Center(
          child: Padding(
            padding: const EdgeInsets.all(32.0),
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                Icon(
                  isDelivMapRunning ? Icons.warning_amber_rounded : Icons.error_outline,
                  size: 64,
                  color: isDelivMapRunning ? Colors.orange : Colors.red,
                ),
                const SizedBox(height: 16),
                Text(
                  isDelivMapRunning
                      ? 'DelivMap è già in esecuzione'
                      : 'Porta 8080 occupata da un altro programma',
                  style: const TextStyle(fontSize: 20, fontWeight: FontWeight.bold),
                ),
                const SizedBox(height: 12),
                Text(
                  isDelivMapRunning
                      ? 'È stata rilevata un\'altra istanza di DelivMap già aperta sulla workstation.\nPer evitare conflitti, questa finestra verrà chiusa.'
                      : 'Un\'altra applicazione sulla workstation sta già utilizzando la porta locale 8080.\nChiudi l\'applicazione in conflitto e riavvia DelivMap.',
                  textAlign: TextAlign.center,
                ),
                const SizedBox(height: 24),
                ElevatedButton.icon(
                  onPressed: () => exit(0),
                  icon: const Icon(Icons.check),
                  label: const Text('OK, Ho Capito'),
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }
}
```

### 5.4 Bootstrap nel `main.dart` con Generazione Token, Path Isolate e Verifica Conflitti

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';
import 'dart:isolate';
import 'dart:math';

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:http/http.dart' as http;
import 'package:intl/date_symbol_data_local.dart';
import 'package:path/path.dart' as p;
import 'package:path_provider/path_provider.dart';
import 'package:window_manager/window_manager.dart';

import 'api/server.dart';
import 'ui/app.dart';
import 'ui/widgets/single_instance_dialog.dart';

SendPort? _serverCommandPort;
Isolate? _serverIsolate;

/// Token di autenticazione locale generato randomicamente all'avvio in RAM.
/// Viene condiviso unicamente tra UI e Server Isolate via memoria di processo.
String? appLocalAuthToken;

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await windowManager.ensureInitialized();

  // Inizializzazione della localizzazione italiana per DateFormat (intl)
  await initializeDateFormatting('it_IT', null);

  const windowOptions = WindowOptions(
    size: Size(1280, 800),
    minimumSize: Size(1024, 720),
    center: true,
    backgroundColor: Colors.transparent,
    skipTaskbar: false,
    title: 'DelivMap — Gestione Consegne',
    titleBarStyle: TitleBarStyle.normal,
  );

  await windowManager.waitUntilReadyToShow(windowOptions, () async {
    await windowManager.show();
    await windowManager.focus();
  });

  // Risoluzione sicura del path SQLite nel Main Isolate (nessun plugin nei worker Isolate)
  final appSupportDir = await getApplicationSupportDirectory();
  final dbPath = p.join(appSupportDir.path, 'delivmap.sqlite');

  // Generazione token segreto casuale per il loopback HTTP locale (32 byte crittografici)
  final random = Random.secure();
  final tokenBytes = List<int>.generate(32, (_) => random.nextInt(256));
  appLocalAuthToken = base64UrlEncode(tokenBytes);

  // Avvio server locale su isolate
  final serverResult = await _bootstrapServer(
    port: 8080,
    dbPath: dbPath,
    authToken: appLocalAuthToken!,
  );

  if (serverResult == ServerStatusType.portInUse) {
    // Verifica se la porta è occupata da DelivMap o da un altro software terzo
    final isDelivMap = await _checkIfDelivMapIsRunning(8080);
    runApp(SingleInstanceAlertApp(isDelivMapRunning: isDelivMap));
    return;
  }

  // Avvio regolare dell'applicazione con Riverpod ProviderScope
  runApp(
    const ProviderScope(
      child: DelivMapApp(),
    ),
  );
}

/// Esegue una probe su /health per verificare se chi risponde è davvero DelivMap
Future<bool> _checkIfDelivMapIsRunning(int port) async {
  try {
    final response = await http
        .get(Uri.parse('http://127.0.0.1:$port/health'))
        .timeout(const Duration(seconds: 2));
    if (response.statusCode == 200) {
      final json = jsonDecode(response.body) as Map<String, dynamic>;
      return json['app'] == 'DelivMap';
    }
  } catch (_) {
    // Non risponde o risponde con protocollo diverso
  }
  return false;
}

Future<ServerStatusType> _bootstrapServer({
  required int port,
  required String dbPath,
  required String authToken,
}) async {
  final statusReceivePort = ReceivePort();
  final completer = Completer<ServerStatusType>();

  final config = ServerIsolateConfig(
    statusPort: statusReceivePort.sendPort,
    port: port,
    dbPath: dbPath,
    authToken: authToken,
  );

  _serverIsolate = await Isolate.spawn(
    runServerIsolate,
    config,
    debugName: 'DelivMap_Shelf_Isolate',
  );

  statusReceivePort.listen((message) {
    if (message is Map) {
      final status = message['status'];
      if (status == ServerStatusType.ready.name) {
        _serverCommandPort = message['commandPort'] as SendPort;
        if (!completer.isCompleted) completer.complete(ServerStatusType.ready);
      } else if (status == ServerStatusType.portInUse.name) {
        if (!completer.isCompleted) completer.complete(ServerStatusType.portInUse);
      } else {
        if (!completer.isCompleted) completer.complete(ServerStatusType.error);
      }
    }
  });

  try {
    return await completer.future.timeout(const Duration(seconds: 5));
  } catch (_) {
    return ServerStatusType.error;
  }
}
```

---

## Task 0.6 — Shell Flutter Desktop

### 6.1 Router con GoRouter e `StatefulShellRoute` (`lib/ui/router.dart`)

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

import 'screens/clienti_screen.dart';
import 'screens/dashboard_screen.dart';
import 'screens/impostazioni_screen.dart';
import 'screens/mappa_screen.dart';
import 'screens/shell_screen.dart';
import 'screens/turni_screen.dart';

final _rootNavigatorKey = GlobalKey<NavigatorState>();

final appRouter = GoRouter(
  navigatorKey: _rootNavigatorKey,
  initialLocation: '/dashboard',
  routes: [
    StatefulShellRoute.indexedStack(
      builder: (context, state, navigationShell) {
        return ShellScreen(navigationShell: navigationShell);
      },
      branches: [
        StatefulShellBranch(
          routes: [
            GoRoute(
              path: '/dashboard',
              name: 'dashboard',
              pageBuilder: (context, state) => const NoTransitionPage(
                child: DashboardScreen(),
              ),
            ),
          ],
        ),
        StatefulShellBranch(
          routes: [
            GoRoute(
              path: '/turni',
              name: 'turni',
              pageBuilder: (context, state) => const NoTransitionPage(
                child: TurniScreen(),
              ),
            ),
          ],
        ),
        StatefulShellBranch(
          routes: [
            GoRoute(
              path: '/clienti',
              name: 'clienti',
              pageBuilder: (context, state) => const NoTransitionPage(
                child: ClientiScreen(),
              ),
            ),
          ],
        ),
        StatefulShellBranch(
          routes: [
            GoRoute(
              path: '/mappa',
              name: 'mappa',
              pageBuilder: (context, state) => const NoTransitionPage(
                child: MappaScreen(),
              ),
            ),
          ],
        ),
        StatefulShellBranch(
          routes: [
            GoRoute(
              path: '/impostazioni',
              name: 'impostazioni',
              pageBuilder: (context, state) => const NoTransitionPage(
                child: ImpostazioniScreen(),
              ),
            ),
          ],
        ),
      ],
    ),
  ],
);
```

### 6.2 Shell Principale Desktop con `NavigationRail` (`lib/ui/screens/shell_screen.dart`)

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';
import '../widgets/server_indicator.dart';

class ShellScreen extends StatelessWidget {
  final StatefulNavigationShell navigationShell;

  const ShellScreen({
    super.key,
    required this.navigationShell,
  });

  void _onDestinationSelected(int index) {
    navigationShell.goBranch(
      index,
      initialLocation: index == navigationShell.currentIndex,
    );
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);

    return Scaffold(
      body: Row(
        children: [
          NavigationRail(
            selectedIndex: navigationShell.currentIndex,
            onDestinationSelected: _onDestinationSelected,
            extended: true,
            minExtendedWidth: 200,
            leading: Padding(
              padding: const EdgeInsets.symmetric(vertical: 24.0, horizontal: 16.0),
              child: Row(
                children: [
                  Container(
                    padding: const EdgeInsets.all(8),
                    decoration: BoxDecoration(
                      color: theme.colorScheme.primaryContainer,
                      borderRadius: BorderRadius.circular(8),
                    ),
                    child: Icon(
                      Icons.delivery_dining,
                      color: theme.colorScheme.onPrimaryContainer,
                      size: 28,
                    ),
                  ),
                  const SizedBox(width: 12),
                  Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Text(
                        'DelivMap',
                        style: theme.textTheme.titleMedium?.copyWith(
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                      Text(
                        'Local Edition',
                        style: theme.textTheme.bodySmall?.copyWith(
                          color: theme.colorScheme.outline,
                        ),
                      ),
                    ],
                  ),
                ],
              ),
            ),
            trailing: const Expanded(
              child: Align(
                alignment: Alignment.bottomLeft,
                child: Padding(
                  padding: EdgeInsets.all(16.0),
                  child: ServerIndicator(),
                ),
              ),
            ),
            destinations: const [
              NavigationRailDestination(
                icon: Icon(Icons.dashboard_outlined),
                selectedIcon: Icon(Icons.dashboard),
                label: Text('Dashboard'),
              ),
              NavigationRailDestination(
                icon: Icon(Icons.access_time_outlined),
                selectedIcon: Icon(Icons.access_time_filled),
                label: Text('Turni'),
              ),
              NavigationRailDestination(
                icon: Icon(Icons.people_outline),
                selectedIcon: Icon(Icons.people),
                label: Text('Clienti'),
              ),
              NavigationRailDestination(
                icon: Icon(Icons.map_outlined),
                selectedIcon: Icon(Icons.map),
                label: Text('Mappa'),
              ),
              NavigationRailDestination(
                icon: Icon(Icons.settings_outlined),
                selectedIcon: Icon(Icons.settings),
                label: Text('Impostazioni'),
              ),
            ],
          ),
          const VerticalDivider(thickness: 1, width: 1),
          Expanded(
            child: navigationShell,
          ),
        ],
      ),
    );
  }
}
```

### 6.3 Root Widget e Tema Material 3 (`lib/ui/app.dart`)

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'router.dart';

class DelivMapApp extends ConsumerWidget {
  const DelivMapApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    const primaryColor = Color(0xFFE65100); // Arancio caldo da food delivery
    const secondaryColor = Color(0xFF00695C); // Petrolio

    return MaterialApp.router(
      title: 'DelivMap — Gestione Consegne',
      debugShowCheckedModeBanner: false,
      routerConfig: appRouter,
      theme: ThemeData(
        useMaterial3: true,
        colorScheme: ColorScheme.fromSeed(
          seedColor: primaryColor,
          primary: primaryColor,
          secondary: secondaryColor,
          brightness: Brightness.light,
        ),
      ),
      darkTheme: ThemeData(
        useMaterial3: true,
        colorScheme: ColorScheme.fromSeed(
          seedColor: primaryColor,
          primary: primaryColor,
          secondary: secondaryColor,
          brightness: Brightness.dark,
        ),
      ),
      themeMode: ThemeMode.system,
    );
  }
}
```

### 6.4 Indicatore Stato Server Locale (`lib/ui/widgets/server_indicator.dart`)

```dart
import 'dart:convert';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:http/http.dart' as http;

final serverOnlineProvider = StreamProvider.autoDispose<bool>((ref) async* {
  while (true) {
    try {
      final res = await http
          .get(Uri.parse('http://127.0.0.1:8080/health'))
          .timeout(const Duration(seconds: 1));
      if (res.statusCode == 200) {
        final body = jsonDecode(res.body);
        yield body['status'] == 'ok';
      } else {
        yield false;
      }
    } catch (_) {
      yield false;
    }
    await Future.delayed(const Duration(seconds: 5));
  }
});

class ServerIndicator extends ConsumerWidget {
  const ServerIndicator({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final status = ref.watch(serverOnlineProvider);

    return status.when(
      data: (online) => Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          Container(
            width: 10,
            height: 10,
            decoration: BoxDecoration(
              shape: BoxShape.circle,
              color: online ? Colors.green : Colors.red,
            ),
          ),
          const SizedBox(width: 8),
          Text(
            online ? 'API Online' : 'API Offline',
            style: TextStyle(
              fontSize: 12,
              color: online ? Colors.green[700] : Colors.red[700],
              fontWeight: FontWeight.w600,
            ),
          ),
        ],
      ),
      loading: () => const Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          SizedBox(
            width: 10,
            height: 10,
            child: CircularProgressIndicator(strokeWidth: 2),
          ),
          SizedBox(width: 8),
          Text('Connessione...', style: TextStyle(fontSize: 12, color: Colors.grey)),
        ],
      ),
      error: (_, __) => const Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          Icon(Icons.error, size: 12, color: Colors.red),
          SizedBox(width: 8),
          Text('Errore API', style: TextStyle(fontSize: 12, color: Colors.red)),
        ],
      ),
    );
  }
}
```

### 6.5 Placeholder delle Schermate Principali (`lib/ui/screens/`)

Creare i seguenti placeholder minimi per permettere la compilazione completa del router dichiarativo in Fase 0:

```dart
// lib/ui/screens/dashboard_screen.dart
import 'package:flutter/material.dart';
class DashboardScreen extends StatelessWidget {
  const DashboardScreen({super.key});
  @override
  Widget build(BuildContext context) => const Scaffold(body: Center(child: Text('Dashboard (Fase 1)')));
}

// lib/ui/screens/turni_screen.dart
import 'package:flutter/material.dart';
class TurniScreen extends StatelessWidget {
  const TurniScreen({super.key});
  @override
  Widget build(BuildContext context) => const Scaffold(body: Center(child: Text('Gestione Turni (Fase 1)')));
}

// lib/ui/screens/clienti_screen.dart
import 'package:flutter/material.dart';
class ClientiScreen extends StatelessWidget {
  const ClientiScreen({super.key});
  @override
  Widget build(BuildContext context) => const Scaffold(body: Center(child: Text('Anagrafica Clienti (Fase 2)')));
}

// lib/ui/screens/mappa_screen.dart
import 'package:flutter/material.dart';
class MappaScreen extends StatelessWidget {
  const MappaScreen({super.key});
  @override
  Widget build(BuildContext context) => const Scaffold(body: Center(child: Text('Mappa e Percorsi (Fase 3)')));
}

// lib/ui/screens/impostazioni_screen.dart
import 'package:flutter/material.dart';
class ImpostazioniScreen extends StatelessWidget {
  const ImpostazioniScreen({super.key});
  @override
  Widget build(BuildContext context) => const Scaffold(body: Center(child: Text('Impostazioni e Backup (Fasi 4-5)')));
}
```

---

## Criteri di Accettazione

Al completamento della Fase 0, tutti i seguenti requisiti devono essere verificati:

- [ ] **Ambiente e Struttura**:
  - [ ] Il comando `flutter doctor -v` segnala supporto desktop Windows attivo e toolchain C++ configurato.
  - [ ] L'albero `lib/` è interamente creato rispettando l'approccio di scaffolding puro, **senza** `database_provider.dart` né DAO/route CRUD premature.
- [ ] **Configurazione Dipendenze**:
  - [ ] Le dipendenze (Drift, Shelf, Riverpod, GoRouter, Window Manager, SQLite nativo, Bcrypt) sono installate e risolte senza conflitti da `flutter pub get`.
- [ ] **Database Relazionale e Drift**:
  - [ ] Tutte e 5 le tabelle (`ristoranti`, `utenti`, `clienti`, `turni`, `ordini`) sono definite.
  - [ ] L'enum `StatoOrdine` è associato alla colonna `stato` tramite `StatoOrdineConverter`, gestendo esclusivamente i 3 stati operativi reali (`in_attesa`, `in_consegna`, `consegnato`).
  - [ ] La tabella `ordini` include i campi snapshot (`nome_cliente_consegna`, `telefono_consegna`, `indirizzo_consegna`) e `orario_richiesto` come testo `HH:mm`.
  - [ ] La tabella `ristoranti` include `evidenzia_ritardi` (bool) e `tolleranza_ritardo_minuti` (int).
  - [ ] La tabella `utenti` possiede la colonna `password_hash` idonea a memorizzare stringhe bcrypt standard a 60 caratteri.
  - [ ] `dart run build_runner build` genera `lib/data/database.g.dart` con successo.
  - [ ] Il database `delivmap.sqlite` viene istanziato esclusivamente all'interno dell'Isolate del server Shelf.
- [ ] **Server Shelf e Rilevamento Istanza Singola**:
  - [ ] Il server si avvia correttamente in un Isolate di background senza bloccare il thread grafico della UI.
  - [ ] L'endpoint `GET http://127.0.0.1:8080/health` risponde con codice HTTP 200 e payload JSON con `status: "ok"`.
  - [ ] L'avvio di una seconda istanza contemporanea dell'app intercetta la `SocketException` sulla porta 8080, mostra la finestra di avviso `SingleInstanceAlertApp` e termina senza conflitti sul database.
- [ ] **Shell Desktop Flutter**:
  - [ ] L'applicazione si avvia come finestra Windows nativa (dimensioni minime 1024x720, titolo "DelivMap — Gestione Consegne").
  - [ ] La barra `NavigationRail` laterale consente di navigare tra le 5 viste (Dashboard, Turni, Clienti, Mappa, Impostazioni) preservando lo stato interno.
  - [ ] L'indicatore visivo mostra lo stato del server locale come "Online".

---

## Checklist di Testing Manuale

Seguire la seguente sequenza di test manuale per validare la Fase 0:

### Test 1: Compilazione del Database e Generazione Codice
1. Aprire il terminale PowerShell nella root del progetto: `c:\Users\39366\StudioProjects\DelivMap`.
2. Eseguire:
   ```powershell
   dart run build_runner build --delete-conflicting-outputs
   ```
3. **Esito atteso:** Il comando termina con esito positivo. Il file `lib/data/database.g.dart` risulta generato e include il mapping del converter `StatoOrdineConverter`.

### Test 2: Avvio e Verifica della Finestra Desktop
1. Avviare l'applicazione su target Windows:
   ```powershell
   flutter run -d windows
   ```
2. **Esito atteso:**
   - La finestra si apre al centro dello schermo con titolo *"DelivMap — Gestione Consegne"*.
   - Il ridimensionamento si arresta al raggiungimento della dimensione minima di 1024x720 pixel.

### Test 3: Verifica Liveness del Server Shelf Locale
1. Mantenendo l'applicazione aperta, aprire una console PowerShell separata ed eseguire:
   ```powershell
   curl.exe -i http://127.0.0.1:8080/health
   ```
2. **Esito atteso:** Risposta HTTP/1.1 200 OK con payload JSON contenente `"status": "ok"`, `"mode": "local-first"`.

### Test 4: Verifica Rilevamento Istanza Singola (Single Instance)
1. Con la prima istanza di DelivMap ancora in esecuzione, aprire un secondo terminale e lanciare una seconda istanza dell'eseguibile compilato:
   ```powershell
   .\build\windows\x64\runner\Debug\deliv_map.exe
   ```
2. **Esito atteso:** 
   - La seconda finestra rileva che la porta 8080 è già impegnata.
   - Viene visualizzata la schermata di avviso *"DelivMap è già in esecuzione"*.
   - Cliccando su *"OK, Ho Capito"* l'applicazione chiude pulitamente il processo secondario (`exit(0)`) senza interferire con l'istanza primaria né provocare corruzioni o lock al file SQLite.

### Test 5: Navigazione Shell e Reattività UI
1. All'interno della finestra di DelivMap, cliccare su ciascuna voce della barra laterale (`Turni`, `Clienti`, `Mappa`, `Impostazioni`, `Dashboard`).
2. **Esito atteso:** Transizioni istantanee tra le sezioni e indicatore verde "API Online" visibile nella sidebar.
