# Fase 4 — Autenticazione e Impostazioni

La **Fase 4** ha l'obiettivo di realizzare l'intera infrastruttura di sicurezza locale, autenticazione e configurazione applicativa di **DelivMap**. In un'architettura desktop local-first, la sicurezza garantisce che solo il titolare del ristorante o il personale autorizzato acceda ai dati sensibili dei clienti, gestisca i turni di consegna e configuri le integrazioni esterne (come la chiave Google Maps).

La durata stimata per l'implementazione completa di questa fase è di **3-4 giorni lavorativi**.

---

## 4.1 — Hashing delle Password

### Scelta della Libreria: `bcrypt` vs `crypto`

Nel contesto di un'applicazione desktop con database SQLite locale, il file del database (`delivmap.sqlite`) risiede nel filesystem della macchina host. Qualora un soggetto non autorizzato acceda direttamente al file del database, gli hash delle password non devono essere invertibili né vulnerabili ad attacchi con dizionari o tabelle arcobaleno (rainbow tables).

| Criterio | Pacchetto `crypto` (con salt manuale) | Pacchetto `bcrypt` | Verdetto per DelivMap |
|---|---|---|---|
| **Algoritmo di base** | Tipicamente SHA-256 o PBKDF2 | Blowfish crypt (Eksblowfish) | **`bcrypt`**: standard de-facto industriale per memorizzazione credenziali. |
| **Resistenza a brute-force GPU/ASIC** | Bassa (SHA-256 è progettato per essere veloce e altamente parallelizzabile) | Elevata (algoritmo memory-hard e CPU-intensive) | **`bcrypt`**: rallenta esponenzialmente i tentativi di attacco su GPU. |
| **Gestione del Salt** | Manuale: generazione casuale con `Random.secure()`, serializzazione e concatenazione manuale | Automatica: salt crittografico a 128 bit integrato nel formato modulare della stringa | **`bcrypt`**: zero rischio di implementazione errata del salting. |
| **Fattore di costo (Work Factor)** | Parametro manuale su PBKDF2 | Cost factor logaritmico integrato ($2^{\text{cost}}$ iterazioni) | **`bcrypt`**: facilmente incrementabile in futuro senza rompere la retrocompatibilità. |
| **Dipendenze esterne** | `crypto: ^3.0.3` | `bcrypt: ^1.1.3` | **`bcrypt`**: libreria leggera, pura Dart, priva di dipendenze binarie native C++. |

Per DelivMap viene selezionato il pacchetto **`bcrypt`**. Viene impostato un **cost factor pari a 12** ($2^{12} = 4096$ iterazioni). Nei benchmark su CPU desktop standard (x86_64), questo comporta un tempo di calcolo di circa 250-350 ms per operazione di hash o verifica: del tutto impercettibile per l'operatore umano al momento del login, ma estremamente costoso per un aggressore che tenti attacchi di forza bruta offline sul dump SQLite.

### Implementazione di `PasswordService`

Il servizio viene posizionato in `lib/services/password_service.dart`. Fornisce un'interfaccia ad alto livello per la generazione dell'hash e la verifica delle credenziali in fase di login o cambio password.

```dart
// lib/services/password_service.dart

import 'package:bcrypt/bcrypt.dart';

/// Servizio responsabile della sicurezza crittografica delle credenziali utente.
/// Utilizza l'algoritmo bcrypt con cost factor calibrato per ambiente desktop.
class PasswordService {
  /// Cost factor (work factor) per l'algoritmo bcrypt.
  /// Un valore di 12 garantisce $2^12 = 4.096$ round di cifratura,
  /// offrendo un equilibrio ottimale tra reattività della UI desktop (~300ms)
  /// e protezione contro attacchi offline a forza bruta.
  static const int _costFactor = 12;

  /// Lunghezza minima ammessa per una nuova password.
  static const int minPasswordLength = 8;

  /// Esegue l'hashing sicuro di una password in chiaro.
  ///
  /// Genera automaticamente un salt casuale crittograficamente sicuro a 128 bit
  /// e restituisce una stringa formattata standard (es. $2a$12$...).
  ///
  /// Lancia un'eccezione [ArgumentError] se la password non rispetta i requisiti minimi.
  static String hashPassword(String plaintext) {
    if (plaintext.trim().length < minPasswordLength) {
      throw ArgumentError(
        'La password deve contenere almeno $minPasswordLength caratteri.',
      );
    }

    final String salt = BCrypt.gensalt(logRounds: _costFactor);
    final String hashedPassword = BCrypt.hashpw(plaintext, salt);
    return hashedPassword;
  }

  /// Verifica la corrispondenza tra una password in chiaro inserita dall'utente
  /// e l'hash bcrypt memorizzato nel database SQLite.
  ///
  /// Restituisce `true` se la password è corretta, `false` altrimenti.
  /// Gestisce in modo sicuro formati di hash invalidi o corruzioni senza interrompere l'app.
  static bool verifyPassword(String plaintext, String hash) {
    if (plaintext.isEmpty || hash.isEmpty) {
      return false;
    }

    try {
      return BCrypt.checkpw(plaintext, hash);
    } catch (_) {
      // In caso di hash malformato o errore di parsing, la verifica fallisce in sicurezza
      return false;
    }
  }

  /// Valida la complessità minima della password richiesta dalle policy del locale.
  static PasswordValidationResult validatePasswordStrength(String password) {
    if (password.length < minPasswordLength) {
      return const PasswordValidationResult(
        isValid: false,
        errorMessage: 'La password deve contenere almeno 8 caratteri.',
      );
    }
    final bool hasLetter = password.contains(RegExp(r'[A-Za-z]'));
    final bool hasDigit = password.contains(RegExp(r'[0-9]'));

    if (!hasLetter || !hasDigit) {
      return const PasswordValidationResult(
        isValid: false,
        errorMessage: 'La password deve includere almeno una lettera e un numero.',
      );
    }

    return const PasswordValidationResult(isValid: true);
  }
}

/// Oggetto di risultato per la validazione client-side della password.
class PasswordValidationResult {
  final bool isValid;
  final String? errorMessage;

  const PasswordValidationResult({
    required this.isValid,
    this.errorMessage,
  });
}
```

---

## 4.2 — DAO e API per Autenticazione

L'architettura locale di DelivMap prevede che la logica applicativa risieda nel server HTTP Shelf in ascolto su `localhost`. Il server accede al database tramite DAO Drift e protegge i propri endpoint attraverso un middleware di sessione.

```
┌────────────────────────┐         ┌────────────────────────┐         ┌────────────────────────┐
│   Flutter Desktop UI   │ ──HTTP─▶│ Shelf Server: localhost│ ──DAO──▶│  Drift SQLite Database │
│ (Riverpod AuthNotifier)│ ◀──JSON─│  + SessionManager      │ ◀───────│   (utenti, ristoranti) │
└────────────────────────┘         └────────────────────────┘         └────────────────────────┘
```

### 1. Drift Table e UtentiDao

Di seguito le definizioni Drift per le tabelle `utenti` e `ristoranti` e la relativa classe DAO `UtentiDao`.

```dart
// lib/data/tables/utenti_table.dart

import 'package:drift/drift.dart';

/// Tabella SQLite per la gestione degli account operatori e gestori del locale.
class Utenti extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get nomeUtente => text().withLength(min: 3, max: 50).unique()();
  TextColumn get passwordHash => text()();
  TextColumn get ruolo => text().withDefault(const Constant('gestore'))();
  DateTimeColumn get createdAt => dateTime().withDefault(currentDateAndTime)();
}

/// Tabella SQLite per i parametri del locale (record singolo nel v1).
class Ristoranti extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get nome => text().withLength(min: 1, max: 120)();
  TextColumn get indirizzo => text()();
  RealColumn get zonaConsegnaKm => real().withDefault(const Constant(5.0))();
  TextColumn get orariAperturaJson => text().nullable()();
  TextColumn get mapsApiKey => text().nullable()();
  IntColumn get sessionTimeoutMinutes => integer().withDefault(const Constant(30))();
}
```

```dart
// lib/data/daos/utenti_dao.dart

import 'package:drift/drift.dart';
import '../database.dart';
import '../tables/utenti_table.dart';
import '../../services/password_service.dart';

part 'utenti_dao.g.dart';

@DriftAccessor(tables: [Utenti])
class UtentiDao extends DatabaseAccessor<AppDatabase> with _$UtentiDaoMixin {
  UtentiDao(AppDatabase db) : super(db);

  /// Crea un nuovo utente memorizzando l'hash bcrypt della password.
  Future<int> creaUtente({
    required String nomeUtente,
    required String plaintextPassword,
    String ruolo = 'gestore',
  }) async {
    final String hashed = PasswordService.hashPassword(plaintextPassword);
    return into(utenti).insert(
      UtentiCompanion.insert(
        nomeUtente: nomeUtente.trim(),
        passwordHash: hashed,
        ruolo: Value(ruolo),
      ),
    );
  }

  /// Recupera un utente in base al nome utente (case-insensitive a livello logico).
  Future<Utente?> getUtenteByUsername(String username) {
    return (select(utenti)..where((tbl) => tbl.nomeUtente.equals(username.trim())))
        .getSingleOrNull();
  }

  /// Recupera un utente per ID univoco.
  Future<Utente?> getUtenteById(int id) {
    return (select(utenti)..where((tbl) => tbl.id.equals(id))).getSingleOrNull();
  }

  /// Verifica le credenziali confrontando username e password in chiaro con il relativo hash.
  /// Restituisce l'entità [Utente] se valide, altrimenti `null`.
  Future<Utente?> verificaCredenziali(String username, String plaintextPassword) async {
    final Utente? utente = await getUtenteByUsername(username);
    if (utente == null) {
      return null;
    }

    final bool passwordValida = PasswordService.verifyPassword(
      plaintextPassword,
      utente.passwordHash,
    );

    return passwordValida ? utente : null;
  }

  /// Aggiorna la password di un utente esistente previo hashing.
  Future<bool> aggiornaPassword(int userId, String nuovaPassword) async {
    final String nuovoHash = PasswordService.hashPassword(nuovaPassword);
    final int righeModificate = await (update(utenti)..where((tbl) => tbl.id.equals(userId)))
        .write(UtentiCompanion(passwordHash: Value(nuovoHash)));
    return righeModificate > 0;
  }

  /// Conta il numero complessivo di utenti presenti (utile per il First-Run check).
  Future<int> contaUtenti() async {
    final countExp = utenti.id.count();
    final query = selectOnly(utenti)..addColumns([countExp]);
    final result = await query.map((row) => row.read(countExp)).getSingle();
    return result ?? 0;
  }
}
```

