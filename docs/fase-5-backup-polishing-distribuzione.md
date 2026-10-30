# Fase 5 — Backup, Polishing e Distribuzione

La **Fase 5** rappresenta la fase di consolidamento, robustezza, rifinitura estetica e rilascio di **DelivMap**. Durante questa fase l'applicazione viene resa a prova di guasto attraverso procedure avanzate di backup e ripristino del database locale SQLite, un sistema di gestione centralizzata degli errori e del feedback visivo, un design system coerente basato su Material 3 e un installer pronto all'uso per Windows realizzato con Inno Setup.

- **Durata stimata**: 4-5 giorni lavorativi
- **Piattaforma target principale**: Windows Desktop (10 / 11) a 64 bit
- **Architettura di riferimento**: Flutter Desktop (Riverpod) → API Shelf Locale (`http://127.0.0.1:8080`) → Drift ORM (`delivmap.sqlite`)

---

## 5.1 — Backup Automatico del Database

Nell'architettura local-first di DelivMap, l'intero patrimonio informativo del ristorante (clienti, ordini, turni, configurazioni) risiede in un singolo file SQLite (`delivmap.sqlite`) salvato nella cartella `Application Support` del sistema operativo. La perdita o corruzione di questo file causerebbe il blocco operativo delle consegne; la strategia di salvaguardia deve quindi operare in modo autonomo, trasparente e resiliente.

### Architettura e Specifiche del Backup

1. **Ubicazione sorgente**: `%APPDATA%\com.delivmap\app\delivmap.sqlite`.
2. **Ubicazione destinazione predefinita**: `%APPDATA%\com.delivmap\app\backups\`, modificabile a piacere dal ristoratore (ad esempio su una cartella sincronizzata con OneDrive, Google Drive o su una chiavetta USB).
3. **Naming Convention**: `delivmap_backup_YYYYMMDD_HHmmss.sqlite` (es. `delivmap_backup_20261017_203000.sqlite`).
4. **Schedulazione periodica**: `Timer.periodic` configurabile (default: ogni 24 ore), che si attiva in background senza bloccare l'interfaccia utente né le chiamate API Shelf.
5. **Politica di Retention**: Mantenimento degli ultimi *N* backup (default: 7 copie). All'esecuzione di ogni nuovo backup, i file più vecchi che superano la soglia vengono eliminati automaticamente.
6. **Integrità prima della copia**: Esecuzione del checkpoint WAL (`PRAGMA wal_checkpoint(TRUNCATE)`) prima della copia del file binario, per garantire che tutte le transazioni temporanee pendenti nel file `-wal` vengano scritte sul file SQLite master.

### Implementazione: BackupService

Il file `lib/services/backup_service.dart` implementa l'intera logica di schedulazione, copia sicura, rotazione dei file e notifica:

```dart
import 'dart:async';
import 'dart:io';
import 'package:flutter/foundation.dart';
import 'package:intl/intl.dart';
import 'package:path/path.dart' as p;
import 'package:path_provider/path_provider.dart';
import 'package:drift/drift.dart';
import '../data/database.dart';

/// Metadati descrittivi di un file di backup individuato su disco.
class BackupInfo {
  final String fileName;
  final String filePath;
  final DateTime createdAt;
  final int sizeInBytes;

  BackupInfo({
    required this.fileName,
    required this.filePath,
    required this.createdAt,
    required this.sizeInBytes,
  });

  String get formattedSize {
    if (sizeInBytes < 1024) return '$sizeInBytes B';
    if (sizeInBytes < 1024 * 1024) {
      return '${(sizeInBytes / 1024).toStringAsFixed(1)} KB';
    }
    return '${(sizeInBytes / (1024 * 1024)).toStringAsFixed(2)} MB';
  }

  String get formattedDate =>
      DateFormat('dd/MM/yyyy HH:mm:ss').format(createdAt);
}

/// Servizio responsabile della creazione, pianificazione e rotazione
/// dei backup del database SQLite.
class BackupService extends ChangeNotifier {
  final AppDatabase _db;
  Timer? _periodicTimer;

  String? _customBackupDirectory;
  int _retentionCount = 7;
  int _intervalHours = 24;
  bool _isAutoBackupEnabled = true;
  bool _isBackupRunning = false;
  DateTime? _lastBackupTime;

  bool get isBackupRunning => _isBackupRunning;
  DateTime? get lastBackupTime => _lastBackupTime;
  int get retentionCount => _retentionCount;
  int get intervalHours => _intervalHours;
  bool get isAutoBackupEnabled => _isAutoBackupEnabled;

  static final RegExp _backupFileNameRegex = RegExp(
    r'^delivmap_backup_(\d{8})_(\d{6})\.sqlite$',
  );

  BackupService(this._db);

  /// Inizializza il servizio caricando le impostazioni e avviando il timer periodico.
  Future<void> initialize({
    String? customBackupDirectory,
    int retentionCount = 7,
    int intervalHours = 24,
    bool isAutoBackupEnabled = true,
  }) async {
    _customBackupDirectory = customBackupDirectory;
    _retentionCount = retentionCount;
    _intervalHours = intervalHours;
    _isAutoBackupEnabled = isAutoBackupEnabled;

    if (_isAutoBackupEnabled) {
      _avviaTimerPeriodico();
    }
  }

  /// Avvia il timer di backup periodico in background.
  void _avviaTimerPeriodico() {
    _periodicTimer?.cancel();
    _periodicTimer = Timer.periodic(
      Duration(hours: _intervalHours),
      (timer) async {
        debugPrint('[BackupService] Esecuzione backup automatico programmato...');
        await eseguiBackup(isManuale: false);
      },
    );
    debugPrint(
      '[BackupService] Timer backup avviato con frequenza: ogni $_intervalHours ore.',
    );
  }

  /// Restituisce la cartella in cui salvare i backup.
  Future<Directory> ottieniCartellaBackup() async {
    if (_customBackupDirectory != null &&
        _customBackupDirectory!.trim().isNotEmpty) {
      final customDir = Directory(_customBackupDirectory!);
      if (!await customDir.exists()) {
        await customDir.create(recursive: true);
      }
      return customDir;
    }

    final appSupportDir = await getApplicationSupportDirectory();
    final defaultBackupDir = Directory(p.join(appSupportDir.path, 'backups'));
    if (!await defaultBackupDir.exists()) {
      await defaultBackupDir.create(recursive: true);
    }
    return defaultBackupDir;
  }

  /// Restituisce il percorso fisico del file SQLite principale in uso.
  Future<File> ottieniFileDatabasePrincipale() async {
    final appSupportDir = await getApplicationSupportDirectory();
    return File(p.join(appSupportDir.path, 'delivmap.sqlite'));
  }

  /// Esegue la procedura di snapshot del database e applica la retention.
  Future<BackupInfo> eseguiBackup({bool isManuale = true}) async {
    if (_isBackupRunning) {
      throw StateError('Una procedura di backup è già in corso.');
    }

    _isBackupRunning = true;
    notifyListeners();

    try {
      final dbFile = await ottieniFileDatabasePrincipale();
      if (!await dbFile.exists()) {
        throw StateError(
          'File database principale non trovato in: ${dbFile.path}',
        );
      }

      // Forza il checkpoint WAL prima della copia del file, così da
      // riversare eventuali scritture nel file principale.
      try {
        await _db.customStatement('PRAGMA wal_checkpoint(TRUNCATE);');
      } catch (e) {
        debugPrint('[BackupService] Avviso WAL checkpoint: $e');
      }

      final backupDir = await ottieniCartellaBackup();
      final now = DateTime.now();
      final timestampStr = DateFormat('yyyyMMdd_HHmmss').format(now);
      final backupFileName = 'delivmap_backup_$timestampStr.sqlite';
      final backupFilePath = p.join(backupDir.path, backupFileName);

      // Copia binaria del file
      final backupFile = await dbFile.copy(backupFilePath);
      final size = await backupFile.length();

      _lastBackupTime = now;
      debugPrint('[BackupService] Backup completato: $backupFilePath ($size bytes)');

      // Pulizia automatica file obsoleti
      await _applicaRetention(backupDir);

      final info = BackupInfo(
        fileName: backupFileName,
        filePath: backupFilePath,
        createdAt: now,
        sizeInBytes: size,
      );

      return info;
    } finally {
      _isBackupRunning = false;
      notifyListeners();
    }
  }

  /// Mantiene solo gli ultimi [_retentionCount] backup eliminando i più datati.
  Future<void> _applicaRetention(Directory backupDir) async {
    try {
      final listaBackup = await elencaBackupDisponibili();
      if (listaBackup.length <= _retentionCount) {
        return;
      }

      // Ordina dal più recente al più vecchio
      listaBackup.sort((a, b) => b.createdAt.compareTo(a.createdAt));

      // Quelli oltre l'indice _retentionCount vengono cancellati
      final daEliminare = listaBackup.sublist(_retentionCount);
      for (final backup in daEliminare) {
        final f = File(backup.filePath);
        if (await f.exists()) {
          await f.delete();
          debugPrint(
            '[BackupService] Rimosso backup obsoleto per retention: ${backup.fileName}',
          );
        }
      }
    } catch (e) {
      debugPrint('[BackupService] Errore durante l\'applicazione della retention: $e');
    }
  }

