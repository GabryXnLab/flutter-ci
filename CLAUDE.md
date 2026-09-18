# CLAUDE.md — flutter-ci

## Cos'è

Repository dei **reusable workflow** GitHub Actions per i progetti Flutter/Dart
dell'organizzazione. La logica di build, aggiornamento e verifica sta qui una volta
sola; nei progetti restano thin wrapper con le sole scelte. È l'analogo Flutter di
`expo-ci` (Expo/React Native) e `desktop-ci` (Tauri).

`README.md` ha l'uso lato progetto; questo file ha i vincoli di chi ci mette mano.

## Stack

Solo YAML di GitHub Actions. Nessuna dipendenza, nessun build system: gli script negli
step sono bash (`set -euo pipefail`) e, dove serve parsare, `python3` del sistema.

## Struttura

```text
.github/workflows/
  flutter-build.yml   APK/AAB Android o bundle Linux, artifact + invio Telegram
  flutter-update.yml  flutter pub upgrade, verifica, commit del lockfile
  flutter-check.yml   format, analyze, test, controllo del codice generato
```

## Comandi

Non c'è nulla da compilare. Prima di pushare, la validazione minima:

```bash
python3 -c "import yaml,glob; [yaml.safe_load(open(f)) for f in glob.glob('.github/workflows/*.yml')]"
gh workflow list --repo GabryXnLab/<progetto>   # i wrapper che chiamano questi reusable
```

## Note architetturali importanti

- **`@main` è live per tutti.** I consumatori referenziano `uses: …@main`, senza pin per
  SHA: un push qui cambia il comportamento di ogni progetto al run successivo. Gli input
  restano retrocompatibili — se ne aggiungono con `default`, non se ne rinominano né
  rimuovono senza aggiornare tutti i wrapper.
- **L'SDK Flutter non si scarica in CI.** Il runner è ARM64 e la tarball ufficiale esiste
  solo per x86-64: sotto QEMU `dart` va in `SIGSEGV`. L'SDK è un clone git in
  `/home/ubuntu/sdk/flutter` e i workflow si limitano a verificarlo e metterlo in `PATH`.
  Per lo stesso motivo non si usa `subosito/flutter-action`.
- **`/opt/android-sdk` è condiviso con altri servizi della macchina.** I workflow non ci
  installano e non ci rimuovono niente. Attenzione: *Gradle* sì, di suo, quando un
  progetto dichiara un NDK o una platform che non c'è — è il progetto a doverlo
  vincolare, non questo repo.
- **`flutter pub`, mai `dart pub`.** In un progetto Flutter il secondo risolve senza le
  dipendenze dell'SDK e lascia un lockfile incoerente. `dart run build_runner` invece è
  corretto ed è il comando documentato dai progetti.
- **Niente firma di release.** I progetti che passano di qui sono sideload: l'APK
  `release` esce firmato con il `debug.keystore` generato dal template Flutter. Il giorno
  che servirà un'identità di firma vera (Play Services, Google Sign-In) si aggiungerà un
  input `signing_keystore` come in `expo-ci`, con il keystore sul runner in
  `/home/ubuntu/secrets/` e la verifica dell'impronta dopo la build — non prima che
  serva.
- **La notifica Telegram è best-effort.** Un errore di rete non marca rosso un job in cui
  l'artefatto è stato compilato e caricato. L'invio usa il Local Bot API Server su
  `localhost:8081` se risponde a `getMe` (limite ~2 GB), altrimenti la Bot API cloud
  (50 MB) e in tal caso, oltre soglia, manda solo il link al run.
- **Il topic Telegram va passato come campo a parte** (`message_thread_id`): un valore
  vuoto viene **rifiutato** dalla Bot API, quindi l'argomento `curl` si costruisce solo
  quando l'input è valorizzato.
- **`flutter-update.yml` non è un OTA.** In Flutter il codice Dart è AOT dentro l'APK:
  non esiste un aggiornamento che non passi da una nuova build. Quel workflow aggiorna le
  **dipendenze** e committa il lockfile; l'APK lo rifà `flutter-build.yml`.
- **Nei test di condizione in coda a uno script con `set -e` si usa `if`, non `a && b`**:
  una condizione falsa come ultimo comando fa uscire lo step con exit 1 e la build muore
  senza un errore vero.

## Convenzioni

Nomi di input e identificatori in inglese, descrizioni e commenti in italiano.
I commenti spiegano il perché, non il cosa. Commit atomici, messaggi in italiano.