### 2. Gestione della Sessione Server-Side (`SessionManager`)

Nel contesto locale a singola postazione (o LAN in prospettiva v2), i token di sessione sono stringhe UUID v4 gestite in memoria con un meccanismo di scadenza per inattività e thread di pulizia periodico.

```dart
// lib/api/session_manager.dart

import 'dart:async';
import 'package:uuid/uuid.dart';

/// Informazioni sulla sessione autenticata attiva.
class SessionInfo {
  final String token;
  final int userId;
  final String username;
  final String ruolo;
  final DateTime createdAt;
  DateTime lastActivity;

  SessionInfo({
    required this.token,
    required this.userId,
    required this.username,
    required this.ruolo,
    required this.createdAt,
    required this.lastActivity,
  });

  /// Verifica se la sessione ha superato il timeout di inattività.
  bool isExpired(Duration timeout) {
    return DateTime.now().difference(lastActivity) > timeout;
  }

  Map<String, dynamic> toJson() => {
    'userId': userId,
    'username': username,
    'ruolo': ruolo,
    'createdAt': createdAt.toIso8601String(),
    'lastActivity': lastActivity.toIso8601String(),
  };
}

/// Gestore in-memory delle sessioni attive sul server Shelf locale.
class SessionManager {
  static final SessionManager _instance = SessionManager._internal();
  factory SessionManager() => _instance;
  SessionManager._internal() {
    // Schedulazione pulizia token scaduti ogni 5 minuti
    _cleanupTimer = Timer.periodic(const Duration(minutes: 5), (_) => _cleanup());
  }

  final Map<String, SessionInfo> _sessions = {};
  final Uuid _uuid = const Uuid();
  Timer? _cleanupTimer;

  /// Durata predefinita del timeout di inattività.
  Duration sessionTimeout = const Duration(minutes: 30);

  /// Crea una nuova sessione per l'utente autenticato e restituisce il token.
  String createSession({
    required int userId,
    required String username,
    required String ruolo,
  }) {
    final String token = _uuid.v4();
    final now = DateTime.now();

    _sessions[token] = SessionInfo(
      token: token,
      userId: userId,
      username: username,
      ruolo: ruolo,
      createdAt: now,
      lastActivity: now,
    );

    return token;
  }

  /// Recupera e aggiorna il timestamp di attività della sessione (se valida).
  SessionInfo? validateAndTouch(String token) {
    final SessionInfo? session = _sessions[token];
    if (session == null) {
      return null;
    }

    if (session.isExpired(sessionTimeout)) {
      _sessions.remove(token);
      return null;
    }

    session.lastActivity = DateTime.now();
    return session;
  }

  /// Recupera le info della sessione senza aggiornare l'orario di attività.
  SessionInfo? getSession(String token) {
    final session = _sessions[token];
    if (session != null && session.isExpired(sessionTimeout)) {
      _sessions.remove(token);
      return null;
    }
    return session;
  }

  /// Invalida ed elimina un token di sessione (logout).
  bool invalidateSession(String token) {
    return _sessions.remove(token) != null;
  }

  /// Rimuove tutte le sessioni scadute dalla memoria.
  void _cleanup() {
    final now = DateTime.now();
    _sessions.removeWhere((_, session) => now.difference(session.lastActivity) > sessionTimeout);
  }

  /// Rilascia le risorse allocate.
  void dispose() {
    _cleanupTimer?.cancel();
    _sessions.clear();
  }
}
```

### 3. Middleware Shelf di Protezione Endpoint (`auth_middleware.dart`)

Il middleware intercetta le richieste HTTP in entrata verso l'API locale:
1. Verifica se l'endpoint è compreso nella whitelist di percorsi pubblici (login, status setup, health check).
2. Estrae l'header `Authorization: Bearer <token>`.
3. Valida il token tramite `SessionManager`.
4. Se valido, inietta l'oggetto `SessionInfo` nel context della `shelf.Request` (`request.change(context: {'session': session})`).
5. Se non valido o assente, restituisce immediatamente una risposta HTTP `401 Unauthorized` con payload JSON esplicativo.

```dart
// lib/api/middleware/auth_middleware.dart

import 'dart:convert';
import 'package:shelf/shelf.dart';
import '../session_manager.dart';

/// Middleware Shelf per l'autenticazione tramite Bearer Token.
Middleware authMiddleware({SessionManager? sessionManager}) {
  final manager = sessionManager ?? SessionManager();

  // Percorsi pubblici accessibili senza token
  const Set<String> publicEndpoints = {
    '/api/v1/auth/login',
    '/api/v1/setup/status',
    '/api/v1/setup/init',
    '/api/v1/health',
  };

  return (Handler innerHandler) {
    return (Request request) async {
      final String path = request.url.path.startsWith('/') 
          ? request.url.path 
          : '/${request.url.path}';

      // Consenti accesso immediato agli endpoint pubblici
      if (publicEndpoints.contains(path)) {
        return innerHandler(request);
      }

      // Estrazione header Authorization
      final String? authHeader = request.headers['Authorization'] ?? 
                                 request.headers['authorization'];

      if (authHeader == null || !authHeader.startsWith('Bearer ')) {
        return Response(
          401,
          body: jsonEncode({
            'success': false,
            'error': 'Autenticazione richiesta. Token mancante o formato non valido.',
            'code': 'UNAUTHORIZED',
          }),
          headers: {'content-type': 'application/json; charset=utf-8'},
        );
      }

      final String token = authHeader.substring(7).trim();
      final SessionInfo? session = manager.validateAndTouch(token);

      if (session == null) {
        return Response(
          401,
          body: jsonEncode({
            'success': false,
            'error': 'Sessione non valida o scaduta per inattività.',
            'code': 'SESSION_EXPIRED',
          }),
          headers: {'content-type': 'application/json; charset=utf-8'},
        );
      }

      // Iniezione dei dati di sessione nel contesto della richiesta Shelf
      final updatedRequest = request.change(context: {
        'session': session,
        'userId': session.userId,
        'username': session.username,
        'ruolo': session.ruolo,
      });

      return innerHandler(updatedRequest);
    };
  };
}
```

### 4. Router Shelf per Autenticazione (`auth_routes.dart`)

```dart
// lib/api/routes/auth_routes.dart

import 'dart:convert';
import 'package:shelf/shelf.dart';
import 'package:shelf_router/shelf_router.dart';
import '../../data/daos/utenti_dao.dart';
import '../../data/database.dart';
import '../../services/password_service.dart';
import '../session_manager.dart';

class AuthRoutes {
  final UtentiDao utentiDao;
  final SessionManager sessionManager;

  AuthRoutes({
    required this.utentiDao,
    SessionManager? sessionManager,
  }) : sessionManager = sessionManager ?? SessionManager();

  Router get router {
    final router = Router();

    // POST /api/v1/auth/login
    router.post('/login', (Request request) async {
      try {
        final bodyString = await request.readAsString();
        final body = jsonDecode(bodyString) as Map<String, dynamic>;

        final username = body['username'] as String?;
        final password = body['password'] as String?;

        if (username == null || password == null || username.trim().isEmpty || password.isEmpty) {
          return Response(
            400,
            body: jsonEncode({
              'success': false,
              'error': 'Campi username e password obbligatori.',
              'code': 'INVALID_PAYLOAD',
            }),
            headers: {'content-type': 'application/json; charset=utf-8'},
          );
        }

        final Utente? utente = await utentiDao.verificaCredenziali(username, password);

        if (utente == null) {
          return Response(
            401,
            body: jsonEncode({
              'success': false,
              'error': 'Credenziali non valide. Verifica username e password.',
              'code': 'INVALID_CREDENTIALS',
            }),
            headers: {'content-type': 'application/json; charset=utf-8'},
          );
        }

        final String token = sessionManager.createSession(
          userId: utente.id,
          username: utente.nomeUtente,
          ruolo: utente.ruolo,
        );

        final DateTime expiresAt = DateTime.now().add(sessionManager.sessionTimeout);

        return Response.ok(
          jsonEncode({
            'success': true,
            'token': token,
            'user': {
              'id': utente.id,
              'username': utente.nomeUtente,
              'ruolo': utente.ruolo,
            },
            'expires_at': expiresAt.toIso8601String(),
          }),
          headers: {'content-type': 'application/json; charset=utf-8'},
        );
      } catch (e) {
        return Response.internalServerError(
          body: jsonEncode({'success': false, 'error': 'Errore interno del server durante il login.'}),
          headers: {'content-type': 'application/json; charset=utf-8'},
        );
      }
    });

    // POST /api/v1/auth/logout
    router.post('/logout', (Request request) async {
      final authHeader = request.headers['Authorization'] ?? request.headers['authorization'];
      if (authHeader != null && authHeader.startsWith('Bearer ')) {
        final token = authHeader.substring(7).trim();
        sessionManager.invalidateSession(token);
      }

      return Response.ok(
        jsonEncode({'success': true, 'message': 'Logout effettuato con successo.'}),
        headers: {'content-type': 'application/json; charset=utf-8'},
      );
    });

    // GET /api/v1/auth/me
    router.get('/me', (Request request) async {
      final session = request.context['session'] as SessionInfo?;
      if (session == null) {
        return Response(401, body: jsonEncode({'success': false, 'error': 'Non autorizzato.'}));
      }

      final Utente? utente = await utentiDao.getUtenteById(session.userId);
      if (utente == null) {
        return Response(404, body: jsonEncode({'success': false, 'error': 'Utente non trovato.'}));
      }

      return Response.ok(
        jsonEncode({
          'success': true,
          'user': {
            'id': utente.id,
            'username': utente.nomeUtente,
            'ruolo': utente.ruolo,
            'created_at': utente.createdAt.toIso8601String(),
          },
        }),
        headers: {'content-type': 'application/json; charset=utf-8'},
      );
    });

    // PUT /api/v1/auth/password
    router.put('/password', (Request request) async {
      final session = request.context['session'] as SessionInfo?;
      if (session == null) {
        return Response(401, body: jsonEncode({'success': false, 'error': 'Non autorizzato.'}));
      }

      try {
        final body = jsonDecode(await request.readAsString()) as Map<String, dynamic>;
        final currentPassword = body['current_password'] as String?;
        final newPassword = body['new_password'] as String?;

        if (currentPassword == null || newPassword == null) {
          return Response(
            400,
            body: jsonEncode({
              'success': false,
              'error': 'Specificare sia la password attuale che la nuova password.',
            }),
            headers: {'content-type': 'application/json; charset=utf-8'},
          );
        }

        // Verifica validità complessità nuova password
        final validation = PasswordService.validatePasswordStrength(newPassword);
        if (!validation.isValid) {
          return Response(
            400,
            body: jsonEncode({'success': false, 'error': validation.errorMessage}),
            headers: {'content-type': 'application/json; charset=utf-8'},
          );
        }

        // Verifica password attuale
        final utente = await utentiDao.getUtenteById(session.userId);
        if (utente == null || !PasswordService.verifyPassword(currentPassword, utente.passwordHash)) {
          return Response(
            403,
            body: jsonEncode({
              'success': false,
              'error': 'La password attuale inserita non è corretta.',
              'code': 'INVALID_CURRENT_PASSWORD',
            }),
            headers: {'content-type': 'application/json; charset=utf-8'},
          );
        }

        await utentiDao.aggiornaPassword(session.userId, newPassword);

        return Response.ok(
          jsonEncode({'success': true, 'message': 'Password aggiornata con successo.'}),
          headers: {'content-type': 'application/json; charset=utf-8'},
        );
      } catch (e) {
        return Response.internalServerError(
          body: jsonEncode({'success': false, 'error': 'Errore nell\'aggiornamento password.'}),
          headers: {'content-type': 'application/json; charset=utf-8'},
        );
      }
    });

    return router;
  }
}
```

