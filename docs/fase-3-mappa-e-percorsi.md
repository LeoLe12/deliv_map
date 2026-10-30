# Fase 3 — Mappa, Geocoding e Calcolo Percorsi

> **Stato di avanzamento**: Pianificazione dettagliata (Post-Revisione Tecnica)  
> **Scomposizione modulare**: **Fase 3a** (Geocoding, Impostazioni e Mappa WebView) e **Fase 3b** (Calcolo Percorsi Routes API, Giri di Consegna e Ottimizzazione)  
> **Durata stimata complessiva**: 6–8 giorni lavorativi (3a: 3–4 gg, 3b: 3–4 gg)  
> **Complessità tecnica**: Alta (integrazione cartografica WebView2, consumo Routes API, solver TSP locale, georeferenziazione asincrona resiliente)  
> **Riferimenti architetturali**: [Piano di sviluppo](file:///c:/Users/39366/StudioProjects/DelivMap/piano-sviluppo-app-consegne.md), [Stack tecnologico](file:///c:/Users/39366/StudioProjects/DelivMap/stack-tecnologico-e-strumenti.md), [Riepilogo Database](file:///c:/Users/39366/StudioProjects/DelivMap/riepilogo-database.md)

---

## Panoramica della Fase 3

La **Fase 3** introduce la cartografia interattiva e l'ottimizzazione logistica delle consegne a domicilio per DelivMap. L'obiettivo è dotare il ristorante di:
1. **Georeferenziazione automatica degli indirizzi** dei clienti tramite Google Maps Geocoding API.
2. **Visualizzazione cartografica desktop** tramite Microsoft Edge WebView2, con marker differenziati per stato operativo, sequenza di consegna e badge ritardo dinamico.
3. **Composizione di "Giri di Consegna" (sub-runs)**: un fattorino non esegue un anello continuo di 4 ore con 25 pizze fredde; le consegne vengono raggruppate in uscite omogenee per fascia oraria (soglia predefinita: 15 minuti) con partenza dalla sede del locale.
4. **Calcolo tratte ed ETA realistici** tramite la moderna **Google Routes API (`computeRoutes`)**, che sostituisce la dismessa Directions API Legacy.
5. **Suggerimento sequenza ottimale** calcolato tramite un solver TSP locale (permutazioni esatte fino a 8 tappe su distanze), per non superare lo SKU base *Routes Essentials* e azzerare i costi extra per il ristoratore.

### Architettura e Modello BYOK (*Bring Your Own Key*)

1. **Chiamate Google centralizzate nell'Isolate Shelf**: La UI Flutter non contatta mai Google direttamente né accede in chiaro alla chiave API. Tutte le chiamate REST (Geocoding e Routes API) vengono effettuate dal `MapsService` in esecuzione nell'Isolate del server Shelf locale.
2. **Quote Reali Google Maps Platform (dal 2024/2026)**:
   - Non esiste più il voucher cumulativo da 200\$/mese. Google applica soglie mensili gratuite specifiche per SKU:
     - **Geocoding API**: **10.000 richieste/mese gratuite** (successive a 5,00\$ ogni 1.000).
     - **Routes API (SKU Essentials)**: **10.000 richieste/mese gratuite** (successive a 5,00\$ ogni 1.000).
     - **Maps JavaScript API**: 28.500 caricamenti mappa/mese gratuiti.
   - Per una pizzeria o ristorante medio (500–1.200 ordini/mese con un riuso medio del 60% dell'anagrafica), i consumi rientrano **totalmente all'interno del free tier a costo zero**.
3. **Privacy & GDPR**:
   - Vengono inviati a Google Maps unicamente le stringhe di recapito stradale (es. *"Via Roma 12, Milano"*). Nessun nominativo, recapito telefonico, nota interna o importo esce dalla macchina locale.
   - La policy di conservazione delle coordinate nel DB locale SQLite rispetta il principio di minimizzazione e il diritto all'oblio (eliminazione cliente disaccoppiata dallo storico consegne).

---

## Flusso Architetturale della Fase 3

```mermaid
flowchart TD
    subgraph UI["Flutter Desktop UI (Main Isolate)"]
        UI_Order["Form Nuovo Ordine"]
        UI_Settings["Impostazioni (BYOK Key & Ristorante)"]
        UI_Turno["Dettaglio Turno (Split List / Map)"]
        UI_Giri["Gestione Giri (Selezione Sottogruppi)"]
        UI_WebView["Widget Mappa (WebView2 Desktop)"]
    end

    subgraph Shelf["Server Locale Shelf (Server Isolate)"]
        API_Orders["POST /api/v1/ordini"]
        API_Geocode["POST /api/v1/ordini/:id/geocodifica"]
        API_Settings["GET/PUT /api/v1/impostazioni"]
        API_MapConfig["GET /api/v1/impostazioni/mappa"]
        API_Route["POST /api/v1/percorsi/calcola"]
        MapsSvc["MapsService (Geocoding & computeRoutes)"]
        TSPSolver["Solver TSP Locale (<= 8 tappe)"]
    end

    subgraph DB["Database Locale (SQLite / Drift v2)"]
        T_Ordini["ordini (lat, lng, geocoding_stato, geocoded_at, place_id)"]
        T_Clienti["clienti (lat, lng, indirizzo_geocodificato)"]
        T_Risto["ristoranti (lat, lng, sosta_minuti, soglia_orario_minuti)"]
        T_Usage["maps_usage (mese, servizio, conteggio)"]
    end

    subgraph GoogleCloud["Google Maps Platform"]
        G_Geo["Geocoding API (REST)"]
        G_Routes["Routes API: computeRoutes (REST POST)"]
        G_JS["Maps JavaScript API (WebView)"]
    end

    UI_Settings -->|Test & Salva Key / Sede| API_Settings
    API_Settings --> MapsSvc
    MapsSvc -->|Verifica Key & Geocode Sede| G_Geo
    API_Settings --> T_Risto

    UI_Order -->|Crea Ordine| API_Orders
    API_Orders -->|Salva subito (da_fare)| T_Ordini
    API_Orders -.->|Task background isolato| MapsSvc
    MapsSvc -->|Lookup cache per indirizzo| T_Clienti
    MapsSvc -->|Cache miss: HTTP REST| G_Geo
    MapsSvc -->|Aggiorna coordinate| T_Ordini
    MapsSvc -->|Popola cache| T_Clienti

    UI_Giri -->|Invia ID tappe del giro| API_Route
    API_Route --> TSPSolver
    API_Route -->|computeRoutes per tratte ed ETA| MapsSvc
    MapsSvc -->|POST Routes Essentials| G_Routes
    API_Route -->|Restituisce tratte, polyline, ETA| UI_Turno

    UI_MapConfig -->|HTML + Key iniettata in RAM| UI_WebView
    UI_WebView --> G_JS
    UI_Turno -->|PostMessage: Markers & Polyline| UI_WebView
```

---

## 3.1 — Schema Database Canonico e Migrazione (Schema v1 → v2)

Lo schema estende le 5 tabelle canoniche concordate nelle Fasi 0–2 senza mai duplicare le definizioni di tabella.

### 1.1 Estensioni Tabelle Drift

1. **Tabella `clienti`** (`lib/data/tables/clienti.dart`):
   - `latitudine`: `RealColumn get latitudine => real().nullable()();`
   - `longitudine`: `RealColumn get longitudine => real().nullable()();`
   - `indirizzoGeocodificato`: `TextColumn get indirizzoGeocodificato => text().named('indirizzo_geocodificato').nullable()();`
   *(La cache è valida solo se `indirizzoGeocodificato == indirizzo.trim()`, evitando coordinate errate se il cliente ordina da un indirizzo diverso).*

2. **Tabella `ordini`** (`lib/data/tables/ordini.dart`):
   - Mantiene tutti i campi immutabili: `nomeClienteConsegna`, `telefonoConsegna`, `indirizzoConsegna`, `orarioRichiesto` (formato `HH:mm`), `stato` (con `StatoOrdineConverter`), `clienteId` (nullable, `onDelete: setNull`), `turnoId` (`onDelete: cascade`).
   - `latitudine`: `RealColumn get latitudine => real().nullable()();`
   - `longitudine`: `RealColumn get longitudine => real().nullable()();`
   - `geocodingStato`: `TextColumn get geocodingStato => text().named('geocoding_stato').withDefault(const Constant('da_fare'))();` (Valori ammessi: `'da_fare'`, `'ok'`, `'approssimato'`, `'fallito'`).
   - `geocodingErrore`: `TextColumn get geocodingErrore => text().named('geocoding_errore').nullable()();`
   - `geocodedAt`: `DateTimeColumn get geocodedAt => dateTime().named('geocoded_at').nullable()();`
   - `placeId`: `TextColumn get placeId => text().named('place_id').nullable()();`

3. **Tabella `ristoranti`** (`lib/data/tables/ristoranti.dart`):
   - Mantiene: `id`, `nome`, `indirizzo`, `zonaConsegna`, `mapsApiKey`, `evidenziaRitardi`, `tolleranzaRitardoMinuti`.
   - `latitudine`: `RealColumn get latitudine => real().nullable()();`
   - `longitudine`: `RealColumn get longitudine => real().nullable()();`
   - `sostaMinuti`: `IntColumn get sostaMinuti => integer().named('sosta_minuti').withDefault(const Constant(3))();`
   - `sogliaOrarioMinuti`: `IntColumn get sogliaOrarioMinuti => integer().named('soglia_orario_minuti').withDefault(const Constant(15))();`

4. **Nuova Tabella `maps_usage`** (`lib/data/tables/maps_usage.dart`):
   - Traccia i consumi mensili del ristorante per evitare sforamenti involontari delle quote gratuite.
   ```dart
   import 'package:drift/drift.dart';

   @DataClassName('MapsUsageEntry')
   class MapsUsage extends Table {
     IntColumn get id => integer().autoIncrement()();
     TextColumn get mese => text().withLength(min: 7, max: 7)(); // YYYY-MM
     TextColumn get servizio => text()(); // 'geocoding' | 'routes'
     IntColumn get conteggio => integer().withDefault(const Constant(0))();

     @override
     List<Set<Column>> get uniqueKeys => [{mese, servizio}];
   }
   ```

### 1.2 Configurazione `AppDatabase` e Strategia di Migrazione

```dart
// lib/data/database.dart
import 'dart:io';
import 'package:drift/drift.dart';
import 'package:drift/native.dart';
import 'package:path/path.dart' as p;

import 'converters/stato_ordine_converter.dart';
import 'tables/clienti.dart';
import 'tables/maps_usage.dart';
import 'tables/ordini.dart';
import 'tables/ristoranti.dart';
import 'tables/turni.dart';
import 'tables/utenti.dart';

part 'database.g.dart';

@DriftDatabase(tables: [
  Ristoranti,
  Utenti,
  Clienti,
  Turni,
  Ordini,
  MapsUsage,
])
class AppDatabase extends _$AppDatabase {
  final String dbPath;

  AppDatabase({required this.dbPath, QueryExecutor? executor})
      : super(executor ?? _openConnection(dbPath));

  @override
  int get schemaVersion => 2;

  @override
  MigrationStrategy get migration {
    return MigrationStrategy(
      beforeOpen: (details) async {
        await customStatement('PRAGMA foreign_keys = ON;');
        await customStatement('PRAGMA journal_mode = WAL;');
      },
      onCreate: (Migrator m) async {
        await m.createAll();
        await into(ristoranti).insert(
          RistorantiCompanion.insert(
            nome: 'Il Mio Ristorante',
            indirizzo: 'Via Roma 1, Milano',
            zonaConsegna: '5',
            sostaMinuti: const Value(3),
            sogliaOrarioMinuti: const Value(15),
          ),
        );
      },
      onUpgrade: (Migrator m, int from, int to) async {
        if (from < 2) {
          // Migrazione tabelle esistenti v1 -> v2
          await m.addColumn(clienti, clienti.latitudine);
          await m.addColumn(clienti, clienti.longitudine);
          await m.addColumn(clienti, clienti.indirizzoGeocodificato);

          await m.addColumn(ordini, ordini.latitudine);
          await m.addColumn(ordini, ordini.longitudine);
          await m.addColumn(ordini, ordini.geocodingStato);
          await m.addColumn(ordini, ordini.geocodingErrore);
          await m.addColumn(ordini, ordini.geocodedAt);
          await m.addColumn(ordini, ordini.placeId);

          await m.addColumn(ristoranti, ristoranti.latitudine);
          await m.addColumn(ristoranti, ristoranti.longitudine);
          await m.addColumn(ristoranti, ristoranti.sostaMinuti);
          await m.addColumn(ristoranti, ristoranti.sogliaOrarioMinuti);

          await m.createTable(mapsUsage);
        }
      },
    );
  }
}

LazyDatabase _openConnection(String dbPath) {
  return LazyDatabase(() async {
    final file = File(dbPath);
    if (!await file.parent.exists()) {
      await file.parent.create(recursive: true);
    }
    return NativeDatabase.createInBackground(file);
  });
}
```

---

## 3.2 — Modelli DTO e Coordinate Geospaziali

### 2.1 Modello Puro `LatLng` con Validazione

```dart
// lib/models/lat_lng.dart
class LatLng {
  final double latitude;
  final double longitude;

  const LatLng(this.latitude, this.longitude);

  static LatLng validated(double lat, double lng) {
    if (lat < -90.0 || lat > 90.0) {
      throw ArgumentError('Latitudine non valida (-90..90): $lat');
    }
    if (lng < -180.0 || lng > 180.0) {
      throw ArgumentError('Longitudine non valida (-180..180): $lng');
    }
    return LatLng(lat, lng);
  }

  Map<String, dynamic> toJson() => {'lat': latitude, 'lng': longitude};

  factory LatLng.fromJson(Map<String, dynamic> json) {
    return LatLng.validated(
      (json['lat'] as num).toDouble(),
      (json['lng'] as num).toDouble(),
    );
  }

  @override
  String toString() => 'LatLng($latitude, $longitude)';

  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      other is LatLng && latitude == other.latitude && longitude == other.longitude;

  @override
  int get hashCode => latitude.hashCode ^ longitude.hashCode;
}
```

### 2.2 Estensione di `OrdineResponse` con Dati Cartografici

I DTO definiti in `lib/models/ordine_dtos.dart` vengono estesi in modo non distruttivo senza creare modelli paralleli:

```dart
// Estensione di OrdineResponse in lib/models/ordine_dtos.dart
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

  // Nuovi campi Fase 3
  final double? latitudine;
  final double? longitudine;
  @JsonKey(name: 'geocoding_stato')
  final String geocodingStato; // 'da_fare', 'ok', 'approssimato', 'fallito'
  @JsonKey(name: 'geocoding_errore')
  final String? geocodingErrore;

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
    this.latitudine,
    this.longitudine,
    this.geocodingStato = 'da_fare',
    this.geocodingErrore,
  });

  factory OrdineResponse.fromJson(Map<String, dynamic> json) => _$OrdineResponseFromJson(json);
  Map<String, dynamic> toJson() => _$OrdineResponseToJson(this);
}
```

---

## 3.3 — MapsService: Geocoding con Bounds e Routing con Routes API

Tutto il codice seguente viene eseguito **esclusivamente nell'Isolate del server Shelf**.

### 3.1 Eccezioni Maps Senza Leak della Chiave

```dart
// lib/services/maps/maps_exceptions.dart
class MapsApiException implements Exception {
  final String message;
  final int? statusCode;
  final String? errorCode;

  const MapsApiException(this.message, {this.statusCode, this.errorCode});

  @override
  String toString() => 'MapsApiException: $message (code: $errorCode)';
}

class ApiKeyInvalidException extends MapsApiException {
  const ApiKeyInvalidException([String msg = 'Chiave Google Maps non valida o priva di autorizzazioni'])
      : super(msg, errorCode: 'API_KEY_INVALID');
}

class ApiNotEnabledException extends MapsApiException {
  const ApiNotEnabledException(String apiName)
      : super('API non abilitata su Google Cloud Console: $apiName', errorCode: 'API_NOT_ENABLED');
}

class AddressNotFoundException extends MapsApiException {
  const AddressNotFoundException([String msg = 'Indirizzo non trovato su Google Maps'])
      : super(msg, errorCode: 'ZERO_RESULTS');
}

class NetworkException extends MapsApiException {
  const NetworkException(String reason)
      : super('Errore di connessione verso Google Maps Platform: $reason', errorCode: 'NETWORK_ERROR');
}
```

### 3.2 Implementazione `MapsService`

```dart
// lib/services/maps/maps_service.dart
import 'dart:convert';
import 'package:http/http.dart' as http;
import '../../models/lat_lng.dart';
import 'maps_exceptions.dart';

enum GeocodingPrecision { ok, approssimato, fallito }

class GeocodingResult {
  final LatLng coordinates;
  final GeocodingPrecision precisione;
  final String? placeId;
  final String? formattedAddress;

  GeocodingResult({
    required this.coordinates,
    required this.precisione,
    this.placeId,
    this.formattedAddress,
  });
}

class MapsService {
  final http.Client _client;
  static const String _geocodeEndpoint = 'https://maps.googleapis.com/maps/api/geocode/json';
  static const String _computeRoutesEndpoint = 'https://routes.googleapis.com/directions/v2:computeRoutes';

  MapsService({http.Client? client}) : _client = client ?? http.Client();

  /// Geocodifica un indirizzo con restrizione su Italia e bounds attorno alla sede
  Future<GeocodingResult?> geocodeAddress({
    required String indirizzo,
    required String apiKey,
    LatLng? restaurantCenter,
  }) async {
    final cleanAddress = indirizzo.trim();
    if (cleanAddress.isEmpty) throw const AddressNotFoundException('Indirizzo vuoto');

    final queryParams = <String, String>{
      'address': cleanAddress,
      'key': apiKey.trim(),
      'language': 'it',
      'region': 'it',
      'components': 'country:IT',
    };

    // Imposta un rettangolo di priorità (bounds) di circa 15 km attorno al ristorante
    if (restaurantCenter != null) {
      const delta = 0.15;
      queryParams['bounds'] =
          '${restaurantCenter.latitude - delta},${restaurantCenter.longitude - delta}|'
          '${restaurantCenter.latitude + delta},${restaurantCenter.longitude + delta}';
    }

    final uri = Uri.parse(_geocodeEndpoint).replace(queryParameters: queryParams);

    http.Response response;
    try {
      response = await _client.get(uri).timeout(const Duration(seconds: 8));
    } catch (e) {
      throw const NetworkException('Connessione scaduta o assente');
    }

    if (response.statusCode != 200) {
      throw MapsApiException('Risposta HTTP ${response.statusCode} da Geocoding API', statusCode: response.statusCode);
    }

    final Map<String, dynamic> data = jsonDecode(response.body);
    final status = data['status'] as String?;

    if (status == 'ZERO_RESULTS') {
      return null;
    }
    if (status == 'REQUEST_DENIED') {
      throw const ApiKeyInvalidException();
    }
    if (status != 'OK') {
      throw MapsApiException('Geocoding fallito con stato: $status', errorCode: status);
    }

    final results = data['results'] as List<dynamic>;
    if (results.isEmpty) return null;

    final first = results.first as Map<String, dynamic>;
    final geometry = first['geometry'] as Map<String, dynamic>;
    final location = geometry['location'] as Map<String, dynamic>;
    final locationType = geometry['location_type'] as String?;
    final isPartialMatch = first['partial_match'] == true;
    final placeId = first['place_id'] as String?;
    final formattedAddress = first['formatted_address'] as String?;

    final lat = (location['lat'] as num).toDouble();
    final lng = (location['lng'] as num).toDouble();

    // Valutazione precisione del geocoding
    final precisione = (!isPartialMatch && (locationType == 'ROOFTOP' || locationType == 'RANGE_INTERPOLATED'))
        ? GeocodingPrecision.ok
        : GeocodingPrecision.approssimato;

    return GeocodingResult(
      coordinates: LatLng.validated(lat, lng),
      precisione: precisione,
      placeId: placeId,
      formattedAddress: formattedAddress,
    );
  }

  /// Verifica la validità della chiave testando sia Geocoding che Routes API
  Future<String> testApiKey(String apiKey) async {
    try {
      // 1. Test rapido su Geocoding (Roma centro)
      final geoTestUri = Uri.parse(_geocodeEndpoint).replace(queryParameters: {
        'address': 'Roma',
        'key': apiKey.trim(),
      });
      final geoRes = await _client.get(geoTestUri).timeout(const Duration(seconds: 6));
      final geoData = jsonDecode(geoRes.body);
      if (geoData['status'] == 'REQUEST_DENIED') {
        return 'chiave_non_valida';
      }

      // 2. Test su Routes API (tratta 100m)
      final routesRes = await _client.post(
        Uri.parse(_computeRoutesEndpoint),
        headers: {
          'Content-Type': 'application/json',
          'X-Goog-Api-Key': apiKey.trim(),
          'X-Goog-FieldMask': 'routes.duration',
        },
        body: jsonEncode({
          'origin': {'location': {'latLng': {'latitude': 41.9028, 'longitude': 12.4964}}},
          'destination': {'location': {'latLng': {'latitude': 41.9035, 'longitude': 12.4970}}},
          'travelMode': 'DRIVE',
        }),
      ).timeout(const Duration(seconds: 6));

      if (routesRes.statusCode == 403 || routesRes.statusCode == 400) {
        final errJson = jsonDecode(routesRes.body);
        final status = errJson['error']?['status'];
        if (status == 'PERMISSION_DENIED') return 'routes_non_abilitata';
        return 'chiave_non_valida';
      }

      return 'ok';
    } catch (_) {
      return 'errore_rete';
    }
  }

  /// Calcola tratte ed ETA per un giro di consegna tramite Google Routes API
  Future<GiroPercorsoResult> computeGiroRoute({
    required LatLng origine,
    required List<LatLng> waypoints,
    required String apiKey,
    required DateTime oraPartenza,
    int sostaMinutiPerTappa = 3,
  }) async {
    if (waypoints.isEmpty) throw ArgumentError('Nessun waypoint fornito');
    if (waypoints.length > 25) {
      throw const MapsApiException('Il giro supera il massimo di 25 tappe ammesse da Routes API');
    }

    final intermediates = waypoints.map((w) => {
      'location': {
        'latLng': {'latitude': w.latitude, 'longitude': w.longitude}
      }
    }).toList();

    final bodyPayload = {
      'origin': {
        'location': {
          'latLng': {'latitude': origine.latitude, 'longitude': origine.longitude}
        }
      },
      'destination': {
        'location': {
          'latLng': {'latitude': origine.latitude, 'longitude': origine.longitude}
        }
      },
      'intermediates': intermediates,
      'travelMode': 'DRIVE',
      'routingPreference': 'TRAFFIC_AWARE',
    };

    final response = await _client.post(
      Uri.parse(_computeRoutesEndpoint),
      headers: {
        'Content-Type': 'application/json',
        'X-Goog-Api-Key': apiKey.trim(),
        'X-Goog-FieldMask':
            'routes.duration,routes.distanceMeters,routes.polyline.encodedPolyline,routes.legs.duration,routes.legs.distanceMeters',
      },
      body: jsonEncode(bodyPayload),
    ).timeout(const Duration(seconds: 12));

    if (response.statusCode != 200) {
      throw MapsApiException('Errore HTTP ${response.statusCode} da Routes API');
    }

    final data = jsonDecode(response.body);
    final routes = data['routes'] as List<dynamic>?;
    if (routes == null || routes.isEmpty) {
      throw const MapsApiException('Nessun percorso calcolabile per le tappe fornite');
    }

    final route = routes.first;
    final totalDistance = (route['distanceMeters'] as num?)?.toInt() ?? 0;
    final totalDurationSec = _parseDurationSeconds(route['duration'] as String?);
    final encodedPolyline = route['polyline']?['encodedPolyline'] as String? ?? '';
    final legs = route['legs'] as List<dynamic>? ?? [];

    final tratteCalcolate = <TrattaInfo>[];
    DateTime orarioProgressivo = oraPartenza;

    for (int i = 0; i < waypoints.length && i < legs.length; i++) {
      final leg = legs[i];
      final legDist = (leg['distanceMeters'] as num?)?.toInt() ?? 0;
      final legSec = _parseDurationSeconds(leg['duration'] as String?);

      // Tempo di arrivo alla consegna = orario corrente + tempo di guida
      orarioProgressivo = orarioProgressivo.add(Duration(seconds: legSec));
      final etaConsegna = orarioProgressivo;

      // Aggiunge tempo di sosta per il cliente prima della tappa successiva
      orarioProgressivo = orarioProgressivo.add(Duration(minutes: sostaMinutiPerTappa));

      tratteCalcolate.add(TrattaInfo(
        tappaIndex: i,
        distanzaMetri: legDist,
        durataSecondi: legSec,
        etaPrevisto: etaConsegna,
      ));
    }

    return GiroPercorsoResult(
      distanzaTotaleMetri: totalDistance,
      durataTotaleSecondi: totalDurationSec,
      encodedPolyline: encodedPolyline,
      tratte: tratteCalcolate,
    );
  }

  static int _parseDurationSeconds(String? durationStr) {
    if (durationStr == null) return 0;
    // Formato Google Routes: "420s"
    final clean = durationStr.replaceAll('s', '');
    return int.tryParse(clean) ?? 0;
  }
}

class TrattaInfo {
  final int tappaIndex;
  final int distanzaMetri;
  final int durataSecondi;
  final DateTime etaPrevisto;

  TrattaInfo({
    required this.tappaIndex,
    required this.distanzaMetri,
    required this.durataSecondi,
    required this.etaPrevisto,
  });

  Map<String, dynamic> toJson() => {
    'tappa_index': tappaIndex,
    'distanza_metri': distanzaMetri,
    'durata_secondi': durataSecondi,
    'eta_previsto': etaPrevisto.toIso8601String(),
  };
}

class GiroPercorsoResult {
  final int distanzaTotaleMetri;
  final int durataTotaleSecondi;
  final String encodedPolyline;
  final List<TrattaInfo> tratte;

  GiroPercorsoResult({
    required this.distanzaTotaleMetri,
    required this.durataTotaleSecondi,
    required this.encodedPolyline,
    required this.tratte,
  });

  Map<String, dynamic> toJson() => {
    'distanza_totale_metri': distanzaTotaleMetri,
    'durata_totale_secondi': durataTotaleSecondi,
    'encoded_polyline': encodedPolyline,
    'tratte': tratte.map((t) => t.toJson()).toList(),
  };
}
```

---

## 3.4 — Sottogruppi di Consegna ("Giri") e Solver TSP Locale

Per non far scattare lo SKU *Routes Pro* di Google (costoso e con metà quota gratuita), la sequenza di consegna all'interno del sottogruppo viene ordinata **in locale** sulla CPU del server tramite un solver di permutazioni esatte fino a 8 ordini.

```dart
// lib/services/maps/local_tsp_solver.dart
import 'dart:math';
import '../../models/lat_lng.dart';

class TspItem {
  final int ordineId;
  final LatLng coordinate;
  final String orarioRichiesto; // HH:mm

  TspItem({required this.ordineId, required this.coordinate, required this.orarioRichiesto});
}

class LocalTspSolver {
  /// Calcola la distanza euclidea approssimata tra due punti geografici (in km)
  static double distKm(LatLng a, LatLng b) {
    const r = 6371.0;
    final dLat = (b.latitude - a.latitude) * pi / 180.0;
    final dLng = (b.longitude - a.longitude) * pi / 180.0;
    final lat1 = a.latitude * pi / 180.0;
    final lat2 = b.latitude * pi / 180.0;

    final h = sin(dLat / 2) * sin(dLat / 2) +
        cos(lat1) * cos(lat2) * sin(dLng / 2) * sin(dLng / 2);
    return 2 * r * asin(sqrt(h));
  }

  /// Trova la sequenza ottimale tra origine (ristorante) e la lista di tappe
  static List<TspItem> optimize(LatLng origine, List<TspItem> items) {
    if (items.length <= 2) return items;

    // Fino a 8 tappe si esegue la ricerca della permutazione minima esatta (8! = 40.320 cicli, < 3 ms in Dart)
    if (items.length <= 8) {
      double bestDist = double.infinity;
      List<TspItem> bestSeq = items;

      for (final perm in _permute(items)) {
        double currentDist = distKm(origine, perm.first.coordinate);
        for (int i = 0; i < perm.length - 1; i++) {
          currentDist += distKm(perm[i].coordinate, perm[i + 1].coordinate);
        }
        currentDist += distKm(perm.last.coordinate, origine); // Ritorno

        if (currentDist < bestDist) {
          bestDist = currentDist;
          bestSeq = perm;
        }
      }
      return bestSeq;
    }

    // Oltre 8 tappe si applica l'euristica Nearest-Neighbor
    final unvisited = List<TspItem>.from(items);
    final route = <TspItem>[];
    LatLng currentPos = origine;

    while (unvisited.isNotEmpty) {
      unvisited.sort((a, b) => distKm(currentPos, a.coordinate).compareTo(distKm(currentPos, b.coordinate)));
      final nearest = unvisited.removeAt(0);
      route.add(nearest);
      currentPos = nearest.coordinate;
    }
    return route;
  }

  static Iterable<List<TspItem>> _permute(List<TspItem> list) sync* {
    if (list.length <= 1) {
      yield list;
      return;
    }
    for (int i = 0; i < list.length; i++) {
      final current = list[i];
      final remaining = List<TspItem>.from(list)..removeAt(i);
      for (final sub in _permute(remaining)) {
        yield [current, ...sub];
      }
    }
  }
}
```

---

## 3.5 — Template HTML/JS per WebView2 Desktop (`assets/map/map_view.html`)

Il template carica Google Maps JS in modo sicuro senza file temporanei su disco, evita XSS usando `textContent` ed evita di resettare lo zoom della mappa ad ogni aggiornamento dello stato.

```html
<!DOCTYPE html>
<html lang="it">
<head>
  <meta charset="utf-8" />
  <title>DelivMap - Mappa Turno</title>
  <style>
    html, body, #map { height: 100%; margin: 0; padding: 0; font-family: 'Segoe UI', Tahoma, sans-serif; }
    .card { padding: 8px; font-size: 13px; max-width: 250px; }
    .title { font-weight: bold; font-size: 14px; margin-bottom: 4px; }
    .row { margin: 2px 0; color: #333; }
    .badge { display: inline-block; padding: 2px 6px; border-radius: 4px; color: #fff; font-size: 11px; font-weight: bold; margin-top: 4px; }
    .badge-in_attesa { background-color: #64748B; }
    .badge-in_consegna { background-color: #2563EB; }
    .badge-consegnato { background-color: #16A34A; }
    .badge-ritardo { background-color: #DC2626; }
  </style>
</head>
<body>
  <div id="map"></div>

  <script>
    let map;
    let markers = [];
    let restaurantMarker = null;
    let routePolyline = null;
    let sharedInfoWindow = null;
    let isInitialFit = false;

    const COLORS = {
      'in_attesa': '#64748B',
      'in_consegna': '#2563EB',
      'consegnato': '#16A34A',
      'ritardo': '#DC2626'
    };

    function initMap() {
      sharedInfoWindow = new google.maps.InfoWindow();
      map = new google.maps.Map(document.getElementById('map'), {
        zoom: 13,
        center: { lat: 41.9028, lng: 12.4964 },
        mapTypeControl: false,
        streetViewControl: false,
        fullscreenControl: false,
      });

      notifyFlutter({ event: 'MAP_READY' });
    }

    function notifyFlutter(payload) {
      if (window.chrome && window.chrome.webview) {
        window.chrome.webview.postMessage(JSON.stringify(payload));
      }
    }

    function buildMarkerSvg(numero, colore, isRitardo) {
      const strokeColor = isRitardo ? '#DC2626' : '#FFFFFF';
      const strokeWidth = isRitardo ? '3' : '2';
      const svg = `
        <svg xmlns="http://www.w3.org/2000/svg" width="36" height="48" viewBox="0 0 36 48">
          <path fill="${colore}" stroke="${strokeColor}" stroke-width="${strokeWidth}" d="M18 0 C8.06 0 0 8.06 0 18 C0 31.5 18 48 18 48 C18 48 36 31.5 36 18 C36 8.06 27.94 0 18 0 Z"/>
          <circle cx="18" cy="18" r="12" fill="#FFFFFF"/>
          <text x="18" y="22" font-size="12" font-weight="bold" fill="${colore}" text-anchor="middle">${numero}</text>
        </svg>`;
      return {
        url: 'data:image/svg+xml;charset=UTF-8,' + encodeURIComponent(svg),
        scaledSize: new google.maps.Size(36, 48),
        anchor: new google.maps.Point(18, 48)
      };
    }

    function setRestaurant(data) {
      const pos = { lat: data.lat, lng: data.lng };
      if (restaurantMarker) {
        restaurantMarker.setPosition(pos);
      } else {
        const ristoSvg = `
          <svg xmlns="http://www.w3.org/2000/svg" width="40" height="40" viewBox="0 0 24 24">
            <circle cx="12" cy="12" r="11" fill="#EA580C" stroke="#FFFFFF" stroke-width="2"/>
            <path fill="#FFFFFF" d="M11 9H9V2H7v7H5V2H3v7c0 2.12 1.66 3.84 3.75 3.97V22h2.5v-9.03C11.34 12.84 13 11.12 13 9V2h-2v7zm5-3v8h2.5v8H21V2c-2.76 0-5 2.24-5 4z"/>
          </svg>`;
        restaurantMarker = new google.maps.Marker({
          position: pos,
          map: map,
          title: data.nome || 'Ristorante',
          icon: {
            url: 'data:image/svg+xml;charset=UTF-8,' + encodeURIComponent(ristoSvg),
            scaledSize: new google.maps.Size(40, 40),
            anchor: new google.maps.Point(20, 20)
          },
          zIndex: 999
        });
      }
      if (!isInitialFit) {
        map.setCenter(pos);
      }
    }

    function updateMarkers(orders) {
      markers.forEach(m => m.setMap(null));
      markers = [];

      const bounds = new google.maps.LatLngBounds();
      if (restaurantMarker) bounds.extend(restaurantMarker.getPosition());

      orders.forEach((o, idx) => {
        if (o.lat == null || o.lng == null) return;
        const pos = { lat: o.lat, lng: o.lng };
        bounds.extend(pos);

        const colore = COLORS[o.stato] || '#64748B';
        const marker = new google.maps.Marker({
          position: pos,
          map: map,
          icon: buildMarkerSvg(o.sequenza + 1, colore, o.is_ritardo),
          title: `#${o.sequenza + 1} - ${o.cliente_nome}`
        });

        marker.addListener('click', () => {
          // Creazione card sicura con textContent anti-XSS
          const container = document.createElement('div');
          container.className = 'card';

          const title = document.createElement('div');
          title.className = 'title';
          title.textContent = `#${o.sequenza + 1} - ${o.cliente_nome}`;
          container.appendChild(title);

          const rigaIndirizzo = document.createElement('div');
          rigaIndirizzo.className = 'row';
          rigaIndirizzo.textContent = o.indirizzo;
          container.appendChild(rigaIndirizzo);

          const rigaInfo = document.createElement('div');
          rigaInfo.className = 'row';
          rigaInfo.textContent = `Tel: ${o.telefono} | Ore: ${o.orario_richiesto} | € ${parseFloat(o.importo).toFixed(2)}`;
          container.appendChild(rigaInfo);

          if (o.note) {
            const rigaNote = document.createElement('div');
            rigaNote.className = 'row';
            rigaNote.textContent = `Note: ${o.note}`;
            container.appendChild(rigaNote);
          }

          sharedInfoWindow.setContent(container);
          sharedInfoWindow.open(map, marker);

          notifyFlutter({ event: 'MARKER_CLICKED', ordineId: o.id });
        });

        markers.push(marker);
      });

      // Esegue fitBounds solo al primo render o su richiesta esplicita
      if (!isInitialFit && !bounds.isEmpty()) {
        map.fitBounds(bounds, { top: 50, bottom: 50, left: 50, right: 50 });
        isInitialFit = true;
      }
    }

    function drawRoute(encodedPolyline) {
      if (routePolyline) {
        routePolyline.setMap(null);
        routePolyline = null;
      }
      if (!encodedPolyline || !google.maps.geometry) return;

      const path = google.maps.geometry.encoding.decodePath(encodedPolyline);
      routePolyline = new google.maps.Polyline({
        path: path,
        geodesic: true,
        strokeColor: '#0284C7',
        strokeOpacity: 0.9,
        strokeWeight: 5,
        map: map
      });
    }

    // Bridge messaggi da Flutter
    if (window.chrome && window.chrome.webview) {
      window.chrome.webview.addEventListener('message', ev => {
        try {
          const msg = typeof ev.data === 'string' ? JSON.parse(ev.data) : ev.data;
          if (msg.action === 'SET_RESTAURANT') setRestaurant(msg.data);
          if (msg.action === 'UPDATE_MARKERS') updateMarkers(msg.data);
          if (msg.action === 'DRAW_ROUTE') drawRoute(msg.data.encodedPolyline);
          if (msg.action === 'FIT_BOUNDS') {
            isInitialFit = false;
            updateMarkers(msg.data || []);
          }
        } catch (e) {
          console.error(e);
        }
      });
    }
  </script>
  <script id="gmaps-script"></script>