  /// Restituisce la lista di tutti i backup validi presenti nella directory.
  Future<List<BackupInfo>> elencaBackupDisponibili() async {
    final dir = await ottieniCartellaBackup();
    final files = dir.listSync().whereType<File>();
    final List<BackupInfo> risultato = [];

    for (final file in files) {
      final name = p.basename(file.path);
      final match = _backupFileNameRegex.firstMatch(name);
      if (match != null) {
        final stat = file.statSync();
        // Estrazione data/ora direttamente dal nome file per precisione assoluta
        final datePart = match.group(1)!;
        final timePart = match.group(2)!;
        DateTime dataCreazione;
        try {
          dataCreazione = DateTime.parse(
            '${datePart.substring(0, 4)}-${datePart.substring(4, 6)}-${datePart.substring(6, 8)} '
            '${timePart.substring(0, 2)}:${timePart.substring(2, 4)}:${timePart.substring(4, 6)}',
          );
        } catch (_) {
          dataCreazione = stat.modified;
        }

        risultato.add(
          BackupInfo(
            fileName: name,
            filePath: file.path,
            createdAt: dataCreazione,
            sizeInBytes: stat.size,
          ),
        );
      }
    }

    risultato.sort((a, b) => b.createdAt.compareTo(a.createdAt));
    return risultato;
  }

  /// Aggiorna le impostazioni del servizio a runtime.
  Future<void> aggiornaConfigurazione({
    String? customBackupDirectory,
    int? retentionCount,
    int? intervalHours,
    bool? isAutoBackupEnabled,
  }) async {
    if (customBackupDirectory != null) {
      _customBackupDirectory = customBackupDirectory;
    }
    if (retentionCount != null && retentionCount > 0) {
      _retentionCount = retentionCount;
    }
    if (intervalHours != null && intervalHours > 0) {
      _intervalHours = intervalHours;
    }
    if (isAutoBackupEnabled != null) {
      _isAutoBackupEnabled = isAutoBackupEnabled;
      if (_isAutoBackupEnabled) {
        _avviaTimerPeriodico();
      } else {
        _periodicTimer?.cancel();
        _periodicTimer = null;
      }
    }
    notifyListeners();
  }

  @override
  void dispose() {
    _periodicTimer?.cancel();
    super.dispose();
  }
}
```

---

## 5.2 — Restore da Backup

La funzionalità di ripristino consente di riportare il sistema a un punto nel tempo precedente. Trattandosi di un'operazione distruttiva sui dati correnti e con vincoli ferrei legati al file locking di Windows su file SQLite aperti, il processo deve seguire una sequenza di sicurezza atomica.

### Sequenza Operativa del Restore

```
┌────────────────────────┐
│ 1. Richiesta Restore   │
│   (UI o File Esterno)  │
└───────────┬────────────┘
            │
            ▼
┌────────────────────────┐
│ 2. Dialog di Conferma  │  (Warning chiaro: perdita dati non salvati)
│    confermato?         ├────── No ───▶ Annulla
└───────────┬────────────┘
            │ Sì
            ▼
┌────────────────────────┐
│ 3. Snapshot Sicurezza  │  (Crea copia `delivmap_pre_restore.sqlite`)
└───────────┬────────────┘
            │
            ▼
┌────────────────────────┐
│ 4. Chiusura Connessione│  (Drift `database.close()` per liberare il lock)
└───────────┬────────────┘
            │
            ▼
┌────────────────────────┐
│ 5. Sovrascrittura File │  (Copia del file di backup su `delivmap.sqlite`)
└───────────┬────────────┘
            │
            ▼
┌────────────────────────┐
│ 6. Riapertura & Check  │  (Esecuzione `PRAGMA integrity_check`)
│    Integrità OK?       ├────── No ───▶ Ripristino Snapshot di Sicurezza
└───────────┬────────────┘               e Segnalazione Errore Critico
            │ Sì
            ▼
┌────────────────────────┐
│ 7. Ricaricamento Stato │  (Invalidazione provider Riverpod e reset UI)
└────────────────────────┘
```

### Implementazione: DatabaseManager & Restore Logic

Nel file `lib/data/database_manager.dart`, viene gestito il ciclo di vita della connessione SQLite per consentire la chiusura sicura, la sovrascrittura del file e la riapertura con test di integrità:

```dart
import 'dart:io';
import 'package:flutter/foundation.dart';
import 'package:path/path.dart' as p;
import 'package:path_provider/path_provider.dart';
import 'package:drift/drift.dart';
import 'package:drift/native.dart';
import 'database.dart';

/// Gestore del ciclo di vita del database, necessario per supportare
/// la chiusura della connessione, il restore da file e la verifica d'integrità.
class DatabaseManager extends ChangeNotifier {
  AppDatabase? _database;

  AppDatabase get db {
    _database ??= _creaIstanzaDatabase();
    return _database!;
  }

  AppDatabase _creaIstanzaDatabase() {
    return AppDatabase();
  }