### 5. Contratti API Endpoints

| Metodo | Endpoint | Autenticazione | Body Richiesta | Risposta di Successo (200 OK) | Codici Errore |
|---|---|---|---|---|---|
| `POST` | `/api/v1/auth/login` | No (Pubblico) | `{"username": "...", "password": "..."}` | `{"success": true, "token": "UUID", "user": {...}, "expires_at": "..."}` | 400 (Bad Request), 401 (Credenziali errate) |
| `POST` | `/api/v1/auth/logout` | Sì (`Bearer <token>`) | Vuoto | `{"success": true, "message": "Logout effettuato"}` | 401 (Token non valido) |
| `GET` | `/api/v1/auth/me` | Sì (`Bearer <token>`) | Vuoto | `{"success": true, "user": {"id": 1, "username": "...", "ruolo": "gestore"}}` | 401, 404 |
| `PUT` | `/api/v1/auth/password` | Sì (`Bearer <token>`) | `{"current_password": "...", "new_password": "..."}` | `{"success": true, "message": "Password aggiornata"}` | 400 (Debole/Mancante), 403 (Password attuale errata) |
| `GET` | `/api/v1/setup/status` | No (Pubblico) | Vuoto | `{"first_run": true/false, "configured": true/false}` | 500 |
| `POST` | `/api/v1/setup/init` | No (Pubblico) | `{"nome": "...", "indirizzo": "...", "username": "...", "password": "...", "maps_api_key": "..."}` | `{"success": true, "token": "...", "user": {...}}` | 400, 409 (Già configurato) |

---

## 4.3 — UI — Schermata di Login

La schermata di login è la prima interfaccia presentata all'avvio su un sistema già inizializzato. È concepita per l'ambiente desktop di cassa/banco, supporta la digitazione rapida da tastiera (tasto `Enter` per inviare) e implementa un timer di inattività globale che esegue l'auto-logout a tutela della sicurezza.

### 1. Gestione dello Stato con Riverpod (`auth_provider.dart`)

```dart
// lib/ui/providers/auth_provider.dart

import 'dart:async';
import 'dart:convert';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:http/http.dart' as http;

/// Modello dell'utente autenticato nella sessione client.
class AuthUser {
  final int id;
  final String username;
  final String ruolo;

  const AuthUser({
    required this.id,
    required this.username,
    required this.ruolo,
  });

  factory AuthUser.fromJson(Map<String, dynamic> json) {
    return AuthUser(
      id: json['id'] as int,
      username: json['username'] as String,
      ruolo: json['ruolo'] as String,
    );
  }
}

/// Stati possibili dell'autenticazione nell'interfaccia utente.
enum AuthStatus {
  initial,
  checkingFirstRun,
  firstRunRequired,
  unauthenticated,
  authenticating,
  authenticated,
  error,
}

class AuthState {
  final AuthStatus status;
  final AuthUser? user;
  final String? token;
  final String? errorMessage;

  const AuthState({
    required this.status,
    this.user,
    this.token,
    this.errorMessage,
  });

  factory AuthState.initial() => const AuthState(status: AuthStatus.initial);

  AuthState copyWith({
    AuthStatus? status,
    AuthUser? user,
    String? token,
    String? errorMessage,
  }) {
    return AuthState(
      status: status ?? this.status,
      user: user ?? this.user,
      token: token ?? this.token,
      errorMessage: errorMessage ?? this.errorMessage,
    );
  }
}

class AuthNotifier extends StateNotifier<AuthState> {
  final String apiBaseUrl;
  Timer? _inactivityTimer;
  Duration _timeoutDuration = const Duration(minutes: 30);

  AuthNotifier({this.apiBaseUrl = 'http://localhost:8080'}) : super(AuthState.initial()) {
    checkAppInitialization();
  }

  void setTimeoutDuration(Duration duration) {
    _timeoutDuration = duration;
    _resetInactivityTimer();
  }

  /// Verifica lo stato iniziale del database locale (se vuoto -> First Run).
  Future<void> checkAppInitialization() async {
    state = state.copyWith(status: AuthStatus.checkingFirstRun);
    try {
      final response = await http.get(Uri.parse('$apiBaseUrl/api/v1/setup/status'));
      if (response.statusCode == 200) {
        final data = jsonDecode(response.body) as Map<String, dynamic>;
        final bool isFirstRun = data['first_run'] as bool? ?? false;
        if (isFirstRun) {
          state = state.copyWith(status: AuthStatus.firstRunRequired);
          return;
        }
      }
      state = state.copyWith(status: AuthStatus.unauthenticated);
    } catch (_) {
      // Se il server locale è momentaneamente in avvio o non risponde
      state = state.copyWith(status: AuthStatus.unauthenticated);
    }
  }

  /// Esegue il login tramite API Shelf locale.
  Future<bool> login(String username, String password) async {
    state = state.copyWith(status: AuthStatus.authenticating, errorMessage: null);

    try {
      final response = await http.post(
        Uri.parse('$apiBaseUrl/api/v1/auth/login'),
        headers: {'Content-Type': 'application/json'},
        body: jsonEncode({'username': username.trim(), 'password': password}),
      );

      final data = jsonDecode(response.body) as Map<String, dynamic>;

      if (response.statusCode == 200 && data['success'] == true) {
        final token = data['token'] as String;
        final user = AuthUser.fromJson(data['user'] as Map<String, dynamic>);

        state = AuthState(
          status: AuthStatus.authenticated,
          user: user,
          token: token,
        );

        _resetInactivityTimer();
        return true;
      } else {
        state = state.copyWith(
          status: AuthStatus.error,
          errorMessage: data['error'] as String? ?? 'Credenziali non valide.',
        );
        return false;
      }
    } catch (e) {
      state = state.copyWith(
        status: AuthStatus.error,
        errorMessage: 'Impossibile connettersi al server locale DelivMap.',
      );
      return false;
    }
  }

  /// Esegue il logout ed invalida la sessione.
  Future<void> logout() async {
    _inactivityTimer?.cancel();
    if (state.token != null) {
      try {
        await http.post(
          Uri.parse('$apiBaseUrl/api/v1/auth/logout'),
          headers: {'Authorization': 'Bearer ${state.token}'},
        );
      } catch (_) {}
    }
    state = const AuthState(status: AuthStatus.unauthenticated);
  }

  /// Segnala attività dell'utente per resettare il conto alla rovescia del timeout.
  void recordUserActivity() {
    if (state.status == AuthStatus.authenticated) {
      _resetInactivityTimer();
    }
  }

  void _resetInactivityTimer() {
    _inactivityTimer?.cancel();
    if (_timeoutDuration == Duration.zero) return; // Timeout disattivato

    _inactivityTimer = Timer(_timeoutDuration, () {
      logout();
    });
  }

  @override
  void dispose() {
    _inactivityTimer?.cancel();
    super.dispose();
  }
}

final authNotifierProvider = StateNotifierProvider<AuthNotifier, AuthState>((ref) {
  return AuthNotifier();
});
```

### 2. Wrapper di Tracciamento Inattività (`SessionTimeoutWrapper`)

Questo widget racchiude l'albero della UI desktop, catturando eventi di puntatore (mouse) e tastiera senza interferire con la propagazione dei gesti.

