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
- **L'AOT Android su host ARM64 passa da QEMU, e serve QEMU 10.** Due limiti, misurati:
  (1) Flutter pubblica `gen_snapshot` per host **linux-x64** e non per host linux-arm64 —
  `…/<engine>/android-arm64-release/linux-arm64.zip` risponde **404**, la stessa URL con
  `linux-x64.zip` risponde 200 — quindi `release` e `profile` muoiono con «Failed to find
  … /linux-arm64/gen_snapshot» e nessun `flutter precache` li salva; (2) il binario
  x86-64 sotto il QEMU di sistema (**8.2** di Ubuntu 24.04) cade con «QEMU internal
  SIGSEGV {code=MAPERR, addr=0x20}» — è QEMU a morire, non Dart, e non dipende da
  `reserved_va` né dall'ASLR (provati). Con **QEMU 10** (Debian trixie, estratto con
  `dpkg-deb` sotto `$HOME`, nessun sudo) la stessa compilazione passa. Lo step *Fallback
  gen_snapshot per host ARM64* mette un wrapper dove Flutter cerca l'eseguibile e, se il
  QEMU non c'è, se lo installa. Costo misurato su Kagami: APK release arm64 da 22,4 MB in
  **8m10s** (contro ~70 s di una debug). `debug` non ci passa: è JIT.
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