  /// Esegue il ripristino del database a partire dal percorso del file indicato.
  Future<void> ripristinaBackup(String percorsoFileBackup) async {
    final sorgenteBackup = File(percorsoFileBackup);
    if (!await sorgenteBackup.exists()) {
      throw FileSystemException(
        'Il file di backup specificato non esiste',
        percorsoFileBackup,
      );
    }

    final appSupportDir = await getApplicationSupportDirectory();
    final dbPath = p.join(appSupportDir.path, 'delivmap.sqlite');
    final dbFileCorrente = File(dbPath);
    final snapshotSicurezza = File(
      p.join(appSupportDir.path, 'delivmap_pre_restore_safety.sqlite'),
    );

    // 1. Chiusura formale della connessione Drift per rilasciare l'handle SQLite
    if (_database != null) {
      debugPrint('[DatabaseManager] Chiusura connessione database in corso...');
      await _database!.close();
      _database = null;
    }

    // 2. Creazione copia di salvataggio pre-restore (in caso di fallimento)
    if (await dbFileCorrente.exists()) {
      await dbFileCorrente.copy(snapshotSicurezza.path);
      debugPrint(
        '[DatabaseManager] Snapshot di sicurezza creato in: ${snapshotSicurezza.path}',
      );
    }

    // 3. Sovrascrittura fisica del file di database principale
    try {
      await sorgenteBackup.copy(dbPath);
      debugPrint('[DatabaseManager] File di backup copiato su: $dbPath');

      // Elimina eventuali file ausiliari di WAL/SHM residui
      final walFile = File('$dbPath-wal');
      final shmFile = File('$dbPath-shm');
      if (await walFile.exists()) await walFile.delete();
      if (await shmFile.exists()) await shmFile.delete();

      // 4. Riapertura della connessione
      _database = _creaIstanzaDatabase();

      // 5. Verifica integrità strutturale SQLite
      final List<QueryRow> checkResult =
          await _database!.customSelect('PRAGMA integrity_check;').get();

      final String status =
          checkResult.isNotEmpty ? checkResult.first.data.values.first.toString() : '';

      if (status.toLowerCase() != 'ok') {
        throw FormatException(
          'Il database ripristinato non ha superato l\'integrity check: $status',
        );
      }

      debugPrint('[DatabaseManager] Restore completato con successo. Integrity check: OK');

      // Se tutto ha avuto successo, possiamo rimuovere lo snapshot temporaneo
      if (await snapshotSicurezza.exists()) {
        await snapshotSicurezza.delete();
      }

      notifyListeners();
    } catch (errore) {
      debugPrint('[DatabaseManager] Errore critico durante il restore: $errore');

      // Rollback di emergenza
      if (await snapshotSicurezza.exists()) {
        debugPrint('[DatabaseManager] Ripristino dello snapshot di sicurezza in corso...');
        await snapshotSicurezza.copy(dbPath);
        _database = _creaIstanzaDatabase();
      }
      rethrow;
    }
  }
}
```

### Sezione UI Impostazioni: Gestione e Ripristino Backup

Nel file `lib/ui/screens/settings/backup_settings_view.dart`, la sezione offre la panoramica dei backup e i pulsanti di azione con `file_picker`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:file_picker/file_picker.dart';
import '../../../services/backup_service.dart';
import '../../../ui/providers/app_providers.dart';

class BackupSettingsView extends ConsumerStatefulWidget {
  const BackupSettingsView({super.key});

  @override
  ConsumerState<BackupSettingsView> createState() => _BackupSettingsViewState();
}

class _BackupSettingsViewState extends ConsumerState<BackupSettingsView> {
  late Future<List<BackupInfo>> _backupListFuture;

  @override
  void initState() {
    super.initState();
    _ricaricaLista();
  }

  void _ricaricaLista() {
    final backupService = ref.read(backupServiceProvider);
    setState(() {
      _backupListFuture = backupService.elencaBackupDisponibili();
    });
  }

  Future<void> _selezionaCartellaBackup() async {
    final result = await FilePicker.platform.getDirectoryPath(
      dialogTitle: 'Seleziona cartella per i backup di DelivMap',
    );
    if (result != null) {
      final backupService = ref.read(backupServiceProvider);
      await backupService.aggiornaConfigurazione(customBackupDirectory: result);
      _ricaricaLista();
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Cartella backup aggiornata: $result')),
        );
      }
    }
  }

  Future<void> _eseguiBackupManuale() async {
    final backupService = ref.read(backupServiceProvider);
    try {
      final info = await backupService.eseguiBackup(isManuale: true);
      _ricaricaLista();
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(
            backgroundColor: Colors.green.shade700,
            content: Text('Backup completato con successo: ${info.fileName}'),
          ),
        );
      }
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(
            backgroundColor: Theme.of(context).colorScheme.error,
            content: Text('Errore durante il backup: $e'),
          ),
        );
      }
    }
  }

  Future<void> _richiediConfermaERipristina(String percorsoFile) async {
    final confermato = await showDialog<bool>(
      context: context,
      builder: (ctx) => AlertDialog(
        icon: const Icon(Icons.warning_amber_rounded, color: Colors.amber, size: 48),
        title: const Text('Attenzione: Ripristino Database'),
        content: const Text(
          'Il ripristino sostituirà integralmente tutti i dati correnti '
          '(clienti, ordini, turni e impostazioni) con quelli contenuti nel backup selezionato.\n\n'
          'Tutte le modifiche non presenti nel backup andranno perse in modo irreversibile.\n\n'
          'Sei sicuro di voler proseguire?',
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.of(ctx).pop(false),
            child: const Text('Annulla'),
          ),
          FilledButton(
            style: FilledButton.styleFrom(backgroundColor: Colors.red.shade700),
            onPressed: () => Navigator.of(ctx).pop(true),
            child: const Text('Sì, Sovrascrivi e Ripristina'),
          ),
        ],
      ),
    );

    if (confermato != true) return;

    try {
      final dbManager = ref.read(databaseManagerProvider);
      await dbManager.ripristinaBackup(percorsoFile);

      // Invalida lo stato globale per forzare il refresh delle schermate
      ref.invalidate(currentTurnoProvider);
      ref.invalidate(ordiniListProvider);

      if (mounted) {
        showDialog(
          context: context,
          barrierDismissible: false,
          builder: (ctx) => AlertDialog(
            title: const Text('Ripristino Completato'),
            content: const Text(
              'Il database è stato ripristinato con successo e la sua integrità è stata verificata.',
            ),
            actions: [
              FilledButton(
                onPressed: () {
                  Navigator.of(ctx).pop();
                  _ricaricaLista();
                },
                child: const Text('OK'),
              ),
            ],
          ),
        );
      }
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(
            backgroundColor: Theme.of(context).colorScheme.error,
            content: Text('Errore durante il ripristino: $e'),
          ),
        );
      }
    }
  }

  Future<void> _importaBackupEsterno() async {
    final result = await FilePicker.platform.pickFiles(
      dialogTitle: 'Seleziona file di backup SQLite da importare',
      type: FileType.custom,
      allowedExtensions: ['sqlite', 'db'],
    );

    if (result != null && result.files.single.path != null) {
      await _richiediConfermaERipristina(result.files.single.path!);
    }
  }

  @override
  Widget build(BuildContext context) {
    final backupService = ref.watch(backupServiceProvider);

    return Card(
      elevation: 0,
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(12),
        side: BorderSide(color: Theme.of(context).colorScheme.outlineVariant),
      ),
      child: Padding(
        padding: const EdgeInsets.all(20),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              children: [
                const Icon(Icons.backup_outlined, size: 28),
                const SizedBox(width: 12),
                Text(
                  'Backup e Ripristino Dati',
                  style: Theme.of(context).textTheme.titleLarge,
                ),
                const Spacer(),
                OutlinedButton.icon(
                  onPressed: _importaBackupEsterno,
                  icon: const Icon(Icons.file_open_outlined),
                  label: const Text('Ripristina da file esterno...'),
                ),
                const SizedBox(width: 8),
                FilledButton.icon(
                  onPressed:
                      backupService.isBackupRunning ? null : _eseguiBackupManuale,
                  icon: backupService.isBackupRunning
                      ? const SizedBox(
                          width: 16,
                          height: 16,
                          child: CircularProgressIndicator(strokeWidth: 2),
                        )
                      : const Icon(Icons.save_outlined),
                  label: const Text('Esegui Backup Adesso'),
                ),
              ],
            ),
            const Divider(height: 32),
            Row(
              children: [
                Expanded(
                  child: FutureBuilder<Directory>(
                    future: backupService.ottieniCartellaBackup(),
                    builder: (context, snapshot) {
                      final path = snapshot.data?.path ?? 'In caricamento...';
                      return ListTile(
                        contentPadding: EdgeInsets.zero,
                        title: const Text('Cartella di destinazione backup'),
                        subtitle: Text(path, style: const TextStyle(fontSize: 12)),
                        trailing: IconButton(
                          icon: const Icon(Icons.folder_open),
                          tooltip: 'Cambia cartella',
                          onPressed: _selezionaCartellaBackup,
                        ),
                      );
                    },
                  ),
                ),
              ],
            ),
            const SizedBox(height: 16),
            Text(
              'Backup Disponibili su Disco (ultimi ${backupService.retentionCount} conservati):',
              style: Theme.of(context).textTheme.titleSmall,
            ),
            const SizedBox(height: 8),
            Expanded(
              child: FutureBuilder<List<BackupInfo>>(
                future: _backupListFuture,
                builder: (context, snapshot) {
                  if (snapshot.connectionState == ConnectionState.waiting) {
                    return const Center(child: CircularProgressIndicator());
                  }
                  final backups = snapshot.data ?? [];
                  if (backups.isEmpty) {
                    return const Center(
                      child: Text('Nessun backup trovato nella cartella configurata.'),
                    );
                  }

                  return ListView.separated(
                    itemCount: backups.length,
                    separatorBuilder: (_, __) => const Divider(height: 1),
                    itemBuilder: (context, index) {
                      final item = backups[index];
                      return ListTile(
                        leading: const Icon(Icons.history_toggle_off),
                        title: Text(item.fileName),
                        subtitle: Text(
                          'Data: ${item.formattedDate} — Dimensione: ${item.formattedSize}',
                        ),
                        trailing: OutlinedButton.icon(
                          icon: const Icon(Icons.restore, size: 18),
                          label: const Text('Ripristina'),
                          onPressed: () =>
                              _richiediConfermaERipristina(item.filePath),
                        ),
                      );
                    },
                  );
                },
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## 5.3 — Export Dati in Formato CSV

Il ristoratore deve poter esportare lo storico degli ordini per finalità contabili, controllo degli incassi serali o analisi di marketing. L'export genera un file CSV compatibile direttamente con Microsoft Excel (italiano) tramite l'uso del separatore punto e virgola (`;`) e l'inclusione del Byte Order Mark UTF-8 (`\uFEFF`).

### Campi Tracciati nell'Export

| Campo CSV | Tipo Dato | Sorgente nel Database | Esempio |
|---|---|---|---|
| `Data_Turno` | Data (YYYY-MM-DD) | `turni.data` | `2026-10-17` |
| `ID_Ordine` | Intero | `ordini.id` | `1042` |
| `Nome_Cliente` | Testo | `clienti.nome` | `Mario Rossi` |
| `Telefono` | Testo | `clienti.telefono` | `+393331234567` |
| `Indirizzo` | Testo | `clienti.indirizzo` | `Via Roma 12, Milano` |
| `Importo_EUR` | Decimale (virgola) | `ordini.importo` | `24,50` |
| `Orario_Richiesto` | Orario (HH:mm) | `ordini.orario_richiesto` | `20:15` |
| `Stato` | Enum / Stringa | `ordini.stato` | `consegnato` |
| `Note` | Testo | `ordini.note` | `Citofono 4B, non suonare` |

### Implementazione: OrderExportService

Nel file `lib/services/order_export_service.dart`:

```dart
import 'dart:convert';
import 'dart:io';
import 'package:flutter/foundation.dart';
import 'package:intl/intl.dart';
import 'package:file_picker/file_picker.dart';
import '../data/database.dart';

class OrderExportRecord {
  final String dataTurno;
  final int ordineId;
  final String nomeCliente;
  final String telefonoCliente;
  final String indirizzoCliente;
  final double importo;
  final String orarioRichiesto;
  final String stato;
  final String note;

  OrderExportRecord({
    required this.dataTurno,
    required this.ordineId,
    required this.nomeCliente,
    required this.telefonoCliente,
    required this.indirizzoCliente,
    required this.importo,
    required this.orarioRichiesto,
    required this.stato,
    required this.note,
  });
}

class OrderExportService {
  final AppDatabase _db;

  OrderExportService(this._db);