```dart
// lib/ui/widgets/session_timeout_wrapper.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../providers/auth_provider.dart';

class SessionTimeoutWrapper extends ConsumerWidget {
  final Widget child;

  const SessionTimeoutWrapper({Key? key, required this.child}) : super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Listener(
      behavior: HitTestBehavior.translucent,
      onPointerDown: (_) => ref.read(authNotifierProvider.notifier).recordUserActivity(),
      onPointerMove: (_) => ref.read(authNotifierProvider.notifier).recordUserActivity(),
      child: Focus(
        onKeyEvent: (node, event) {
          ref.read(authNotifierProvider.notifier).recordUserActivity();
          return KeyEventResult.ignored;
        },
        child: child,
      ),
    );
  }
}
```

### 3. Schermata di Login Completa (`LoginScreen`)

```dart
// lib/ui/screens/login_screen.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../providers/auth_provider.dart';

class LoginScreen extends ConsumerStatefulWidget {
  const LoginScreen({Key? key}) : super(key: key);

  @override
  ConsumerState<LoginScreen> createState() => _LoginScreenState();
}

class _LoginScreenState extends ConsumerState<LoginScreen> {
  final _formKey = GlobalKey<FormState>();
  final _usernameController = TextEditingController();
  final _passwordController = TextEditingController();
  bool _obscurePassword = true;

  @override
  void dispose() {
    _usernameController.dispose();
    _passwordController.dispose();
    super.dispose();
  }

  Future<void> _handleLogin() async {
    if (!_formKey.currentState!.validate()) return;

    final success = await ref.read(authNotifierProvider.notifier).login(
      _usernameController.text,
      _passwordController.text,
    );

    if (!success && mounted) {
      final error = ref.read(authNotifierProvider).errorMessage ?? 'Accesso negato';
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(
          content: Row(
            children: [
              const Icon(Icons.error_outline, color: Colors.white),
              const SizedBox(width: 12),
              Expanded(child: Text(error)),
            ],
          ),
          backgroundColor: Theme.of(context).colorScheme.error,
          behavior: SnackBarBehavior.floating,
        ),
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    final authState = ref.watch(authNotifierProvider);
    final theme = Theme.of(context);
    final isAuthenticating = authState.status == AuthStatus.authenticating;

    return Scaffold(
      backgroundColor: theme.colorScheme.surfaceVariant.withOpacity(0.3),
      body: Center(
        child: SingleChildScrollView(
          child: Container(
            width: 440,
            padding: const EdgeInsets.all(36),
            decoration: BoxDecoration(
              color: theme.colorScheme.surface,
              borderRadius: BorderRadius.circular(16),
              boxShadow: [
                BoxShadow(
                  color: Colors.black.withOpacity(0.08),
                  blurRadius: 24,
                  offset: const Offset(0, 8),
                ),
              ],
            ),
            child: Form(
              key: _formKey,
              child: Column(
                mainAxisSize: MainAxisSize.min,
                crossAxisAlignment: CrossAxisAlignment.stretch,
                children: [
                  Center(
                    child: Container(
                      padding: const EdgeInsets.all(16),
                      decoration: BoxDecoration(
                        color: theme.colorScheme.primaryContainer,
                        shape: BoxShape.circle,
                      ),
                      child: Icon(
                        Icons.delivery_dining,
                        size: 48,
                        color: theme.colorScheme.primary,
                      ),
                    ),
                  ),
                  const SizedBox(height: 20),
                  Text(
                    'DelivMap',
                    textAlign: TextAlign.center,
                    style: theme.textTheme.headlineMedium?.copyWith(
                      fontWeight: FontWeight.bold,
                      color: theme.colorScheme.onSurface,
                    ),
                  ),
                  const SizedBox(height: 6),
                  Text(
                    'Gestione consegne a domicilio',
                    textAlign: TextAlign.center,
                    style: theme.textTheme.bodyMedium?.copyWith(
                      color: theme.colorScheme.onSurfaceVariant,
                    ),
                  ),
                  const SizedBox(height: 32),
                  TextFormField(
                    controller: _usernameController,
                    enabled: !isAuthenticating,
                    decoration: const InputDecoration(
                      labelText: 'Nome Utente',
                      prefixIcon: Icon(Icons.person_outline),
                      border: OutlineInputBorder(),
                    ),
                    validator: (val) {
                      if (val == null || val.trim().isEmpty) {
                        return 'Inserisci il nome utente';
                      }
                      return null;
                    },
                    textInputAction: TextInputAction.next,
                  ),
                  const SizedBox(height: 20),
                  TextFormField(
                    controller: _passwordController,
                    obscureText: _obscurePassword,
                    enabled: !isAuthenticating,
                    decoration: InputDecoration(
                      labelText: 'Password',
                      prefixIcon: const Icon(Icons.lock_outline),
                      border: const OutlineInputBorder(),
                      suffixIcon: IconButton(
                        icon: Icon(
                          _obscurePassword ? Icons.visibility_outlined : Icons.visibility_off_outlined,
                        ),
                        onPressed: () => setState(() => _obscurePassword = !_obscurePassword),
                      ),
                    ),
                    validator: (val) {
                      if (val == null || val.isEmpty) {
                        return 'Inserisci la password';
                      }
                      return null;
                    },
                    onFieldSubmitted: (_) => _handleLogin(),
                  ),
                  const SizedBox(height: 28),
                  SizedBox(
                    height: 48,
                    child: FilledButton(
                      onPressed: isAuthenticating ? null : _handleLogin,
                      child: isAuthenticating
                          ? const SizedBox(
                              width: 24,
                              height: 24,
                              child: CircularProgressIndicator(
                                strokeWidth: 2.5,
                                color: Colors.white,
                              ),
                            )
                          : const Text(
                              'Accedi',
                              style: TextStyle(fontSize: 16, fontWeight: FontWeight.w600),
                            ),
                    ),
                  ),
                ],
              ),
            ),
          ),
        ),
      ),
    );
  }
}
```

---

## 4.4 — UI — First-Run Wizard (Setup Iniziale)

Al primo avvio assoluto dell'applicazione, qualora la tabella `ristoranti` risulti priva di record o non vi siano utenti configurati, l'app instrada l'utente verso la procedura guidata di inizializzazione.

Il wizard viene realizzato con il widget `Stepper` di Flutter in versione desktop a 4 step:
1. **Benvenuto e Sede Locale**: Raccolta del nome e indirizzo civico del ristorante (punto di partenza delle consegne).
2. **Account Gestore**: Creazione delle credenziali amministrative (username, password e conferma password con controllo di robustezza).
3. **Google Maps API (BYOK - Opzionale)**: Inserimento della chiave API del cliente con possibilità di test immediato o posticipo.
4. **Riepilogo e Attivazione**: Schermata di controllo ed esecuzione atomica del setup.

### Codice Completo del First-Run Wizard (`FirstRunWizardScreen`)