</body>
</html>
```

---

## 3.6 — Widget Flutter Desktop `MappaTurnoWidget`

Il widget carica la pagina HTML direttamente tramite `loadStringContent`, eliminando qualsiasi salvataggio di chiavi in chiaro su file temporanei su disco.

```dart
// lib/ui/widgets/mappa_turno_widget.dart
import 'dart:convert';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart' show rootBundle;
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:webview_windows/webview_windows.dart';

import '../../models/lat_lng.dart';
import '../../models/ordine_dtos.dart';

class MappaTurnoWidget extends StatefulWidget {
  final List<OrdineResponse> ordini;
  final LatLng coordinateRistorante;
  final String nomeRistorante;
  final String? encodedPolyline;
  final String apiKey;
  final void Function(int ordineId)? onMarkerSelected;

  const MappaTurnoWidget({
    super.key,
    required this.ordini,
    required this.coordinateRistorante,
    required this.nomeRistorante,
    required this.apiKey,
    this.encodedPolyline,
    this.onMarkerSelected,
  });

  @override
  State<MappaTurnoWidget> createState() => _MappaTurnoWidgetState();
}

class _MappaTurnoWidgetState extends State<MappaTurnoWidget> {
  final WebviewController _controller = WebviewController();
  bool _isInitialized = false;
  bool _mapReady = false;
  String? _errorMessage;

