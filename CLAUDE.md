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
docs/
  piano-build.md      misure, scelte fatte e aperte per centralizzare e velocizzare
                      le build di tutti i progetti (con expo-ci e desktop-ci)
```

Chi riprende il lavoro sulle build legge prima il `CLAUDE.md` di `GabryXnLab/build-kit`
(architettura comune, modello da seguire), poi `docs/piano-build.md` (misure e storia).

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
- **Di norma tutto gira su nexus-core; `runner: github` è l'eccezione.** Serve quando
  la macchina non deve prendere altro carico. Su `ubuntu-latest` si saltano gli step
  del self-hosted (SDK in `PATH`, QEMU, `env` della macchina scritto in `GITHUB_ENV` e
  non a livello di job, perché un `env:` di job non si può omettere a condizione) e si
  usano `setup-java` e `subosito/flutter-action`. La chiave di firma arriva dal secret
  `ANDROID_DEBUG_KEYSTORE`: senza, l'APK ha un'altra firma e non si installa sopra
  quello del telefono. Vedi README. L'organizzazione è sul piano Free: niente runner
  più grandi (sono solo Team/Enterprise, e i benefici Education valgono per l'account
  personale), e un repo privato ha il runner standard da 2 CPU e 7 GB. Per questo lo
  step «Fit Gradle to the runner» abbassa la memoria di Gradle e Kotlin sotto i 12 GB
  (i valori dei progetti sono per nexus-core: il primo run su GitHub è stato spento per
  memoria esaurita) e aggiunge swap; Gradle e l'SDK sono in cache fra i run.
- **L'SDK Flutter non si scarica in CI** sul self-hosted. Il runner è ARM64 e la tarball ufficiale esiste
  solo per x86-64: sotto QEMU `dart` va in `SIGSEGV`. L'SDK è un clone git in
  `/home/ubuntu/sdk/flutter` e i workflow si limitano a verificarlo e metterlo in `PATH`.
  Per lo stesso motivo lì non si usa `subosito/flutter-action` (sui runner GitHub sì).
- **L'AOT Android su host ARM64 passa da un emulatore x86-64: Box64, con QEMU 10 di
  riserva.** Due limiti, misurati: (1) Flutter pubblica `gen_snapshot` per host
  **linux-x64** e non per host linux-arm64 — `…/<engine>/android-arm64-release/linux-arm64.zip`
  risponde **404**, la stessa URL con `linux-x64.zip` risponde 200 — quindi `release` e
  `profile` muoiono con «Failed to find … /linux-arm64/gen_snapshot» e nessun `flutter
  precache` li salva; (2) il binario x86-64 sotto il QEMU di sistema (**8.2** di Ubuntu
  24.04) cade con «QEMU internal SIGSEGV {code=MAPERR, addr=0x20}». Con QEMU 10 passa, ma era
  il **60% della build** (321 s su 545, profilo del 26/09). Box64 v0.4.4 fa lo stesso lavoro
  in **45 s** con un `app.so` identico byte per byte; se esce con errore il lanciatore
  ripete con QEMU. Lo step *gen_snapshot per host ARM64* mette un wrapper dove Flutter cerca
  l'eseguibile; emulatore e lanciatore li prepara `GabryXnLab/build-kit/x86-64` (input
  `x86_emulator`). `debug` non ci passa: è JIT.
- **Sul self-hosted si va veloci facendo meno lavoro.** Il checkout non pulisce
  (`clean: false`): intermedi di Flutter, Gradle, Kotlin e CMake restano fra i run, e una
  build che non cambia il Dart non rifà l'AOT (Kagami: 41 s di Build, 1m20s il run). Worker,
  cache condivise fra progetti e `clear_cache` di sola esecuzione vengono da
  `GabryXnLab/build-kit/setup`, uguale per expo-ci e desktop-ci: `max_workers` (`auto` |
  `2` | `4`) arriva a Gradle con `GRADLE_OPTS -D`, che prevale su
  `~/.gradle/gradle.properties` della macchina solo per quel run. `clear_cache` cancella le
  cartelle del progetto e spegne la build cache per il run, ma non tocca `~/.gradle`,
  pub-cache e ccache, che sono di tutti. Architettura e modello per ogni build nuova: il
  `CLAUDE.md` di `build-kit`.
- **`/opt/android-sdk` è condiviso con altri servizi della macchina.** I workflow non ci
  installano e non ci rimuovono niente. Attenzione: *Gradle* sì, di suo, quando un
  progetto dichiara un NDK o una platform che non c'è — è il progetto a doverlo
  vincolare, non questo repo.
- **`flutter pub`, mai `dart pub`.** In un progetto Flutter il secondo risolve senza le
  dipendenze dell'SDK e lascia un lockfile incoerente. `dart run build_runner` invece è
  corretto ed è il comando documentato dai progetti.
- **Niente firma di release.** I progetti che passano di qui sono sideload: l'APK
  `release` esce firmato con il `debug.keystore` del runner (su GitHub quello di
  nexus-core, dal secret `ANDROID_DEBUG_KEYSTORE`). Il giorno
  che servirà un'identità di firma vera (Play Services, Google Sign-In) si aggiungerà un
  input `signing_keystore` come in `expo-ci`, con il keystore sul runner in
  `/home/ubuntu/secrets/` e la verifica dell'impronta dopo la build — non prima che
  serva.
- **La notifica Telegram passa dall'azione `GabryXnLab/ci-bot/notify@main`** (bot
  `@BobCI_bot` dedicato alla CI): best-effort, artefatto o riepilogo se il job riesce,
  altrimenti messaggio con il pulsante «🔁 Rilancia». Server locale a 2 GB, topic come campo
  a parte e formato del messaggio stanno lì, non qui: vedi `CLAUDE.md` di `ci-bot`.
- **`flutter-update.yml` non è un OTA.** In Flutter il codice Dart è AOT dentro l'APK:
  non esiste un aggiornamento che non passi da una nuova build. Quel workflow aggiorna le
  **dipendenze** e committa il lockfile; l'APK lo rifà `flutter-build.yml`.
- **Nei test di condizione in coda a uno script con `set -e` si usa `if`, non `a && b`**:
  una condizione falsa come ultimo comando fa uscire lo step con exit 1 e la build muore
  senza un errore vero.

## Convenzioni

Nomi di input e identificatori in inglese, descrizioni e commenti in italiano.
I commenti spiegano il perché, non il cosa. Commit atomici, messaggi in italiano.