```dart
// lib/ui/screens/first_run_wizard_screen.dart

import 'dart:convert';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:http/http.dart' as http;
import '../providers/auth_provider.dart';
import '../../services/password_service.dart';

class FirstRunWizardScreen extends ConsumerStatefulWidget {
  const FirstRunWizardScreen({Key? key}) : super(key: key);

  @override
  ConsumerState<FirstRunWizardScreen> createState() => _FirstRunWizardScreenState();
}

class _FirstRunWizardScreenState extends ConsumerState<FirstRunWizardScreen> {
  int _currentStep = 0;
  bool _isSubmitting = false;
  bool _testingApiKey = false;
  String? _apiKeyTestResult;

  // Controller Form
  final _nomeRistoranteController = TextEditingController();
  final _indirizzoRistoranteController = TextEditingController();
  final _usernameController = TextEditingController();
  final _passwordController = TextEditingController();
  final _confermaPasswordController = TextEditingController();
  final _mapsApiKeyController = TextEditingController();

  final _formKeyStep1 = GlobalKey<FormState>();
  final _formKeyStep2 = GlobalKey<FormState>();

  @override
  void dispose() {
    _nomeRistoranteController.dispose();
    _indirizzoRistoranteController.dispose();
    _usernameController.dispose();
    _passwordController.dispose();
    _confermaPasswordController.dispose();
    _mapsApiKeyController.dispose();
    super.dispose();
  }

  Future<void> _testGoogleMapsKey() async {
    final key = _mapsApiKeyController.text.trim();
    if (key.isEmpty) return;

    setState(() {
      _testingApiKey = true;
      _apiKeyTestResult = null;
    });

    try {
      final res = await http.get(Uri.parse(
        'https://maps.googleapis.com/maps/api/geocode/json?address=Roma&key=$key',
      ));
      final data = jsonDecode(res.body) as Map<String, dynamic>;
      final status = data['status'] as String?;

      setState(() {
        if (status == 'OK' || status == 'ZERO_RESULTS') {
          _apiKeyTestResult = 'SUCCESS: Chiave API valida!';
        } else {
          _apiKeyTestResult = 'ERRORE: ${data['error_message'] ?? status ?? 'Chiave non valida'}';
        }
      });
    } catch (e) {
      setState(() {
        _apiKeyTestResult = 'ERRORE: Impossibile contattare i server Google ($e)';
      });
    } finally {
      setState(() => _testingApiKey = false);
    }
  }

  Future<void> _completeSetup() async {
    setState(() => _isSubmitting = true);

    final payload = {
      'nome_ristorante': _nomeRistoranteController.text.trim(),
      'indirizzo_ristorante': _indirizzoRistoranteController.text.trim(),
      'username': _usernameController.text.trim(),
      'password': _passwordController.text,
      'maps_api_key': _mapsApiKeyController.text.trim().isNotEmpty
          ? _mapsApiKeyController.text.trim()
          : null,
    };

    try {
      final res = await http.post(
        Uri.parse('http://localhost:8080/api/v1/setup/init'),
        headers: {'Content-Type': 'application/json'},
        body: jsonEncode(payload),
      );

      final data = jsonDecode(res.body) as Map<String, dynamic>;

      if (res.statusCode == 200 && data['success'] == true) {
        // Ricarica lo stato auth o esegui il login diretto con le credenziali appena create
        await ref.read(authNotifierProvider.notifier).login(
          _usernameController.text.trim(),
          _passwordController.text,
        );
      } else {
        if (mounted) {
          ScaffoldMessenger.of(context).showSnackBar(
            SnackBar(
              content: Text(data['error'] ?? 'Errore durante la configurazione iniziale.'),
              backgroundColor: Colors.red,
            ),
          );
        }
      }
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(
            content: Text('Errore di connessione al server locale durante il setup.'),
            backgroundColor: Colors.red,
          ),
        );
      }
    } finally {
      if (mounted) setState(() => _isSubmitting = false);
    }
  }

  void _onStepContinue() {
    if (_currentStep == 0) {
      if (!_formKeyStep1.currentState!.validate()) return;
    } else if (_currentStep == 1) {
      if (!_formKeyStep2.currentState!.validate()) return;
    } else if (_currentStep == 3) {
      _completeSetup();
      return;
    }
    setState(() => _currentStep += 1);
  }

  void _onStepCancel() {
    if (_currentStep > 0) {
      setState(() => _currentStep -= 1);
    }
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Configurazione Iniziale DelivMap'),
        elevation: 1,
      ),
      body: Center(
        child: Container(
          constraints: const BoxConstraints(maxWidth: 860),
          padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 20),
          child: Stepper(
            type: StepperType.horizontal,
            currentStep: _currentStep,
            onStepContinue: _isSubmitting ? null : _onStepContinue,
            onStepCancel: _isSubmitting ? null : _onStepCancel,
            controlsBuilder: (context, details) {
              final isLast = _currentStep == 3;
              return Padding(
                padding: const EdgeInsets.only(top: 24),
                child: Row(
                  children: [
                    FilledButton(
                      onPressed: details.onStepContinue,
                      child: _isSubmitting
                          ? const SizedBox(
                              width: 18,
                              height: 18,
                              child: CircularProgressIndicator(strokeWidth: 2, color: Colors.white),
                            )
                          : Text(isLast ? 'Completa e Avvia' : 'Avanti'),
                    ),
                    if (_currentStep > 0) ...[
                      const SizedBox(width: 12),
                      OutlinedButton(
                        onPressed: details.onStepCancel,
                        child: const Text('Indietro'),
                      ),
                    ],
                  ],
                ),
              );
            },
            steps: [
              // STEP 1: Sede Ristorante
              Step(
                title: const Text('Locale'),
                isActive: _currentStep >= 0,
                state: _currentStep > 0 ? StepState.complete : StepState.indexed,
                content: Form(
                  key: _formKeyStep1,
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Text(
                        'Benvenuto in DelivMap',
                        style: theme.textTheme.titleLarge?.copyWith(fontWeight: FontWeight.bold),
                      ),
                      const SizedBox(height: 8),
                      Text(
                        'Inserisci le informazioni sul tuo ristorante per configurare il punto di partenza delle consegne.',
                        style: theme.textTheme.bodyMedium?.copyWith(color: theme.colorScheme.onSurfaceVariant),
                      ),
                      const SizedBox(height: 20),
                      TextFormField(
                        controller: _nomeRistoranteController,
                        decoration: const InputDecoration(
                          labelText: 'Nome Ristorante / Pizzeria *',
                          prefixIcon: Icon(Icons.storefront),
                          border: OutlineInputBorder(),
                        ),
                        validator: (val) => val == null || val.trim().isEmpty ? 'Inserisci il nome del locale' : null,
                      ),
                      const SizedBox(height: 16),
                      TextFormField(
                        controller: _indirizzoRistoranteController,
                        decoration: const InputDecoration(
                          labelText: 'Indirizzo completo (Via, Civico, Città, CAP) *',
                          prefixIcon: Icon(Icons.location_on_outlined),
                          border: OutlineInputBorder(),
                          helperText: 'Questo indirizzo sarà l\'origine per il calcolo dei percorsi dei turni.',
                        ),
                        validator: (val) => val == null || val.trim().isEmpty ? 'Inserisci l\'indirizzo della sede' : null,
                      ),
                    ],
                  ),
                ),
              ),

              // STEP 2: Account Gestore
              Step(
                title: const Text('Gestore'),
                isActive: _currentStep >= 1,
                state: _currentStep > 1 ? StepState.complete : StepState.indexed,
                content: Form(
                  key: _formKeyStep2,
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Text(
                        'Crea l\'Account Amministratore',
                        style: theme.textTheme.titleLarge?.copyWith(fontWeight: FontWeight.bold),
                      ),
                      const SizedBox(height: 8),
                      Text(
                        'Questo account avrà pieno accesso alla gestione ordini, turni e impostazioni dell\'applicazione.',
                        style: theme.textTheme.bodyMedium?.copyWith(color: theme.colorScheme.onSurfaceVariant),
                      ),
                      const SizedBox(height: 20),
                      TextFormField(
                        controller: _usernameController,
                        decoration: const InputDecoration(
                          labelText: 'Nome Utente (min. 3 caratteri) *',
                          prefixIcon: Icon(Icons.person_outline),
                          border: OutlineInputBorder(),
                        ),
                        validator: (val) {
                          if (val == null || val.trim().length < 3) {
                            return 'Il nome utente deve avere almeno 3 caratteri';
                          }
                          return null;
                        },
                      ),
                      const SizedBox(height: 16),
                      TextFormField(
                        controller: _passwordController,
                        obscureText: true,
                        decoration: const InputDecoration(
                          labelText: 'Password *',
                          prefixIcon: Icon(Icons.lock_outline),
                          border: OutlineInputBorder(),
                          helperText: 'Almeno 8 caratteri, includendo una lettera e un numero.',
                        ),
                        validator: (val) {
                          final result = PasswordService.validatePasswordStrength(val ?? '');
                          return result.isValid ? null : result.errorMessage;
                        },
                      ),
                      const SizedBox(height: 16),
                      TextFormField(
                        controller: _confermaPasswordController,
                        obscureText: true,
                        decoration: const InputDecoration(
                          labelText: 'Conferma Password *',
                          prefixIcon: Icon(Icons.lock_reset),
                          border: OutlineInputBorder(),
                        ),
                        validator: (val) {
                          if (val != _passwordController.text) {
                            return 'Le password inserite non coincidono';
                          }
                          return null;
                        },
                      ),
                    ],
                  ),
                ),
              ),

              // STEP 3: Google Maps API Key
              Step(
                title: const Text('Mappe'),
                isActive: _currentStep >= 2,
                state: _currentStep > 2 ? StepState.complete : StepState.indexed,
                content: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      'Configura Google Maps (Opzionale)',
                      style: theme.textTheme.titleLarge?.copyWith(fontWeight: FontWeight.bold),
                    ),
                    const SizedBox(height: 8),
                    Text(
                      'DelivMap adotta il modello BYOK (Bring Your Own Key): i costi di geolocalizzazione restano legati al tuo account Google Cloud (sfruttando il credito gratuito mensile di 200\$). Puoi inserire la chiave ora o configurarla in seguito dalle Impostazioni.',
                      style: theme.textTheme.bodyMedium?.copyWith(color: theme.colorScheme.onSurfaceVariant),
                    ),
                    const SizedBox(height: 20),
                    Row(
                      crossAxisAlignment: CrossAxisAlignment.start,
                      children: [
                        Expanded(
                          child: TextField(
                            controller: _mapsApiKeyController,
                            decoration: const InputDecoration(
                              labelText: 'Google Maps API Key',
                              hintText: 'AIzaSy...',
                              prefixIcon: Icon(Icons.map_outlined),
                              border: OutlineInputBorder(),
                            ),
                          ),
                        ),
                        const SizedBox(width: 12),
                        Padding(
                          padding: const EdgeInsets.only(top: 4),
                          child: OutlinedButton.icon(
                            onPressed: _testingApiKey ? null : _testGoogleMapsKey,
                            icon: _testingApiKey
                                ? const SizedBox(width: 16, height: 16, child: CircularProgressIndicator(strokeWidth: 2))
                                : const Icon(Icons.check_circle_outline),
                            label: const Text('Verifica'),
                          ),
                        ),
                      ],
                    ),
                    if (_apiKeyTestResult != null) ...[
                      const SizedBox(height: 12),
                      Text(
                        _apiKeyTestResult!,
                        style: TextStyle(
                          color: _apiKeyTestResult!.startsWith('SUCCESS') ? Colors.green[700] : Colors.red[700],
                          fontWeight: FontWeight.w600,
                        ),
                      ),
                    ],
                  ],
                ),
              ),

              // STEP 4: Riepilogo
              Step(
                title: const Text('Conferma'),
                isActive: _currentStep >= 3,
                state: StepState.indexed,
                content: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      'Riepilogo Configurazione',
                      style: theme.textTheme.titleLarge?.copyWith(fontWeight: FontWeight.bold),
                    ),
                    const SizedBox(height: 16),
                    Card(
                      elevation: 0,
                      color: theme.colorScheme.surfaceVariant.withOpacity(0.4),
                      child: Padding(
                        padding: const EdgeInsets.all(16),
                        child: Column(
                          children: [
                            ListTile(
                              leading: const Icon(Icons.storefront),
                              title: const Text('Ristorante'),
                              subtitle: Text(_nomeRistoranteController.text),
                            ),
                            const Divider(),
                            ListTile(
                              leading: const Icon(Icons.place),
                              title: const Text('Indirizzo'),
                              subtitle: Text(_indirizzoRistoranteController.text),
                            ),
                            const Divider(),
                            ListTile(
                              leading: const Icon(Icons.person),
                              title: const Text('Amministratore'),
                              subtitle: Text(_usernameController.text),
                            ),
                            const Divider(),
                            ListTile(
                              leading: const Icon(Icons.key),
                              title: const Text('Google Maps API Key'),
                              subtitle: Text(
                                _mapsApiKeyController.text.trim().isNotEmpty
                                    ? 'Configurata (${_mapsApiKeyController.text.substring(0, 6)}...)'
                                    : 'Non inserita (configurabile in seguito)',
                              ),
                            ),
                          ],
                        ),
                      ),
                    ),
                  ],
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

## 4.5 — UI — Pagina Impostazioni Completa

La pagina delle impostazioni centralizza tutti i parametri del locale, i vincoli di consegna, le credenziali e le preferenze dell'applicazione. È suddivisa in schede tematiche tramite `NavigationRail` su desktop.

### Struttura delle Sezioni

1. **Ristorante**: Modifica anagrafica (nome del locale, indirizzo sede operativa).
2. **Zona di Consegna**: Raggio limite in chilometri (da 1 a 30 km con `Slider` e validazione).
3. **Orari di Apertura**: Orari per ciascun giorno della settimana (Lunedì - Domenica) con toggle giorno aperto/chiuso e selettore orario (`TimeOfDay`).
4. **Google Maps API**: Gestione chiave BYOK, mascheramento del testo e test di connettività in tempo reale.
5. **Account**: Modifica password del gestore (verifica vecchia password e validazione nuova).
6. **Applicazione**: Scelta del tema (Chiaro / Scuro / Sistema) e durata timeout inattività sessione (15m, 30m, 60m, Mai).

### Codice Completo della Pagina Impostazioni (`SettingsScreen`)

```dart
// lib/ui/screens/settings_screen.dart