  @override
  void initState() {
    super.initState();
    _initWebView();
  }

  Future<void> _initWebView() async {
    try {
      await _controller.initialize();

      _controller.webMessage.listen((rawMsg) {
        try {
          final data = jsonDecode(rawMsg) as Map<String, dynamic>;
          if (data['event'] == 'MAP_READY') {
            setState(() => _mapReady = true);
            _sendRestaurant();
            _sendMarkers();
            if (widget.encodedPolyline != null) {
              _sendPolyline(widget.encodedPolyline!);
            }
          } else if (data['event'] == 'MARKER_CLICKED') {
            final id = data['ordineId'] as int?;
            if (id != null && widget.onMarkerSelected != null) {
              widget.onMarkerSelected!(id);
            }
          }
        } catch (_) {}
      });

      String html = await rootBundle.loadString('assets/map/map_view.html');
      final scriptTag =
          '<script src="https://maps.googleapis.com/maps/api/js?key=${widget.apiKey}&callback=initMap&libraries=geometry&language=it" async defer></script>';
      html = html.replaceFirst('<script id="gmaps-script"></script>', scriptTag);

      await _controller.loadStringContent(html);

      if (mounted) setState(() => _isInitialized = true);
    } catch (e) {
      if (mounted) {
        setState(() {
          _errorMessage =
              'Impossibile inizializzare Microsoft Edge WebView2.\nVerifica che il runtime WebView2 sia installato sul sistema.';
        });
      }
    }
  }