  /// Recupera i record degli ordini compresi nell'intervallo specificato
  /// eseguendo il join tra ordini, clienti e turni.
  Future<List<OrderExportRecord>> ottieniOrdiniPerExport({
    required DateTime dataInizio,
    required DateTime dataFine,
  }) async {
    final inizioStr = DateFormat('yyyy-MM-dd').format(dataInizio);
    // Include l'intera giornata di fine
    final fineStr = DateFormat('yyyy-MM-dd').format(dataFine);

    final query = '''
      SELECT 
        t.data AS data_turno,
        o.id AS ordine_id,
        c.nome AS nome_cliente,
        c.telefono AS telefono_cliente,
        c.indirizzo AS indirizzo_cliente,
        o.importo AS importo,
        o.orario_richiesto AS orario_richiesto,
        o.stato AS stato,
        COALESCE(o.note, '') AS note
      FROM ordini o
      INNER JOIN clienti c ON o.cliente_id = c.id
      INNER JOIN turni t ON o.turno_id = t.id
      WHERE date(t.data) BETWEEN date(?) AND date(?)
      ORDER BY t.data ASC, o.sequenza ASC;
    ''';

    final rows = await _db.customSelect(
      query,
      variables: [Variable.withString(inizioStr), Variable.withString(fineStr)],
    ).get();

    return rows.map((row) {
      return OrderExportRecord(
        dataTurno: row.read<String>('data_turno'),
        ordineId: row.read<int>('ordine_id'),
        nomeCliente: row.read<String>('nome_cliente'),
        telefonoCliente: row.read<String>('telefono_cliente'),
        indirizzoCliente: row.read<String>('indirizzo_cliente'),
        importo: (row.read<num>('importo')).toDouble(),
        orarioRichiesto: row.read<String>('orario_richiesto'),
        stato: row.read<String>('stato'),
        note: row.read<String>('note'),
      );
    }).toList();
  }

  /// Converte la lista di record in una stringa CSV formattata per Excel italiano.
  String generaStringaCsv(List<OrderExportRecord> records) {
    final buffer = StringBuffer();
    // BOM UTF-8 per corretta visualizzazione di caratteri accentati in Excel Windows
    buffer.write('\uFEFF');

    // Intestazione
    buffer.writeln(
      'Data Turno;ID Ordine;Nome Cliente;Telefono;Indirizzo;Importo EUR;Orario Richiesto;Stato;Note',
    );

    for (final r in records) {
      final safeNome = _escapeCsv(r.nomeCliente);
      final safeTelefono = _escapeCsv(r.telefonoCliente);
      final safeIndirizzo = _escapeCsv(r.indirizzoCliente);
      final safeImporto = r.importo.toStringAsFixed(2).replaceAll('.', ',');
      final safeNote = _escapeCsv(r.note);

      buffer.writeln(
        '${r.dataTurno};${r.ordineId};$safeNome;$safeTelefono;$safeIndirizzo;$safeImporto;${r.orarioRichiesto};${r.stato};$safeNote',
      );
    }

    return buffer.toString();
  }

  String _escapeCsv(String val) {
    var v = val.replaceAll('"', '""');
    if (v.contains(';') || v.contains('\n') || v.contains('\r') || v.contains('"')) {
      return '"$v"';
    }
    return v;
  }

  /// Apre il dialog per il salvataggio del file ed esporta i dati.
  Future<String?> esportaCsvConDialog({
    required DateTime dataInizio,
    required DateTime dataFine,
  }) async {
    final records = await ottieniOrdiniPerExport(
      dataInizio: dataInizio,
      dataFine: dataFine,
    );

    if (records.isEmpty) {
      return null;
    }

    final csvContent = generaStringaCsv(records);
    final nomeDefault =
        'delivmap_export_${DateFormat('yyyyMMdd').format(dataInizio)}_${DateFormat('yyyyMMdd').format(dataFine)}.csv';

    final percorsoSalvataggio = await FilePicker.platform.saveFile(
      dialogTitle: 'Salva esportazione ordini CSV',
      fileName: nomeDefault,
      type: FileType.custom,
      allowedExtensions: ['csv'],
    );

    if (percorsoSalvataggio != null) {
      final file = File(percorsoSalvataggio);
      await file.writeAsString(csvContent, encoding: utf8);
      return percorsoSalvataggio;
    }

    return null;
  }
}
```

### Dialog UI di Esportazione

Nel file `lib/ui/widgets/dialogs/export_orders_dialog.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:intl/intl.dart';
import '../../../ui/providers/app_providers.dart';

class ExportOrdersDialog extends ConsumerStatefulWidget {
  const ExportOrdersDialog({super.key});

  @override
  ConsumerState<ExportOrdersDialog> createState() => _ExportOrdersDialogState();
}

class _ExportOrdersDialogState extends ConsumerState<ExportOrdersDialog> {
  DateTime _dataInizio = DateTime.now().subtract(const Duration(days: 7));
  DateTime _dataFine = DateTime.now();
  bool _isExporting = false;

  Future<void> _selezionaIntervalloDate() async {
    final picked = await showDateRangePicker(
      context: context,
      firstDate: DateTime(2024),
      lastDate: DateTime.now().add(const Duration(days: 365)),
      initialDateRange: DateTimeRange(start: _dataInizio, end: _dataFine),
      builder: (context, child) {
        return Center(
          child: ConstrainedBox(
            constraints: const BoxConstraints(maxWidth: 500, maxHeight: 600),
            child: child,
          ),
        );
      },
    );

    if (picked != null) {
      setState(() {
        _dataInizio = picked.start;
        _dataFine = picked.end;
      });
    }
  }

  Future<void> _confermaEsportazione() async {
    setState(() => _isExporting = true);
    final exportService = ref.read(orderExportServiceProvider);

    try {
      final filePath = await exportService.esportaCsvConDialog(
        dataInizio: _dataInizio,
        dataFine: _dataFine,
      );

      if (!mounted) return;

      if (filePath != null) {
        Navigator.of(context).pop();
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(
            backgroundColor: Colors.green.shade700,
            content: Text('File CSV salvato con successo:\n$filePath'),
          ),
        );
      } else {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(
            content: Text('Nessun ordine trovato nell\'intervallo selezionato.'),
          ),
        );
      }
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(
            backgroundColor: Theme.of(context).colorScheme.error,
            content: Text('Errore durante l\'esportazione: $e'),
          ),
        );
      }
    } finally {
      if (mounted) setState(() => _isExporting = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    final df = DateFormat('dd/MM/yyyy');

    return AlertDialog(
      title: const Row(
        children: [
          Icon(Icons.file_download_outlined),
          SizedBox(width: 8),
          Text('Esporta Ordini in CSV'),
        ],
      ),
      content: SizedBox(
        width: 440,
        child: Column(
          mainAxisSize: MainAxisSize.min,
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            const Text(
              'Seleziona il periodo temporale dei turni da includere nel report CSV (compatibile con Microsoft Excel).',
            ),
            const SizedBox(height: 16),
            InkWell(
              onTap: _selezionaIntervalloDate,
              borderRadius: BorderRadius.circular(8),
              child: Container(
                padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
                decoration: BoxDecoration(
                  border: Border.all(color: Theme.of(context).colorScheme.outline),
                  borderRadius: BorderRadius.circular(8),
                ),
                child: Row(
                  children: [
                    const Icon(Icons.date_range),
                    const SizedBox(width: 12),
                    Text(
                      'Dal ${df.format(_dataInizio)} al ${df.format(_dataFine)}',
                      style: const TextStyle(fontWeight: FontWeight.bold),
                    ),
                    const Spacer(),
                    const Text('Modifica', style: TextStyle(color: Colors.blue)),
                  ],
                ),
              ),
            ),
          ],
        ),
      ),
      actions: [
        TextButton(
          onPressed: _isExporting ? null : () => Navigator.of(context).pop(),
          child: const Text('Annulla'),
        ),
        FilledButton.icon(
          onPressed: _isExporting ? null : _confermaEsportazione,
          icon: _isExporting
              ? const SizedBox(
                  width: 16,
                  height: 16,
                  child: CircularProgressIndicator(strokeWidth: 2),
                )
              : const Icon(Icons.download),
          label: const Text('Esporta CSV...'),
        ),
      ],
    );
  }
}
```

---

## 5.4 — Gestione Errori e Feedback Utente

In un ambiente desktop di produzione, crash non intercettati o schermate rosse di debug Flutter minano radicalmente la fiducia dell'utente. È necessario un sistema di error handling a più livelli che catturi le anomalie alla radice, offra messaggi in lingua italiana contestualizzati ed elimini i tempi morti attraverso skeleton loader ed empty states parlanti.

### Tassonomia degli Errori dell'Applicazione

```dart
/// Classe base per tutte le eccezioni di dominio di DelivMap.
abstract class AppException implements Exception {
  final String messaggio;
  final String? dettagliTecnici;

  AppException(this.messaggio, [this.dettagliTecnici]);

  @override
  String toString() => dettagliTecnici == null
      ? messaggio
      : '$messaggio (Dettagli: $dettagliTecnici)';
}

class LocalServerUnreachableException extends AppException {
  LocalServerUnreachableException([String? details])
      : super(
          'Il server locale di DelivMap non risponde. Verifica che il processo sia attivo.',
          details,
        );
}

class MapsApiKeyInvalidException extends AppException {
  MapsApiKeyInvalidException([String? details])
      : super(
          'Chiave API Google Maps non valida o non configurata. Vai nelle Impostazioni per inserirne una valida.',
          details,
        );
}