import 'dart:convert';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:http/http.dart' as http;
import '../providers/auth_provider.dart';

class SettingsScreen extends ConsumerStatefulWidget {
  const SettingsScreen({Key? key}) : super(key: key);

  @override
  ConsumerState<SettingsScreen> createState() => _SettingsScreenState();
}

class _SettingsScreenState extends ConsumerState<SettingsScreen> {
  int _selectedSection = 0;
  bool _isLoading = true;
  bool _isSaving = false;

  // Dati Ristorante
  final _nomeController = TextEditingController();
  final _indirizzoController = TextEditingController();

  // Zona Consegna
  double _zonaConsegnaKm = 5.0;

  // Orari settimanali: Map con giorno -> { 'aperto': bool, 'apertura': '18:00', 'chiusura': '23:00' }
  final List<String> _giorniSettimana = [
    'Lunedì', 'Martedì', 'Mercoledì', 'Giovedì', 'Venerdì', 'Sabato', 'Domenica'
  ];
  Map<String, Map<String, dynamic>> _orari = {};

  // Google Maps
  final _mapsApiKeyController = TextEditingController();
  bool _obscureApiKey = true;
  String? _apiKeyTestStatus;

  // Cambio Password
  final _currentPasswordController = TextEditingController();
  final _newPasswordController = TextEditingController();
  final _confirmPasswordController = TextEditingController();

  // Applicazione
  ThemeMode _selectedThemeMode = ThemeMode.system;
  int _sessionTimeoutMinutes = 30;

  @override
  void initState() {
    super.initState();
    _initDefaultOrari();
    _loadSettings();
  }

  void _initDefaultOrari() {
    _orari = {
      for (var giorno in _giorniSettimana)
        giorno: {
          'aperto': giorno != 'Lunedì',
          'apertura': '18:30',
          'chiusura': '23:00',
        }
    };
  }

  Future<void> _loadSettings() async {
    final token = ref.read(authNotifierProvider).token;
    setState(() => _isLoading = true);

    try {
      final res = await http.get(
        Uri.parse('http://localhost:8080/api/v1/settings/all'),
        headers: {'Authorization': 'Bearer $token'},
      );

      if (res.statusCode == 200) {
        final data = jsonDecode(res.body)['data'] as Map<String, dynamic>;
        _nomeController.text = data['nome'] ?? '';
        _indirizzoController.text = data['indirizzo'] ?? '';
        _zonaConsegnaKm = (data['zona_consegna_km'] as num?)?.toDouble() ?? 5.0;
        _mapsApiKeyController.text = data['maps_api_key'] ?? '';
        _sessionTimeoutMinutes = data['session_timeout_minutes'] ?? 30;

        if (data['orari_json'] != null) {
          final loadedOrari = jsonDecode(data['orari_json']) as Map<String, dynamic>;
          _orari = loadedOrari.map((k, v) => MapEntry(k, Map<String, dynamic>.from(v)));
        }
      }
    } catch (_) {
      // In caso di errore utilizza i default inizializzati
    } finally {
      if (mounted) setState(() => _isLoading = false);
    }
  }