  void _sendRestaurant() {
    if (!_mapReady) return;
    _post({
      'action': 'SET_RESTAURANT',
      'data': {
        'lat': widget.coordinateRistorante.latitude,
        'lng': widget.coordinateRistorante.longitude,
        'nome': widget.nomeRistorante,
      }
    });
  }

  void _sendMarkers() {
    if (!_mapReady) return;
    final payload = widget.ordini.map((o) => {
      'id': o.id,
      'sequenza': o.sequenza,
      'cliente_nome': o.cliente.nome,
      'indirizzo': o.cliente.indirizzo,
      'telefono': o.cliente.telefono,
      'orario_richiesto': o.orarioRichiesto,
      'importo': o.importo,
      'stato': o.stato,
      'is_ritardo': o.isInRitardo,
      'note': o.note,
      'lat': o.latitudine,
      'lng': o.longitudine,
    }).toList();

    _post({'action': 'UPDATE_MARKERS', 'data': payload});
  }

  void _sendPolyline(String polyline) {
    if (!_mapReady) return;
    _post({
      'action': 'DRAW_ROUTE',
      'data': {'encodedPolyline': polyline}
    });
  }

  void _post(Map<String, dynamic> msg) {
    _controller.postWebMessage(jsonEncode(msg));
  }

  @override
  void didUpdateWidget(covariant MappaTurnoWidget oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (_mapReady) {
      _sendMarkers();
      if (widget.encodedPolyline != oldWidget.encodedPolyline && widget.encodedPolyline != null) {
        _sendPolyline(widget.encodedPolyline!);
      }
    }
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (_errorMessage != null) {
      return Center(
        child: Padding(
          padding: const EdgeInsets.all(24.0),
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              const Icon(Icons.error_outline, color: Colors.amber, size: 48),
              const SizedBox(height: 12),
              Text(_errorMessage!, textAlign: TextAlign.center),
            ],
          ),
        ),
      );
    }

    if (!_isInitialized) {
      return const Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            CircularProgressIndicator(),
            SizedBox(height: 12),
            Text('Caricamento mappa in corso...'),
          ],
        ),
      );
    }

    return ClipRRect(
      borderRadius: BorderRadius.circular(8),
      child: Webview(_controller),
    );
  }
}
```

---

## 3.7 — Suddivisione Operativa: Sotto-Fase 3a e Sotto-Fase 3b

Per garantire uno sviluppo controllato e facilitare i test di accettazione, la Fase 3 è suddivisa in due blocchi sequenziali:

### Fase 3a: Geocoding, Impostazioni e Visualizzazione Mappa
- Estensione schema Drift v2 (coordinate su ordini, clienti, ristorante e tabella `maps_usage`).
- Integrazione `MapsService.geocodeAddress` con gestione asincrona trasparente in `POST /ordini`.
- Schermata Impostazioni: inserimento e mascheramento chiave Google Maps, pulsante test connettività, geocoding e conferma coordinate della sede del locale.
- Integrazione `MappaTurnoWidget` nel dettaglio turno con visualizzazione marker numerati (`sequenza + 1`), colori per stato e badge ritardo dinamico.

### Fase 3b: Giri di Consegna, Routes API e Ottimizzazione
- UI di selezione ordini per comporre il "Giro di Consegna" (subgroup di 3–6 ordini con filtro per orario simile <= 15 min).
- Integrazione `LocalTspSolver` (ordinamento esatto <= 8 tappe su CPU locale).
- Integrazione `computeGiroRoute` con **Google Routes API** (`POST computeRoutes` su SKU Essentials).
- Tracciamento della polyline del giro selezionato, visualizzazione ETA di ogni tappa e pulsante *"Applica sequenza alle consegne"*.

---

## Criteri di Accettazione

| ID | Modulo | Criterio di Accettazione |
|---|---|---|
| **AC-3.1** | Schema v2 | La migrazione o creazione pulita del DB v2 aggiunge coordinate a `clienti`, `ordini`, `ristoranti` e crea la tabella `maps_usage`. I dati preesistenti non subiscono perdite. |
| **AC-3.2** | Caching Indirizzo | Un secondo ordine con lo stesso indirizzo normalizzato riusa le coordinate salvate nella tabella `clienti` senza effettuare chiamate HTTP a Google. Se l'indirizzo varia, la cache non scatta e viene geocodificato il nuovo indirizzo. |
| **AC-3.3** | Geocoding Resiliente | In caso di indirizzo errato (`ZERO_RESULTS`) o assenza di rete, l'ordine viene comunque salvato in SQLite con `geocoding_stato = 'fallito'` e un'icona di avviso permette la riprova manuale dalla UI. |
| **AC-3.4** | Precisione Geocoding | Vengono marcati come `ok` solo i risultati con precisione `ROOFTOP` o `RANGE_INTERPOLATED` senza `partial_match`; gli indirizzi ambigui vengono contrassegnati come `approssimato`. |
| **AC-3.5** | Routes API | Il calcolo percorsi utilizza l'endpoint ufficiale `computeRoutes` di Routes API con header `X-Goog-FieldMask` e travelMode `DRIVE`. |
| **AC-3.6** | Solver TSP Locale | Fino a 8 tappe, la sequenza consigliata del giro viene determinata dal solver locale in meno di 50 ms senza attivare lo SKU a pagamento *Routes Preferred / Pro*. |
| **AC-3.7** | Calcolo ETA & Ritardi | L'ETA di ciascuna tappa somma i secondi di guida effettivi e la sosta configurata del locale (default 3 min). Se l'orario richiesto supera la mezzanotte, il calcolo aggiunge 1 giorno senza produrre ritardi fittizi. |
| **AC-3.8** | Sicurezza Chiave | La chiave Google Maps non compare mai in chiaro nei log, nelle eccezioni, né in file temporanei su disco. La WebView carica l'HTML direttamente in memoria RAM con `loadStringContent`. |
| **AC-3.9** | WebView Desktop | La mappa viene renderizzata stabilmente su Windows via WebView2, si centra sulla sede del ristorante e mostra i marker con numero sequenziale 1-based e colore associato allo stato. |
| **AC-3.10**| Interazione Mappa-Lista | Cliccando su un marker della mappa compare l'InfoWindow protetto da XSS ed evidenzia l'ordine nella lista laterale; viceversa cliccando sull'ordine la mappa centra la visuale. |
| **AC-3.11**| Tolleranza No Key / Offline | In assenza di chiave configurata o senza rete internet, la UI avvisa l'utente tramite banner informativo permettendo comunque tutte le funzioni di gestione ordini manuale. |

---

## Checklist di Testing

### 1. Test Geocoding e Cache
- [ ] Inserimento ordine con indirizzo reale italiano (es. *"Via Dante 10, Milano"*): verifica coordinate corrette salvate su SQLite e stato `ok`.
- [ ] Inserimento secondo ordine allo stesso indirizzo: verifica che la chiamata HTTP a Google non avvenga (cache hit).
- [ ] Inserimento ordine con indirizzo non censito: verifica stato `fallito`, ordine salvato e icona di riprova visibile.

### 2. Test Routes API e Giri di Consegna
- [ ] Creazione giro di 4 ordini: verifica chiamata corretta a `POST computeRoutes` con `X-Goog-FieldMask` e parsing della durata formato `"420s"`.
- [ ] Verifica calcolo ETA progressivo con sosta di 3 minuti inclusa per tappa.
- [ ] Test turno a cavallo della mezzanotte (es. turno ore 23:00 con ordine 00:15): verifica che l'ETA risulti 00:15 del giorno successivo e non in ritardo di 23 ore.
- [ ] Verifica ordinamento del giro: cliccando *"Applica sequenza consigliata"*, la colonna `sequenza` degli ordini nel DB si aggiorna rispettando l'itinerario.

### 3. Test Mappa WebView2 Desktop
- [ ] Verifica rendering mappa fluida centrata sulle coordinate della pizzeria.
- [ ] Verifica marker: numero 1-based (`#1, #2...`), colore coerente con lo stato e bordo rosso marcato se in ritardo.
- [ ] Verifica InfoWindow con caratteri speciali (es. `<script>`, virgolette, `&`): verifica assenza di iniezioni HTML grazie all'uso di `textContent`.
- [ ] Ridimensionamento finestra Windows e stabilità zoom (il focus non si azzera ad ogni aggiornamento della lista ordini).