class GeocodingAddressNotFoundException extends AppException {
  GeocodingAddressNotFoundException(String indirizzo, [String? details])
      : super(
          'Impossibile individuare le coordinate per l\'indirizzo inserito: "$indirizzo". Verifica via, civico e CAP.',
          details,
        );
}

class DatabaseCorruptedException extends AppException {
  DatabaseCorruptedException([String? details])
      : super(
          'Il file di database locale risulta inaccessibile o danneggiato. Esegui un ripristino dall\'ultimo backup.',
          details,
        );
}

class NetworkOfflineException extends AppException {
  NetworkOfflineException([String? details])
      : super(
          'Connessione Internet assente. Le funzioni di mappa e geolocalizzazione sono temporaneamente limitate.',
          details,
        );
}
```

### Wrapper Centralizzato Chiamate HTTP (ApiClient)

Tutte le interazioni tra UI e il server Dart Shelf su `localhost:8080` avvengono tramite `ApiClient`, che incapsula la gestione del timeout, i codici di errore HTTP e le interruzioni di rete:

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';
import 'package:http/http.dart' as http;

class ApiClient {
  final String baseUrl;
  final http.Client _client;

  ApiClient({
    this.baseUrl = 'http://127.0.0.1:8080',
    http.Client? client,
  }) : _client = client ?? http.Client();

  Future<dynamic> get(String endpoint, {Map<String, String>? headers}) async {
    return _sendWithCatch(() => _client.get(
          Uri.parse('$baseUrl$endpoint'),
          headers: _buildHeaders(headers),
        ));
  }

  Future<dynamic> post(
    String endpoint, {
    Map<String, dynamic>? body,
    Map<String, String>? headers,
  }) async {
    return _sendWithCatch(() => _client.post(
          Uri.parse('$baseUrl$endpoint'),
          headers: _buildHeaders(headers),
          body: body != null ? jsonEncode(body) : null,
        ));
  }

  Future<dynamic> put(
    String endpoint, {
    Map<String, dynamic>? body,
    Map<String, String>? headers,
  }) async {
    return _sendWithCatch(() => _client.put(
          Uri.parse('$baseUrl$endpoint'),
          headers: _buildHeaders(headers),
          body: body != null ? jsonEncode(body) : null,
        ));
  }

  Future<dynamic> delete(String endpoint, {Map<String, String>? headers}) async {
    return _sendWithCatch(() => _client.delete(
          Uri.parse('$baseUrl$endpoint'),
          headers: _buildHeaders(headers),
        ));
  }

  Map<String, String> _buildHeaders(Map<String, String>? extra) {
    return {
      'Content-Type': 'application/json; charset=utf-8',
      'Accept': 'application/json',
      ...?extra,
    };
  }

  Future<dynamic> _sendWithCatch(Future<http.Response> Function() call) async {
    try {
      final response = await call().timeout(const Duration(seconds: 10));

      if (response.statusCode >= 200 && response.statusCode < 300) {
        if (response.body.isEmpty) return null;
        return jsonDecode(utf8.decode(response.bodyBytes));
      }

      // Analisi risposte di errore dal server Shelf
      final dynamic errorBody = response.body.isNotEmpty
          ? jsonDecode(utf8.decode(response.bodyBytes))
          : null;
      final serverMessage =
          errorBody is Map ? errorBody['error']?.toString() : null;

      if (response.statusCode == 401 || response.statusCode == 403) {
        throw MapsApiKeyInvalidException(serverMessage);
      } else if (response.statusCode == 404) {
        throw GeocodingAddressNotFoundException(
          'Elemento non trovato',
          serverMessage,
        );
      } else if (response.statusCode >= 500) {
        throw AppException('Errore interno del server locale', serverMessage);
      } else {
        throw AppException(
          serverMessage ?? 'Richiesta non valida (${response.statusCode})',
        );
      }
    } on SocketException catch (e) {
      throw LocalServerUnreachableException(e.message);
    } on TimeoutException {
      throw LocalServerUnreachableException('Timeout connessione server locale');
    } on http.ClientException catch (e) {
      throw NetworkOfflineException(e.message);
    }
  }
}
```

### Feedback Visivo: Snackbar, Error Screen ed Empty State

#### 1. Banner & Snackbar di Notifica
Per segnalare errori non bloccanti o successi:

```dart
import 'package:flutter/material.dart';

class DelivFeedback {
  static void mostraErrore(BuildContext context, String messaggio) {
    ScaffoldMessenger.of(context).removeCurrentSnackBar();
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        backgroundColor: Theme.of(context).colorScheme.error,
        behavior: SnackBarBehavior.floating,
        margin: const EdgeInsets.all(16),
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(8)),
        content: Row(
          children: [
            const Icon(Icons.error_outline, color: Colors.white),
            const SizedBox(width: 12),
            Expanded(
              child: Text(
                messaggio,
                style: const TextStyle(color: Colors.white, fontWeight: FontWeight.w500),
              ),
            ),
          ],
        ),
      ),
    );
  }

  static void mostraSuccesso(BuildContext context, String messaggio) {
    ScaffoldMessenger.of(context).removeCurrentSnackBar();
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        backgroundColor: Colors.green.shade700,
        behavior: SnackBarBehavior.floating,
        margin: const EdgeInsets.all(16),
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(8)),
        content: Row(
          children: [
            const Icon(Icons.check_circle_outline, color: Colors.white),
            const SizedBox(width: 12),
            Expanded(
              child: Text(
                messaggio,
                style: const TextStyle(color: Colors.white, fontWeight: FontWeight.w500),
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

#### 2. Widget Schermata di Errore Critica (`CriticalErrorScreen`)
Quando il database o il server locale falliscono all'avvio:

```dart
import 'package:flutter/material.dart';

class CriticalErrorScreen extends StatelessWidget {
  final String titolo;
  final String messaggio;
  final VoidCallback onRiprova;
  final VoidCallback? onRipristinaBackup;