  Future<void> _saveAllSettings() async {
    setState(() => _isSaving = true);
    final token = ref.read(authNotifierProvider).token;

    final payload = {
      'nome': _nomeController.text.trim(),
      'indirizzo': _indirizzoController.text.trim(),
      'zona_consegna_km': _zonaConsegnaKm,
      'maps_api_key': _mapsApiKeyController.text.trim().isEmpty ? null : _mapsApiKeyController.text.trim(),
      'orari_json': jsonEncode(_orari),
      'session_timeout_minutes': _sessionTimeoutMinutes,
    };

    try {
      final res = await http.put(
        Uri.parse('http://localhost:8080/api/v1/settings/all'),
        headers: {
          'Authorization': 'Bearer $token',
          'Content-Type': 'application/json',
        },
        body: jsonEncode(payload),
      );

      if (res.statusCode == 200 && mounted) {
        // Aggiorna timeout nel provider
        ref.read(authNotifierProvider.notifier).setTimeoutDuration(
          Duration(minutes: _sessionTimeoutMinutes),
        );

        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(
            content: Text('Impostazioni salvate con successo!'),
            backgroundColor: Colors.green,
            behavior: SnackBarBehavior.floating,
          ),
        );
      } else {
        throw Exception('Errore di salvataggio');
      }
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(
            content: Text('Errore durante il salvataggio delle impostazioni.'),
            backgroundColor: Colors.red,
            behavior: SnackBarBehavior.floating,
          ),
        );
      }
    } finally {
      if (mounted) setState(() => _isSaving = false);
    }
  }

  Future<void> _handleChangePassword() async {
    final token = ref.read(authNotifierProvider).token;
    if (_newPasswordController.text != _confirmPasswordController.text) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Le nuove password non coincidono.')),
      );
      return;
    }

    try {
      final res = await http.put(
        Uri.parse('http://localhost:8080/api/v1/auth/password'),
        headers: {
          'Authorization': 'Bearer $token',
          'Content-Type': 'application/json',
        },
        body: jsonEncode({
          'current_password': _currentPasswordController.text,
          'new_password': _newPasswordController.text,
        }),
      );

      final data = jsonDecode(res.body);
      if (res.statusCode == 200 && mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Password cambiata con successo!'), backgroundColor: Colors.green),
        );
        _currentPasswordController.clear();
        _newPasswordController.clear();
        _confirmPasswordController.clear();
      } else if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(data['error'] ?? 'Errore cambio password'), backgroundColor: Colors.red),
        );
      }
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Errore di comunicazione col server.'), backgroundColor: Colors.red),
        );
      }
    }
  }

  Future<void> _pickTime(String giorno, String tipo) async {
    final TimeOfDay initial = tipo == 'apertura'
        ? const TimeOfDay(hour: 18, minute: 30)
        : const TimeOfDay(hour: 23, minute: 0);

    final TimeOfDay? picked = await showTimePicker(
      context: context,
      initialTime: initial,
    );

    if (picked != null) {
      setState(() {
        final formatted = '${picked.hour.toString().padLeft(2, '0')}:${picked.minute.toString().padLeft(2, '0')}';
        _orari[giorno]![tipo] = formatted;
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    if (_isLoading) {
      return const Scaffold(body: Center(child: CircularProgressIndicator()));
    }

    final theme = Theme.of(context);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Impostazioni'),
        actions: [
          Padding(
            padding: const EdgeInsets.only(right: 20),
            child: FilledButton.icon(
              onPressed: _isSaving ? null : _saveAllSettings,
              icon: _isSaving
                  ? const SizedBox(width: 16, height: 16, child: CircularProgressIndicator(strokeWidth: 2, color: Colors.white))
                  : const Icon(Icons.save),
              label: const Text('Salva Modifiche'),
            ),
          )
        ],
      ),
      body: Row(
        children: [
          NavigationRail(
            selectedIndex: _selectedSection,
            onDestinationSelected: (val) => setState(() => _selectedSection = val),
            labelType: NavigationRailLabelType.all,
            minWidth: 100,
            destinations: const [
              NavigationRailDestination(icon: Icon(Icons.storefront), label: Text('Ristorante')),
              NavigationRailDestination(icon: Icon(Icons.radar), label: Text('Zona Consegna')),
              NavigationRailDestination(icon: Icon(Icons.schedule), label: Text('Orari')),
              NavigationRailDestination(icon: Icon(Icons.map), label: Text('Google Maps')),
              NavigationRailDestination(icon: Icon(Icons.lock_person), label: Text('Account')),
              NavigationRailDestination(icon: Icon(Icons.tune), label: Text('Applicazione')),
            ],
          ),
          const VerticalDivider(thickness: 1, width: 1),
          Expanded(
            child: SingleChildScrollView(
              padding: const EdgeInsets.all(32),
              child: ConstrainedBox(
                constraints: const BoxConstraints(maxWidth: 800),
                child: _buildSectionContent(theme),
              ),
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildSectionContent(ThemeData theme) {
    switch (_selectedSection) {
      case 0:
        return _buildRistoranteSection(theme);
      case 1:
        return _buildZonaConsegnaSection(theme);
      case 2:
        return _buildOrariSection(theme);
      case 3:
        return _buildGoogleMapsSection(theme);
      case 4:
        return _buildAccountSection(theme);
      case 5:
        return _buildAppSection(theme);
      default:
        return const SizedBox();
    }
  }

  Widget _buildRistoranteSection(ThemeData theme) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text('Dati del Ristorante', style: theme.textTheme.headlineSmall?.copyWith(fontWeight: FontWeight.bold)),
        const SizedBox(height: 8),
        Text('Configura le informazioni identificative della sede.', style: theme.textTheme.bodyMedium),
        const SizedBox(height: 24),
        TextField(
          controller: _nomeController,
          decoration: const InputDecoration(labelText: 'Nome Ristorante', border: OutlineInputBorder()),
        ),
        const SizedBox(height: 20),
        TextField(
          controller: _indirizzoController,
          decoration: const InputDecoration(
            labelText: 'Indirizzo Completo',
            border: OutlineInputBorder(),
            helperText: 'Indirizzo di partenza per il routing dei fattorini.',
          ),
        ),
      ],
    );
  }

  Widget _buildZonaConsegnaSection(ThemeData theme) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text('Zona di Consegna', style: theme.textTheme.headlineSmall?.copyWith(fontWeight: FontWeight.bold)),
        const SizedBox(height: 8),
        Text('Definisci il raggio massimo coperto dal servizio a domicilio a partire dalla sede.', style: theme.textTheme.bodyMedium),
        const SizedBox(height: 32),
        Card(
          child: Padding(
            padding: const EdgeInsets.all(24),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    const Text('Raggio di copertura massimo:', style: TextStyle(fontSize: 16, fontWeight: FontWeight.w600)),
                    Text('${_zonaConsegnaKm.toStringAsFixed(1)} km', style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold, color: theme.colorScheme.primary)),
                  ],
                ),
                const SizedBox(height: 16),
                Slider(
                  value: _zonaConsegnaKm,
                  min: 1.0,
                  max: 30.0,
                  divisions: 58, // step di 0.5 km
                  label: '${_zonaConsegnaKm.toStringAsFixed(1)} km',
                  onChanged: (val) => setState(() => _zonaConsegnaKm = val),
                ),
                const SizedBox(height: 8),
                Text(
                  'Gli ordini con indirizzo oltre questo raggio verranno segnalati con un avviso di fuori-zona.',
                  style: theme.textTheme.bodySmall?.copyWith(color: theme.colorScheme.onSurfaceVariant),
                ),
              ],
            ),
          ),
        ),
      ],
    );
  }

  Widget _buildOrariSection(ThemeData theme) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text('Orari del Servizio Consegne', style: theme.textTheme.headlineSmall?.copyWith(fontWeight: FontWeight.bold)),
        const SizedBox(height: 8),
        Text('Imposta giorni di apertura e fasce orarie operative per i turni di consegna.', style: theme.textTheme.bodyMedium),
        const SizedBox(height: 24),
        Card(
          child: ListView.separated(
            shrinkWrap: true,
            physics: const NeverScrollableScrollPhysics(),
            itemCount: _giorniSettimana.length,
            separatorBuilder: (_, __) => const Divider(height: 1),
            itemBuilder: (context, index) {
              final giorno = _giorniSettimana[index];
              final info = _orari[giorno] ?? {'aperto': true, 'apertura': '18:30', 'chiusura': '23:00'};
              final bool aperto = info['aperto'] as bool;

              return ListTile(
                title: Text(giorno, style: const TextStyle(fontWeight: FontWeight.w600)),
                trailing: Row(
                  mainAxisSize: MainAxisSize.min,
                  children: [
                    Switch(
                      value: aperto,
                      onChanged: (val) => setState(() => _orari[giorno]!['aperto'] = val),
                    ),
                    const SizedBox(width: 16),
                    OutlinedButton(
                      onPressed: aperto ? () => _pickTime(giorno, 'apertura') : null,
                      child: Text('Apre: ${info['apertura']}'),
                    ),
                    const SizedBox(width: 8),
                    OutlinedButton(
                      onPressed: aperto ? () => _pickTime(giorno, 'chiusura') : null,
                      child: Text('Chiude: ${info['chiusura']}'),
                    ),
                  ],
                ),
              );
            },
          ),
        ),
      ],
    );
  }

  Widget _buildGoogleMapsSection(ThemeData theme) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text('Integrazione Google Maps', style: theme.textTheme.headlineSmall?.copyWith(fontWeight: FontWeight.bold)),
        const SizedBox(height: 8),
        Text('Configura la chiave API BYOK (Maps JavaScript, Directions e Geocoding).', style: theme.textTheme.bodyMedium),
        const SizedBox(height: 24),
        TextField(
          controller: _mapsApiKeyController,
          obscureText: _obscureApiKey,
          decoration: InputDecoration(
            labelText: 'Google Maps API Key',
            border: const OutlineInputBorder(),
            suffixIcon: IconButton(
              icon: Icon(_obscureApiKey ? Icons.visibility : Icons.visibility_off),
              onPressed: () => setState(() => _obscureApiKey = !_obscureApiKey),
            ),
          ),
        ),
        const SizedBox(height: 16),
        Row(
          children: [
            FilledButton.tonalIcon(
              icon: const Icon(Icons.health_and_safety_outlined),
              label: const Text('Testa Connessione API'),
              onPressed: () async {
                setState(() => _apiKeyTestStatus = 'Verifica in corso...');
                try {
                  final key = _mapsApiKeyController.text.trim();
                  final res = await http.get(Uri.parse(
                    'https://maps.googleapis.com/maps/api/geocode/json?address=Roma&key=$key',
                  ));
                  final data = jsonDecode(res.body);
                  setState(() {
                    _apiKeyTestStatus = data['status'] == 'OK'
                        ? 'Chiave API valida e funzionante!'
                        : 'Errore: ${data['error_message'] ?? data['status']}';
                  });
                } catch (e) {
                  setState(() => _apiKeyTestStatus = 'Errore di connessione: $e');
                }
              },
            ),
          ],
        ),
        if (_apiKeyTestStatus != null) ...[
          const SizedBox(height: 12),
          Text(
            _apiKeyTestStatus!,
            style: TextStyle(
              fontWeight: FontWeight.bold,
              color: _apiKeyTestStatus!.contains('valida') ? Colors.green : Colors.red,
            ),
          ),
        ]
      ],
    );
  }

  Widget _buildAccountSection(ThemeData theme) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text('Sicurezza Account', style: theme.textTheme.headlineSmall?.copyWith(fontWeight: FontWeight.bold)),
        const SizedBox(height: 8),
        Text('Modifica la password di accesso del gestore.', style: theme.textTheme.bodyMedium),
        const SizedBox(height: 24),
        TextField(
          controller: _currentPasswordController,
          obscureText: true,
          decoration: const InputDecoration(labelText: 'Password Attuale', border: OutlineInputBorder()),
        ),
        const SizedBox(height: 16),
        TextField(
          controller: _newPasswordController,
          obscureText: true,
          decoration: const InputDecoration(
            labelText: 'Nuova Password (min. 8 caratteri)',
            border: OutlineInputBorder(),
          ),
        ),
        const SizedBox(height: 16),
        TextField(
          controller: _confirmPasswordController,
          obscureText: true,
          decoration: const InputDecoration(labelText: 'Conferma Nuova Password', border: OutlineInputBorder()),
        ),
        const SizedBox(height: 20),
        FilledButton(
          onPressed: _handleChangePassword,
          child: const Text('Aggiorna Password'),
        ),
      ],
    );
  }

  Widget _buildAppSection(ThemeData theme) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text('Preferenze Applicazione', style: theme.textTheme.headlineSmall?.copyWith(fontWeight: FontWeight.bold)),
        const SizedBox(height: 8),
        Text('Configura aspetto visivo e tempi di blocco sessione.', style: theme.textTheme.bodyMedium),
        const SizedBox(height: 24),
        Card(
          child: Padding(
            padding: const EdgeInsets.all(20),
            child: Column(
              children: [
                ListTile(
                  title: const Text('Timeout Inattività Sessione'),
                  subtitle: const Text('Esegue il logout automatico in assenza di interazioni su mouse o tastiera.'),
                  trailing: DropdownButton<int>(
                    value: _sessionTimeoutMinutes,
                    items: const [
                      DropdownMenuItem(value: 15, child: Text('15 Minuti')),
                      DropdownMenuItem(value: 30, child: Text('30 Minuti (Default)')),
                      DropdownMenuItem(value: 60, child: Text('1 Ora')),
                      DropdownMenuItem(value: 120, child: Text('2 Ore')),
                      DropdownMenuItem(value: 0, child: Text('Disabilitato')),
                    ],
                    onChanged: (val) {
                      if (val != null) setState(() => _sessionTimeoutMinutes = val);
                    },
                  ),
                ),
                const Divider(),
                ListTile(
                  title: const Text('Tema Interfaccia'),
                  subtitle: const Text('Seleziona il tema grafico di DelivMap.'),
                  trailing: DropdownButton<ThemeMode>(
                    value: _selectedThemeMode,
                    items: const [
                      DropdownMenuItem(value: ThemeMode.system, child: Text('Segui Sistema')),
                      DropdownMenuItem(value: ThemeMode.light, child: Text('Chiaro')),
                      DropdownMenuItem(value: ThemeMode.dark, child: Text('Scuro')),
                    ],
                    onChanged: (val) {
                      if (val != null) setState(() => _selectedThemeMode = val);
                    },
                  ),
                ),
              ],
            ),
          ),
        ),
      ],
    );
  }
}
```

---

## 4.6 — Protezione delle Route Flutter

Per garantire che le schermate operative (Dashboard, Mappa Turno, Ordini, Impostazioni) siano accessibili unicamente ad operatore autenticato, viene configurato **`GoRouter`** con una guard reattiva collegata allo stato di Riverpod.

### Regole di Routing e Reindirizzamento

```
                       ┌─────────────────────────┐
                       │ Controllo AuthNotifier  │
                       └────────────┬────────────┘
                                    │
           ┌────────────────────────┼────────────────────────┐
           ▼                        ▼                        ▼
[First-Run Richiesto]      [Non Autenticato]           [Autenticato]
  Reindirizza a:              Reindirizza a:            Accesso libero a:
    /setup                       /login                 /dashboard, /settings
```

1. **Stato First-Run (`firstRunRequired`)**: Se il database non è inizializzato, qualunque rotta richiesta viene forzata verso `/setup`.
2. **Stato Non Autenticato (`unauthenticated` o `error`)**: Qualsiasi rotta diversa da `/login` e `/setup` viene reindirizzata a `/login`.
3. **Stato Autenticato (`authenticated`)**: Se l'utente si trova su `/login` o `/setup`, viene automaticamente indirizzato alla `/dashboard`.
4. **Reattività**: Il router è registrato con un `Listenable` derivato dallo state notifier Riverpod, scatenando il ri-orientamento istantaneo al logout o alla scadenza del timeout.

### Codice Completo di Routing (`app_router.dart`)

```dart
// lib/ui/app_router.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'providers/auth_provider.dart';
import 'screens/login_screen.dart';
import 'screens/first_run_wizard_screen.dart';
import 'screens/settings_screen.dart';

/// Adapter per notificare GoRouter delle modifiche dello stato Riverpod.
class RouterListenable extends ChangeNotifier {
  final Ref _ref;

  RouterListenable(this._ref) {
    _ref.listen<AuthState>(
      authNotifierProvider,
      (_, __) => notifyListeners(),
    );
  }
}

final routerListenableProvider = Provider<RouterListenable>((ref) {
  return RouterListenable(ref);
});

final goRouterProvider = Provider<GoRouter>((ref) {
  final refreshListenable = ref.watch(routerListenableProvider);

  return GoRouter(
    initialLocation: '/dashboard',
    refreshListenable: refreshListenable,
    redirect: (BuildContext context, GoRouterState state) {
      final authState = ref.read(authNotifierProvider);

      final isFirstRun = authState.status == AuthStatus.firstRunRequired;
      final isAuthenticated = authState.status == AuthStatus.authenticated;
      final isChecking = authState.status == AuthStatus.checkingFirstRun ||
                         authState.status == AuthStatus.initial;

      final currentPath = state.uri.path;
      final isGoingToLogin = currentPath == '/login';
      final isGoingToSetup = currentPath == '/setup';

      // Durante il boot non effettuare redirect prematuri
      if (isChecking) return null;

      // Se l'app non è configurata, forza il First-Run Wizard
      if (isFirstRun) {
        return isGoingToSetup ? null : '/setup';
      }

      // Se non autenticato, consenti solo /login
      if (!isAuthenticated) {
        return isGoingToLogin ? null : '/login';
      }

      // Se autenticato e sta provando ad andare a login o setup, manda a dashboard
      if (isGoingToLogin || isGoingToSetup) {
        return '/dashboard';
      }

      return null;
    },
    routes: [
      GoRoute(
        path: '/login',
        name: 'login',
        builder: (context, state) => const LoginScreen(),
      ),
      GoRoute(
        path: '/setup',
        name: 'setup',
        builder: (context, state) => const FirstRunWizardScreen(),
      ),
      GoRoute(
        path: '/dashboard',
        name: 'dashboard',
        builder: (context, state) => const Scaffold(
          body: Center(child: Text('Dashboard DelivMap (Fase 5)')),
        ),
      ),
      GoRoute(
        path: '/settings',
        name: 'settings',
        builder: (context, state) => const SettingsScreen(),
      ),
    ],
  );
});
```

---

## Criteri di Accettazione

I seguenti criteri definiscono in maniera inequivocabile il completamento della Fase 4:

### 4.1 Hashing delle Password
- [ ] Nessuna password in chiaro viene mai salvata nel database SQLite né trasmessa nei log di sistema.
- [ ] L'algoritmo impiegato è `bcrypt` con cost factor $\ge 12$.
- [ ] La validazione client e server rifiuta password con lunghezza inferiore a 8 caratteri o prive di lettere/cifre.

### 4.2 DAO e API Shelf
- [ ] `UtentiDao` gestisce correttamente creazione, ricerca per username, verifica delle credenziali e cambio password.
- [ ] Endpoint `/api/v1/auth/login` rilascia un token Bearer UUID valido con scadenza temporale.
- [ ] Endpoint `/api/v1/auth/logout` revoca immediatamente il token da `SessionManager`.
- [ ] Il middleware Shelf blocca qualsiasi chiamata non autorizzata verso endpoint protetti con risposta HTTP `401 Unauthorized`.

### 4.3 UI Schermata di Login
- [ ] Il form di login valida la presenza dei campi e mostra un feedback visivo immediato e comprensibile in caso di credenziali errate.
- [ ] Il pulsante di invio mostra lo stato di caricamento e previene chiamate multiple simultanee.
- [ ] La pressione del tasto `Invio` sul campo password avvia l'accesso.
- [ ] L'inattività dell'operatore per oltre 30 minuti (o valore impostato) comporta la disconnessione automatica e il ritorno alla schermata di login.

### 4.4 First-Run Wizard
- [ ] All'avvio su database vuoto, l'app avvia in automatico il wizard `/setup`.
- [ ] Il wizard impedisce il passaggio allo step successivo se i dati obbligatori non sono validi.
- [ ] Il salvataggio finale crea con successo il record del ristorante e l'utente amministratore, effettuando il login immediato senza richiedere un riavvio.

### 4.5 Pagina Impostazioni
- [ ] È possibile modificare nome del ristorante, indirizzo e raggio della zona di consegna (1-30 km).
- [ ] È possibile configurare l'orario di apertura/chiusura e i giorni di chiusura settimanale.
- [ ] È possibile inserire e testare la chiave Google Maps con esito immediato.
- [ ] È possibile cambiare la password del gestore previo inserimento della password attuale corretta.
- [ ] Tutte le modifiche persistono correttamente nel database Drift.

### 4.6 Protezione Route
- [ ] Il tentativo di navigare direttamente verso `/dashboard` o `/settings` da non autenticato redirige a `/login`.
- [ ] All'avvenuto login la navigazione reindirizza alla rotta principale senza sfarfallii visivi.

---

## Checklist di Testing Dettagliata

La presente checklist deve essere eseguita per la validazione della Fase 4 prima del passaggio alla Fase 5:

### 1. Test Unitari (Unit Testing)
- [ ] **Bcrypt Hashing**: Verificare che due chiamate a `hashPassword` con la medesima password generino hash differenti (per via del salt univoco casuale).
- [ ] **Bcrypt Verifica**: Verificare che `verifyPassword('Password123', hash)` restituisca `true` e `verifyPassword('Sbagliata', hash)` restituisca `false`.
- [ ] **Complessità Password**: Verificare che stringhe inferiori a 8 caratteri o composte solo da numeri o solo da lettere falliscano la validazione.
- [ ] **SessionManager Expiration**: Verificare che una sessione con `lastActivity` anteriore a 30 minuti venga considerata scaduta e rimossa.
- [ ] **SessionManager Touch**: Verificare che `validateAndTouch` aggiorni l'orario di attività prevenendo la scadenza.

### 2. Test di Integrazione API (Shelf + Drift)
- [ ] **Login con Credenziali Valide**: Invio di `POST /api/v1/auth/login` con credenziali corrette $\rightarrow$ HTTP 200 con token UUID e dati utente.
- [ ] **Login con Password Errata**: Invio con password scorretta $\rightarrow$ HTTP 401 `INVALID_CREDENTIALS`.
- [ ] **Login con Utente Inesistente**: Invio con username inesistente $\rightarrow$ HTTP 401 `INVALID_CREDENTIALS` (tempo di risposta uniforme per mitigare timing attacks).
- [ ] **Endpoint Protetto senza Token**: Invio di `GET /api/v1/auth/me` privo di header Authorization $\rightarrow$ HTTP 401 `UNAUTHORIZED`.
- [ ] **Endpoint Protetto con Token Falso o Scaduto**: Invio con Bearer token non presente in memoria $\rightarrow$ HTTP 401 `SESSION_EXPIRED`.
- [ ] **Logout**: Invio di `POST /api/v1/auth/logout` con token valido $\rightarrow$ HTTP 200; successiva chiamata allo stesso token $\rightarrow$ HTTP 401.
- [ ] **Cambio Password API**: Test di `PUT /api/v1/auth/password` con vecchia password corretta (successo) ed errata (errore 403).

### 3. Test Flusso Utente e UI (Flutter Desktop)
- [ ] **Scenario First-Run**:
  1. Avviare l'applicazione con file database rimosso o vuoto.
  2. Verificare l'apparizione immediata della schermata `/setup`.
  3. Completare lo Step 1 con nome locale e indirizzo.
  4. Nello Step 2 inserire password debole $\rightarrow$ verificare errore di validazione bloccante.
  5. Inserire password valida e completare lo Step 3 con test chiave Maps.
  6. Confermare lo Step 4 $\rightarrow$ verificare reindirizzamento automatico alla Dashboard.
- [ ] **Scenario Login e Navigazione**:
  1. Chiudere e riavviare l'app $\rightarrow$ verificare atterraggio su `/login`.
  2. Inserire credenziali errate $\rightarrow$ verificare comparsa SnackBar rossa di errore.
  3. Inserire credenziali corrette e premere `Invio` $\rightarrow$ accesso eseguito e rotta impostata su `/dashboard`.
- [ ] **Scenario Timeout di Inattività**:
  1. Eseguire l'accesso.
  2. Impostare temporaneamente il timeout a 10 secondi per il test (o simulare avanzamento orario).
  3. Non muovere mouse né premere tasti per oltre 10 secondi.
  4. Verificare che l'applicazione esegua il logout automatico e mostri la schermata `/login`.
  5. Ripetere muovendo continuamente il mouse $\rightarrow$ verificare che la sessione rimanga attiva.
- [ ] **Scenario Impostazioni**:
  1. Accedere a `/settings`.
  2. Modificare il raggio della zona di consegna tramite slider a 8.5 km.
  3. Modificare l'orario del Sabato a 19:00 - 23:30.
  4. Premere "Salva Modifiche" $\rightarrow$ verificare notifica di conferma.
  5. Riavviare l'app e tornare su `/settings` $\rightarrow$ verificare che i valori modificati siano stati mantenuti dal database.