  const CriticalErrorScreen({
    super.key,
    required this.titolo,
    required this.messaggio,
    required this.onRiprova,
    this.onRipristinaBackup,
  });

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: ConstrainedBox(
          constraints: const BoxConstraints(maxWidth: 520),
          child: Padding(
            padding: const EdgeInsets.all(32),
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                Container(
                  padding: const EdgeInsets.all(20),
                  decoration: BoxDecoration(
                    color: Colors.red.shade50,
                    shape: BoxShape.circle,
                  ),
                  child: Icon(Icons.gpp_bad, size: 72, color: Colors.red.shade700),
                ),
                const SizedBox(height: 24),
                Text(
                  titolo,
                  textAlign: TextAlign.center,
                  style: Theme.of(context).textTheme.headlineSmall?.copyWith(
                        fontWeight: FontWeight.bold,
                      ),
                ),
                const SizedBox(height: 12),
                Text(
                  messaggio,
                  textAlign: TextAlign.center,
                  style: Theme.of(context).textTheme.bodyMedium?.copyWith(
                        color: Colors.grey.shade700,
                      ),
                ),
                const SizedBox(height: 32),
                Row(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    if (onRipristinaBackup != null) ...[
                      OutlinedButton.icon(
                        onPressed: onRipristinaBackup,
                        icon: const Icon(Icons.restore),
                        label: const Text('Ripristina Backup...'),
                      ),
                      const SizedBox(width: 16),
                    ],
                    FilledButton.icon(
                      onPressed: onRiprova,
                      icon: const Icon(Icons.refresh),
                      label: const Text('Riprova Connessione'),
                    ),
                  ],
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

#### 3. Empty State e Shimmer Loader Riutilizzabili

```dart
import 'package:flutter/material.dart';

class EmptyStateWidget extends StatelessWidget {
  final IconData icon;
  final String title;
  final String description;
  final String? actionLabel;
  final VoidCallback? onAction;

  const EmptyStateWidget({
    super.key,
    required this.icon,
    required this.title,
    required this.description,
    this.actionLabel,
    this.onAction,
  });

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(icon, size: 64, color: Theme.of(context).colorScheme.outline),
            const SizedBox(height: 16),
            Text(
              title,
              style: Theme.of(context).textTheme.titleMedium?.copyWith(
                    fontWeight: FontWeight.bold,
                  ),
              textAlign: TextAlign.center,
            ),
            const SizedBox(height: 8),
            Text(
              description,
              style: Theme.of(context).textTheme.bodySmall?.copyWith(
                    color: Theme.of(context).colorScheme.onSurfaceVariant,
                  ),
              textAlign: TextAlign.center,
            ),
            if (actionLabel != null && onAction != null) ...[
              const SizedBox(height: 16),
              FilledButton.tonal(
                onPressed: onAction,
                child: Text(actionLabel!),
              ),
            ],
          ],
        ),
      ),
    );
  }
}

/// Shimmer Skeleton per la lista degli ordini in fase di caricamento iniziale.
class OrderListSkeleton extends StatefulWidget {
  const OrderListSkeleton({super.key});

  @override
  State<OrderListSkeleton> createState() => _OrderListSkeletonState();
}

class _OrderListSkeletonState extends State<OrderListSkeleton>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 1200),
    )..repeat(reverse: true);
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _controller,
      builder: (context, child) {
        final opacity = 0.3 + (_controller.value * 0.4);
        return ListView.builder(
          itemCount: 6,
          padding: const EdgeInsets.all(12),
          itemBuilder: (context, index) {
            return Card(
              margin: const EdgeInsets.only(bottom: 12),
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: Row(
                  children: [
                    Container(
                      width: 32,
                      height: 32,
                      decoration: BoxDecoration(
                        color: Colors.grey.withOpacity(opacity),
                        shape: BoxShape.circle,
                      ),
                    ),
                    const SizedBox(width: 16),
                    Expanded(
                      child: Column(
                        crossAxisAlignment: CrossAxisAlignment.start,
                        children: [
                          Container(
                            height: 16,
                            width: 160,
                            color: Colors.grey.withOpacity(opacity),
                          ),
                          const SizedBox(height: 8),
                          Container(
                            height: 12,
                            width: 240,
                            color: Colors.grey.withOpacity(opacity),
                          ),
                        ],
                      ),
                    ),
                    Container(
                      height: 20,
                      width: 60,
                      color: Colors.grey.withOpacity(opacity),
                    ),
                  ],
                ),
              ),
            );
          },
        );
      },
    );
  }
}
```

---

## 5.5 — Polish UI e Design System

L'applicazione desktop deve trasmettere immediatamente l'identità del food delivery di qualità: professionale, energica, leggibile anche su monitor ad alto contrasto o in ambienti poco illuminati (cucina / cassa).

### Palette Cromatica e Material 3 ThemeData

- **Colore Primario**: `Color(0xFFE65100)` (Amber / Arancio bruciato — tipico del food e visibile a colpo d'occhio).
- **Colore Secondario**: `Color(0xFF00695C)` (Verde petrolio — per bilanciare l'energia dell'arancio con affidabilità e calma gestionale).
- **Stati Ordine Codificati**:
  - `in_attesa`: Blu indaco (`#1976D2`)
  - `in_consegna`: Arancione vivido (`#FB8C00`)
  - `consegnato`: Verde smeraldo (`#2E7D32`)
  - `in_ritardo`: Rosso corallo pulsante (`#D32F2F`)

Nel file `lib/ui/theme/app_theme.dart`:

```dart
import 'package:flutter/material.dart';

class DelivColors {
  // Brand Food Delivery
  static const Color primary = Color(0xFFE65100);
  static const Color secondary = Color(0xFF00695C);

  // Stati Ordine
  static const Color statoInAttesa = Color(0xFF1976D2);
  static const Color statoInConsegna = Color(0xFFFB8C00);
  static const Color statoConsegnato = Color(0xFF2E7D32);
  static const Color statoInRitardo = Color(0xFFD32F2F);
}

class AppTheme {
  static ThemeData lightTheme() {
    final colorScheme = ColorScheme.fromSeed(
      seedColor: DelivColors.primary,
      secondary: DelivColors.secondary,
      brightness: Brightness.light,
      surface: const Color(0xFFFBFBFB),
    );

    return ThemeData(
      useMaterial3: true,
      colorScheme: colorScheme,
      fontFamily: 'Segoe UI',
      cardTheme: CardTheme(
        elevation: 0,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(12),
          side: BorderSide(color: colorScheme.outlineVariant.withOpacity(0.6)),
        ),
      ),
      dialogTheme: DialogTheme(
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
      ),
      inputDecorationTheme: InputDecorationTheme(
        filled: true,
        fillColor: Colors.white,
        border: OutlineInputBorder(
          borderRadius: BorderRadius.circular(10),
          borderSide: BorderSide(color: colorScheme.outline),
        ),
        enabledBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(10),
          borderSide: BorderSide(color: colorScheme.outlineVariant),
        ),
        focusedBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(10),
          borderSide: BorderSide(color: colorScheme.primary, width: 2),
        ),
      ),
      pageTransitionsTheme: const PageTransitionsTheme(
        builders: {
          TargetPlatform.windows: FadeUpwardsPageTransitionsBuilder(),
          TargetPlatform.linux: FadeUpwardsPageTransitionsBuilder(),
          TargetPlatform.macOS: CupertinoPageTransitionsBuilder(),
        },
      ),
    );
  }

  static ThemeData darkTheme() {
    final colorScheme = ColorScheme.fromSeed(
      seedColor: DelivColors.primary,
      secondary: DelivColors.secondary,
      brightness: Brightness.dark,
      surface: const Color(0xFF1E1E1E),
    );

    return ThemeData(
      useMaterial3: true,
      colorScheme: colorScheme,
      fontFamily: 'Segoe UI',
      scaffoldBackgroundColor: const Color(0xFF121212),
      cardTheme: CardTheme(
        elevation: 0,
        color: const Color(0xFF1E1E1E),
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(12),
          side: BorderSide(color: colorScheme.outlineVariant.withOpacity(0.3)),
        ),
      ),
      dialogTheme: DialogTheme(
        backgroundColor: const Color(0xFF242424),
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
      ),
      inputDecorationTheme: InputDecorationTheme(
        filled: true,
        fillColor: const Color(0xFF2A2A2A),
        border: OutlineInputBorder(
          borderRadius: BorderRadius.circular(10),
          borderSide: BorderSide(color: colorScheme.outline),
        ),
        enabledBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(10),
          borderSide: BorderSide(color: colorScheme.outlineVariant.withOpacity(0.4)),
        ),
        focusedBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(10),
          borderSide: BorderSide(color: colorScheme.primary, width: 2),
        ),
      ),
    );
  }
}
```

### Micro-interazioni e Animazioni di Stato

Badge animato con pulsazione visiva se l'ordine è `in_ritardo`:

```dart
import 'package:flutter/material.dart';
import '../theme/app_theme.dart';

class OrderStatusBadge extends StatefulWidget {
  final String stato; // in_attesa, in_consegna, consegnato, in_ritardo

  const OrderStatusBadge({super.key, required this.stato});

  @override
  State<OrderStatusBadge> createState() => _OrderStatusBadgeState();
}

class _OrderStatusBadgeState extends State<OrderStatusBadge>
    with SingleTickerProviderStateMixin {
  AnimationController? _pulseController;
  Animation<double>? _pulseAnimation;

  @override
  void initState() {
    super.initState();
    if (widget.stato == 'in_ritardo') {
      _initPulse();
    }
  }

  @override
  void didUpdateWidget(covariant OrderStatusBadge oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (widget.stato == 'in_ritardo' && _pulseController == null) {
      _initPulse();
    } else if (widget.stato != 'in_ritardo' && _pulseController != null) {
      _pulseController!.dispose();
      _pulseController = null;
    }
  }

  void _initPulse() {
    _pulseController = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 900),
    )..repeat(reverse: true);
    _pulseAnimation = Tween<double>(begin: 0.85, end: 1.0).animate(
      CurvedAnimation(parent: _pulseController!, curve: Curves.easeInOut),
    );
  }

  @override
  void dispose() {
    _pulseController?.dispose();
    super.dispose();
  }

  Color _colorePerStato() {
    switch (widget.stato) {
      case 'in_consegna':
        return DelivColors.statoInConsegna;
      case 'consegnato':
        return DelivColors.statoConsegnato;
      case 'in_ritardo':
        return DelivColors.statoInRitardo;
      case 'in_attesa':
      default:
        return DelivColors.statoInAttesa;
    }
  }

  String _etichettaPerStato() {
    switch (widget.stato) {
      case 'in_consegna':
        return 'In Consegna';
      case 'consegnato':
        return 'Consegnato';
      case 'in_ritardo':
        return 'In Ritardo';
      case 'in_attesa':
      default:
        return 'In Attesa';
    }
  }

  @override
  Widget build(BuildContext context) {
    final color = _colorePerStato();
    final label = _etichettaPerStato();

    Widget badgeContent = AnimatedContainer(
      duration: const Duration(milliseconds: 300),
      padding: const EdgeInsets.symmetric(horizontal: 10, vertical: 4),
      decoration: BoxDecoration(
        color: color.withOpacity(0.15),
        border: Border.all(color: color, width: 1.2),
        borderRadius: BorderRadius.circular(20),
      ),
      child: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          Container(
            width: 8,
            height: 8,
            decoration: BoxDecoration(color: color, shape: BoxShape.circle),
          ),
          const SizedBox(width: 6),
          Text(
            label,
            style: TextStyle(
              color: color,
              fontWeight: FontWeight.bold,
              fontSize: 12,
            ),
          ),
        ],
      ),
    );

    if (_pulseAnimation != null) {
      return ScaleTransition(scale: _pulseAnimation!, child: badgeContent);
    }

    return badgeContent;
  }
}
```

### Gestione Dimensioni Finestra e Responsività Desktop

All'avvio in `main.dart`, si imposta la dimensione minima della finestra (min 800x600) tramite `window_manager`:

```dart
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:window_manager/window_manager.dart';

Future<void> configuraFinestraDesktop() async {
  if (Platform.isWindows || Platform.isLinux || Platform.isMacOS) {
    await windowManager.ensureInitialized();
    const windowOptions = WindowOptions(
      size: Size(1280, 800),
      minimumSize: Size(800, 600),
      center: true,
      backgroundColor: Colors.transparent,
      skipTaskbar: false,
      title: 'DelivMap — Gestione Consegne Locale',
    );
    await windowManager.waitUntilReadyToShow(windowOptions, () async {
      await windowManager.show();
      await windowManager.focus();
    });
  }
}
```

---

## 5.6 — Build e Distribuzione Windows

La distribuzione verso il ristorante prevede la creazione di un programma d'installazione autonomo (.exe) tramite **Inno Setup**, che installa i binari compilati di Flutter, le dipendenze C++ native e configura collegamenti e disinstallatore pulito.

### 1. Compilazione Release di Flutter

Eseguire il comando di compilazione release per l'architettura Windows a 64 bit:

```powershell
flutter clean
flutter pub get
dart run build_runner build --delete-conflicting-outputs
flutter build windows --release
```

#### Struttura della Cartella di Output (`build/windows/runner/Release/`)

```
build/windows/runner/Release/
│
├── delivmap.exe                 <-- Eseguibile compilato nativo Flutter
├── flutter_windows.dll          <-- Engine Flutter per Windows
├── sqlite3.dll                  <-- Libreria C++ nativa per Drift / SQLite
├── screen_retriever_plugin.dll  <-- Plugin ausiliari window_manager
├── window_manager_plugin.dll    
│
└── data/
    ├── app.so / flutter_assets/ <-- Bundle degli asset, icone e font
    └── icudtl.dat               <-- Dati Unicode/i18n
```

### 2. Script Inno Setup Completo (`installer/delivmap_setup.iss`)

Di seguito lo script completo e commentato per generare `DelivMap_Setup_v1.0.0.exe`:

```pascal
; =====================================================================
; DelivMap - Inno Setup Script
; Piattaforma: Windows x64
; Versione: 1.0.0
; =====================================================================

#define MyAppName "DelivMap"
#define MyAppVersion "1.0.0"
#define MyAppPublisher "DelivMap Team"
#define MyAppURL "https://github.com/tuo-progetto/delivmap"
#define MyAppExeName "delivmap.exe"
#define MyAppSourceDir "..\build\windows\runner\Release"

[Setup]
; Identificativo univoco dell'applicazione generato per prevenire collisioni
AppId={{94F6E35C-6284-4869-906F-8561CD054B72}
AppName={#MyAppName}
AppVersion={#MyAppVersion}
AppPublisher={#MyAppPublisher}
AppPublisherURL={#MyAppURL}
AppSupportURL={#MyAppURL}
AppUpdatesURL={#MyAppURL}
DefaultDirName={autopf}\{#MyAppName}
DisableProgramGroupPage=yes
; Richiede privilegi utente normale (o amministratore se installa in Program Files)
PrivilegesRequired=lowest
PrivilegesRequiredOverridesAllowed=dialog
OutputDir=..\dist
OutputBaseFilename=DelivMap_Setup_v{#MyAppVersion}
SetupIconFile=assets\app_icon.ico
Compression=lzma2/ultra64
SolidCompression=yes
WizardStyle=modern
ArchitecturesInstallIn64BitMode=x64compatible
ArchitecturesAllowed=x64compatible
UninstallDisplayIcon={app}\{#MyAppExeName}

[Languages]
Name: "italian"; MessagesFile: "compiler:Languages\Italian.isl"

[Tasks]
Name: "desktopicon"; Description: "{cm:CreateDesktopIcon}"; GroupDescription: "{cm:AdditionalIcons}"; Flags: unchecked
Name: "quicklaunchicon"; Description: "{cm:CreateQuickLaunchIcon}"; GroupDescription: "{cm:AdditionalIcons}"; Flags: unchecked; OnlyBelowVersion: 6.1; Check: not IsAdminInstallMode

[Files]
; Eseguibile principale e DLL
Source: "{#MyAppSourceDir}\{#MyAppExeName}"; DestDir: "{app}"; Flags: ignoreversion
Source: "{#MyAppSourceDir}\*.dll"; DestDir: "{app}"; Flags: ignoreversion
; Cartella dati e asset di runtime Flutter
Source: "{#MyAppSourceDir}\data\*"; DestDir: "{app}\data"; Flags: ignoreversion recursesubdirs createallsubdirs

[Icons]
Name: "{autoprograms}\{#MyAppName}"; Filename: "{app}\{#MyAppExeName}"
Name: "{autodesktop}\{#MyAppName}"; Filename: "{app}\{#MyAppExeName}"; Tasks: desktopicon

[Registry]
; Registrazione associazione file .sqlite / .db con DelivMap (facoltativo per restore con doppio clic)
Root: HKA; Subkey: "Software\Classes\.delivbackup"; ValueType: string; ValueName: ""; ValueData: "DelivMapBackupFile"; Flags: uninsdeletevalue
Root: HKA; Subkey: "Software\Classes\DelivMapBackupFile"; ValueType: string; ValueName: ""; ValueData: "DelivMap Database Backup"; Flags: uninsdeletekey
Root: HKA; Subkey: "Software\Classes\DelivMapBackupFile\DefaultIcon"; ValueType: string; ValueName: ""; ValueData: "{app}\{#MyAppExeName},0"
Root: HKA; Subkey: "Software\Classes\DelivMapBackupFile\shell\open\command"; ValueType: string; ValueName: ""; ValueData: """{app}\{#MyAppExeName}"" ""%1"""

[Run]
Filename: "{app}\{#MyAppExeName}"; Description: "{cm:LaunchProgram,{#StringChange(MyAppName, '&', '&&')}}"; Flags: nowait postinstall skipifsilent

[UninstallDelete]
; Nota: I dati del database in %APPDATA%\com.delivmap NON vengono rimossi per default
; per impedire cancellazioni accidentali dell'archivio ordini durante aggiornamenti.
Type: files; Name: "{app}\*.log"

[Code]
// Procedura di verifica pre-installazione: accerta che istanze precedenti siano chiuse
function InitializeSetup(): Boolean;
var
  ErrorCode: Integer;
begin
  Result := True;
  // È possibile verificare se delivmap.exe è in esecuzione con FindWindow o tasklist
end;
```

### 3. Versioning dell'Applicazione

Il numero di versione segue lo standard del Semantic Versioning (`MAJOR.MINOR.PATCH+BUILD`) definito in `pubspec.yaml`:

```yaml
name: delivmap
description: "Gestione locale consegne per piccoli ristoranti"
version: 1.0.0+1
```

- Nel file C++ `windows/runner/Runner.rc`, il numero di versione rispecchia quello dell'eseguibile Windows visibile nelle proprietà del file (`File Description`, `Product Version`).
- Durante la pipeline CI/CD o lo script di packaging PowerShell, la versione di Inno Setup viene iniettata tramite il parametro `/DMyAppVersion=1.0.0`.

### 4. Note sul Code Signing (Windows SmartScreen)

- Negli ambienti Windows 10/11, gli eseguibili privi di firma digitale generano l'avviso blu di Microsoft Defender SmartScreen (*"PC protetto da Windows — Microsoft Defender SmartScreen ha impedito l'avvio di un'app non riconosciuta"*).
- **Per la release MVP v1**: istruire il ristoratore a cliccare su *"Ulteriori informazioni"* e poi *"Esegui comunque"*.
- **Per le release commerciali ufficiali**: l'installer viene firmato con un certificato Authenticode (standard OV o EV) tramite `signtool.exe`:
  ```powershell
  signtool sign /tr http://timestamp.digicert.com /td sha256 /fd sha256 /f "MioCertificato.pfx" /p "PasswordCertificato" "dist\DelivMap_Setup_v1.0.0.exe"
  ```

---

## 5.7 — Testing End-to-End Manuale

Un protocollo di test operativo che simula esattamente una serata di lavoro reale all'interno di una pizzeria o ristorante d'asporto.

### Scenario di Test: Serata Operativa Tipo

```
[1. Primo Avvio] ──▶ [2. Login] ──▶ [3. Crea Turno] ──▶ [4. Inserisci 10 Ordini]
                                                                │
[8. Esegui Consegne] ◀── [7. Calcolo Percorso] ◀── [6. Mappa] ◀── [5. Drag & Drop]
        │
        ▼
[9. Test Ritardo] ──▶ [10. Backup Manuale] ──▶ [11. Logout & Chiudi] ──▶ [12. Riapri & Verifica]
```

#### Fasi dello Scenario di Test

| Passo | Azione Operativa | Comportamento Atteso | Esito (P/F) |
|---|---|---|---|
| **1. Primo Avvio** | Avviare `delivmap.exe` su un sistema pulito (senza DB esistente). | Compare la schermata di benvenuto/wizard; viene generato il file `delivmap.sqlite`. Richiesta di inserimento nome locale, indirizzo e API Key Google Maps. | [ ] |
| **2. Login** | Inserire credenziali create nel wizard. | Accesso consentito; transizione fluida verso la Dashboard principale. | [ ] |
| **3. Creazione Turno** | Cliccare su "Nuovo Turno", impostare data odierna e nota *"Venerdì Sera - Turno 1"*. | Il turno viene salvato in SQLite; la vista passa automaticamente alla griglia del turno corrente. | [ ] |
| **4. Inserimento 10 Ordini** | Inserire 10 ordini con mix di clienti:<br>• 5 clienti nuovi (con geocoding indirizzo reale)<br>• 5 clienti esistenti (tramite ricerca rapida per numero di telefono). | Per i nuovi clienti il geocoding restituisce coordinate valide; per i clienti esistenti l'indirizzo si autocompila istantaneamente. | [ ] |
| **5. Riordino Drag & Drop** | Cambiare l'ordine di 3 consegne trascinando le righe nella lista. | Il campo `sequenza` degli ordini si aggiorna a DB; i numeri d'ordine sulla lista riflettono la nuova sequenza. | [ ] |
| **6. Visualizzazione Mappa** | Aprire la vista Mappa del turno. | I marker compaiono correttamente centrati sul locale con numeri corrispondenti alla sequenza (1..10). | [ ] |
| **7. Calcolo Percorso** | Cliccare sul pulsante "Calcola Tempi Percorso". | Viene disegnata la polyline sequenziale; compaiono i tempi stimati di percorrenza per tratta e totale. | [ ] |
| **8. Avanzamento Stati** | Impostare i primi 3 ordini su `in_consegna`, poi su `consegnato`. | Il badge di stato cambia colore (blu → arancio → verde); i marker sulla mappa aggiornano il loro colore. | [ ] |
| **9. Forzatura Ritardo** | Modificare l'orario di un ordine con orario già trascorso di 20 minuti senza stato `consegnato`. | L'ordine entra automaticamente in stato `in_ritardo`, compare il badge rosso pulsante di alert. | [ ] |
| **10. Backup Manuale** | Andare in Impostazioni → Backup e cliccare "Esegui Backup Adesso". | File `delivmap_backup_YYYYMMDD_HHmmss.sqlite` creato nella cartella configurata; compare la riga nella lista. | [ ] |
| **11. Logout e Chiusura** | Effettuare il logout ed uscire dall'applicazione (chiusura finestra con `Alt+F4` o croce in alto). | Il processo Shelf e Flutter terminano correttamente senza rimanere appesi in Gestione Attività. | [ ] |
| **12. Riapertura & Persistenza** | Riaprire `delivmap.exe` ed effettuare il login. | Tutti i 10 ordini, le sequenze modificate, lo stato `consegnato` e il turno risultano integri e coerenti. | [ ] |

### Checklist Caccia ai Bug Desktop (Stress & Edge Cases)

- [ ] **Memory Leak su Mappa**: Navigare avanti e indietro tra vista Mappa e lista ordini per 30 volte consecutive; la memoria occupata (RAM in Gestione Attività) deve rimanere stabile (< 350 MB).
- [ ] **Resize Estremo della Finestra**: Ridimensionare velocemente la finestra da 800x600 a schermo intero 4K; nessun overflow giallo/nero, nessun crash del motore di rendering.
- [ ] **Disconnessione di Rete Improvvisa**: Scollegare il cavo Ethernet o spegnere il Wi-Fi durante il calcolo percorso; l'app deve mostrare la notifica "Connessione Internet assente" senza crashare e preservando la navigazione nei dati locali.
- [ ] **Riconnessione Server Shelf**: Simulare la temporanea non disponibilità della porta locale 8080; l'UI deve mostrare la schermata di errore critico con il pulsante "Riprova Connessione".
- [ ] **Locking File SQLite su Sleep/Wake**: Mandare il PC Windows in sospensione con DelivMap aperto per 5 minuti; alla riattivazione l'app deve poter leggere e scrivere sul database senza eccezioni di lock.
- [ ] **File Backup Danneggiato**: Provare a ripristinare un file fittizio `.sqlite` contenente solo testo casuale; il sistema deve rifiutare il restore tramite `integrity_check`, ripristinare la copia di backup di sicurezza ed informare l'utente.

---

## Criteri di Accettazione

Per considerare conclusa la Fase 5 e pronto il software per il primo cliente pilota, devono essere soddisfatti i seguenti criteri oggettivi:

1. **Backup Automatico e Manuale**:
   - Il timer periodico esegue una copia valida del file SQLite ogni 24 ore (o secondo configurazione) con timestamp conforme.
   - La retention rimuove tempestivamente le copie oltre la quota configurata (default: 7).
   - L'utente può cambiare la cartella di salvataggio dei backup in qualsiasi momento tramite interfaccia grafica.
2. **Restore Sicuro e Verificato**:
   - L'operazione di ripristino richiede esplicita conferma dell'utente.
   - Prima della sovrascrittura viene generato uno snapshot temporaneo di sicurezza.
   - La connessione Drift viene chiusa e riaperta correttamente senza generare errori di violazione di condivisione file su Windows.
   - Viene eseguito `PRAGMA integrity_check`: in caso di esito non conforme, il restore viene annullato e i dati precedenti ripristinati.
3. **Esportazione CSV**:
   - L'utente può filtrare gli ordini per intervallo di date e salvare un file `.csv`.
   - Il file CSV generato si apre su Microsoft Excel (Windows) con caratteri accentati corretti (BOM UTF-8) e colonne separate da punto e virgola.
4. **Gestione Errori e Resilienza**:
   - Nessun crash non gestito dell'applicazione durante la normale operatività.
   - Presenza di messaggi in lingua italiana esplicativi per tutti i casi limite (server locale offline, chiave Google Maps errata, indirizzo introvabile).
   - Le liste vuote mostrano illustrazioni e suggerimenti amichevoli.
5. **Rifinitura Grafica (Polishing)**:
   - Tema Material 3 coerente con supporto chiaro/scuro e colori specifici per gli stati dell'ordine.
   - Animazione a pulsazione visiva per ordini nello stato `in_ritardo`.
   - Dimensione minima della finestra bloccata a 800x600 px.
6. **Pacchetto di Installazione**:
   - Compilazione release per Windows x64 completata senza errori o warning critici.
   - Installer Inno Setup generato, funzionante, in lingua italiana, che crea collegamenti nel menu Start e sul Desktop.
   - Disinstallazione che rimuove i binari preservando l'archivio dati dell'utente.

---

## Checklist di Testing

- [ ] **Unit Test Servizi**:
  - [ ] `BackupService`: test generazione nome file, conteggio retention e rimozione file più vecchi.
  - [ ] `OrderExportService`: test corretta formattazione CSV ed escaping di caratteri speciali (virgolette, punto e virgola, ritorni a capo).
  - [ ] `ApiClient`: mapping corretto dei codici HTTP verso le rispettive `AppException`.
- [ ] **Widget Test**:
  - [ ] `EmptyStateWidget` renderizza correttamente icona, titolo e risponde al tap dell'azione.
  - [ ] `OrderStatusBadge` attiva l'animazione di pulsazione solo per lo stato `in_ritardo`.
- [ ] **Test di Integrazione Desktop**:
  - [ ] Backup manuale da UI → verifica presenza file su cartella reale di Windows.
  - [ ] Restore da file backup → verifica aggiornamento immediato della UI con i dati ripristinati.
  - [ ] Cambio tema Chiaro/Scuro dalle impostazioni → ridisegno immediato di tutte le viste aperte.

---

## Checklist Pre-Release (Consegna al Primo Cliente Pilota)

Prima di procedere all'installazione di DelivMap presso il primo ristorante, verificare scrupolosamente ogni punto:

- [ ] **1. Pulizia Database di Test**: Verificare che l'installer installi un'app con database vuoto o predisposto per il wizard di primo avvio (nessun dato demo o credenziali di test rimaste memorizzate).
- [ ] **2. Verifica Chiave Google Maps (BYOK)**:
  - [ ] Geocoding API abilitata sulla Google Cloud Console del cliente.
  - [ ] Directions API abilitata.
  - [ ] Maps SDK / JavaScript abilitata.
  - [ ] Restrizioni della chiave verificate per prevenire usi non autorizzati.
- [ ] **3. Collaudo Installer su Macchina Pulita**: Testare l'installer `DelivMap_Setup_v1.0.0.exe` su un computer Windows 10/11 "pulito" (senza Flutter né toolchain di sviluppo installati) per accertare che non manchino librerie di runtime C++ (`MSVCP140.dll` / `VCRUNTIME140.dll`).
- [ ] **4. Test Stampa/Esportazione**: Eseguire una prova di esportazione CSV ed aprirla con la versione di Excel presente sul computer della cassa del ristorante.
- [ ] **5. Configurazione Backup Esterno**: Concordare con il ristoratore la cartella di backup (es. una cartella sincronizzata con OneDrive/Google Drive o chiavetta USB di sicurezza) e configurarla nell'app.
- [ ] **6. Mini-Guida di Emergenza**: Consegnare al ristoratore un promemoria sintetico plastificato (1 pagina A4) con:
  - Procedura di login e cambio password.
  - Cosa fare se non si trova un indirizzo (modifica manuale indirizzo o verifica CAP).
  - Come forzare un backup manuale a fine serata.
  - Recapito di supporto tecnico per la fase pilota.
