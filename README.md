# flutter-ci

Workflow GitHub Actions **riusabili** per i progetti Flutter/Dart dell'organizzazione,
eseguiti sul runner self-hosted `nexus-core` (Ubuntu 24.04 **ARM64**).

È la controparte Flutter di [`expo-ci`](https://github.com/GabryXnLab/expo-ci) (Expo /
React Native) e [`desktop-ci`](https://github.com/GabryXnLab/desktop-ci) (Tauri): stesso
schema, un repo centrale con la logica e thin wrapper nei progetti.

| Workflow | Cosa fa |
| :--- | :--- |
| `flutter-build.yml` | APK/AAB Android, bundle Linux desktop o IPA iOS non firmato, artifact del run + invio su Telegram |
| `flutter-update.yml` | `flutter pub upgrade` con verifica e commit del lockfile |
| `flutter-check.yml` | formato, analisi statica, test, pacchetti Dart puri (`dart_packages`), controllo del codice generato |

## Uso

Nel progetto, un wrapper che contiene solo le scelte:

```yaml
name: Build Android
run-name: Build Android · ${{ inputs.build_mode }}

on:
  workflow_dispatch:
    inputs:
      build_mode:
        type: choice
        options: [release, debug, profile]
        default: release

jobs:
  android:
    uses: GabryXnLab/flutter-ci/.github/workflows/flutter-build.yml@main
    with:
      app_name: Kagami
      build_mode: ${{ inputs.build_mode }}
      telegram_topic_id: '6'
    secrets:
      TELEGRAM_BOT_TOKEN: ${{ secrets.TELEGRAM_BOT_TOKEN }}
      TELEGRAM_CHAT_ID: ${{ secrets.TELEGRAM_CHAT_ID }}
```

I secret si passano **per nome**, non con `secrets: inherit` (inaffidabile cross-repo).

## Cosa dà per scontato del runner

Niente di tutto questo viene installato dai workflow: il runner lo ha già, e i
workflow lo verificano e si fermano subito se manca.

| Cosa | Dove | Perché non si installa in CI |
| :--- | :--- | :--- |
| SDK Flutter | `/home/ubuntu/sdk/flutter` (clone git) | La tarball ufficiale `flutter_linux_*` esiste **solo per x86-64**: su questa macchina ARM64 `dart` muore con `QEMU internal SIGSEGV`. Solo il clone git è nativo. Per lo stesso motivo non si usa `subosito/flutter-action`. |
| SDK Android | `/opt/android-sdk` | **Condiviso con altri servizi della macchina.** I workflow non ci installano né rimuovono pacchetti. |
| JDK 17 | `/usr/lib/jvm/java-17-openjdk-arm64` | Gradle gira sulla JVM di sistema. |
| Pub cache | `/home/ubuntu/.pub-cache` | Condivisa fra i run: è ciò che rende `flutter pub get` quasi istantaneo. |
| Box64 v0.4.4 e QEMU 10 x86-64 | `~/ci/tools/box64`, `/home/ubuntu/qemu-x86_64-10/bin/qemu-x86_64` | Servono all'AOT Android: Flutter non pubblica `gen_snapshot` per host linux-arm64, e quello x86-64 sotto il QEMU di sistema (8.2) va in SIGSEGV. Box64 (default: due esecuzioni concordi, ~45 s) con QEMU 10 di riserva (316 s). Li installa da sé, sotto `$HOME` e senza sudo, l'azione `GabryXnLab/build-kit/x86-64`. |

Tutti questi percorsi sono input con quel default: un runner diverso li sovrascrive
senza toccare questo repo.

## Worker e cache: `build-kit`

`flutter-build.yml` passa per `GabryXnLab/build-kit/setup`, come expo-ci e desktop-ci:
input `max_workers` (`auto` = CPU libere di nexus-core in quel momento, minimo 2 | `2` |
`4`) e `clear_cache` con lo stesso significato in tutti i workflow di build. Sul
self-hosted girano due runner (`nexus-core`, `nexus-core-2`), quindi due build insieme;
`~/repos/build-kit/bin/ci-batch` le lancia in un colpo.

## Build sui runner di GitHub

`flutter-build.yml` ha l'input `runner`: `self-hosted` (default, tutto quanto sopra) o
`github`, per quando nexus-core non deve prendere altro carico. Su `ubuntu-latest`
niente della tabella serve: il JDK 17 lo mette `actions/setup-java`, l'SDK Flutter
`subosito/flutter-action` (x86-64, quindi la tarball ufficiale va bene) alla versione di
`flutter_version`, e l'AOT non passa da QEMU. Consuma minuti Actions.

Il wrapper espone la scelta come input `choice` e passa `runner`, `flutter_version` e i
secret della firma.

## Firma

L'APK/AAB `release` si firma con la chiave che il progetto passa nei secret, su qualunque
runner:

| Secret | Senza |
| :--- | :--- |
| `ANDROID_KEYSTORE` | sul self-hosted la chiave della macchina, come sempre; su GitHub `ANDROID_DEBUG_KEYSTORE`, e senza anche quello una chiave nuova a ogni run (avviso nel log) |
| `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD` | quelli di una chiave di debug: `android`, `androiddebugkey`, password della chiave = password del keystore |
| `ANDROID_DEBUG_KEYSTORE` | solo su GitHub e solo senza `ANDROID_KEYSTORE`: il `debug.keystore` di nexus-core, per chi lo passa già |

Il keystore (in base64: `base64 -w0 <file> | gh secret set ANDROID_KEYSTORE -R
GabryXnLab/<progetto>`) arriva a Gradle da `android/key.properties`, lo schema della
[documentazione di Flutter](https://docs.flutter.dev/deployment/android#configure-signing-in-gradle):
**il `build.gradle` del progetto deve leggerlo**. Copiarlo in `~/.android/debug.keystore`
non basta, perché sui runner di GitHub Gradle ignora quel file e ne genera un altro.
Dopo la build il riepilogo del run ha la SHA-1 di ogni file prodotto; se c'era una chiave
nei secret e la firma di una release è un'altra, la build fallisce invece di consegnare un APK che non
si installa sopra il precedente. A fine job `key.properties` si cancella (sul self-hosted
il checkout resta fra i run).

## iOS: IPA non firmato

`platform: ios` compila su `macos-latest`, quindi solo con `runner: github` (con il
self-hosted il job si ferma subito con un errore). `flutter build ios --release
--no-codesign`, poi `Payload/<App>.app` zippato in `<app_name>-ios-<build_mode>-unsigned.ipa`,
artifact del run e file su Telegram. Senza firma: lo firma chi lo installa, con AltStore
o Sideloadly e il proprio Apple ID, quindi non serve un account Apple Developer. I plugin
li risolve Flutter (Swift Package Manager, o CocoaPods per chi non lo supporta);
deployment target e `Podfile`, se servono, sono del progetto. In un repo privato un
minuto macOS vale dieci minuti Linux; nei repo pubblici i minuti sono gratis.

```yaml
  ios:
    uses: GabryXnLab/flutter-ci/.github/workflows/flutter-build.yml@main
    with:
      app_name: Kagami
      runner: github
      platform: ios
      flutter_version: '3.47.4'
```

## Configurazione di chi compila

Due secret facoltativi per ciò che non sta nel repo:

- `GOOGLE_SERVICES_JSON`: il contenuto di `google-services.json`, scritto in
  `android/app/` (solo Android) e tolto a fine job;
- `DART_DEFINES`: come l'input `dart_defines` (coppie `CHIAVE=valore` separate da spazi),
  per i valori da non scrivere nel wrapper. Un secret non si può passare in `with:`, ma
  sì comporre nel blocco `secrets:` del wrapper:

```yaml
    secrets:
      GOOGLE_SERVICES_JSON: ${{ secrets.GOOGLE_SERVICES_JSON }}
      DART_DEFINES: SENTRY_DSN=${{ secrets.SENTRY_DSN }}
```

## Perché non è un submodule

Un reusable workflow GitHub **deve** stare in `.github/workflows/` di un repository e si
referenzia con `uses: owner/repo/.github/workflows/file.yml@ref`. Da dentro un submodule
non viene visto: il submodule è codice nell'albero del progetto, non un workflow
registrato. Vale lo stesso vincolo che ha portato `expo-ci` e `desktop-ci` a essere repo a
sé — l'effetto pratico è identico a un submodule (logica in un posto solo, aggiornata una
volta per tutti), ma il meccanismo è `uses:`.

Poiché i consumatori referenziano `@main`, **ogni push qui ha effetto immediato su tutti i
progetti** al loro prossimo run: gli input vanno mantenuti retrocompatibili (nuovi input
con `default`, mai rinominare o rimuovere senza aggiornare ogni wrapper).

## Visibilità

Questo repo è **pubblico**: lo chiamano anche repo pubblici (Kagami), che GitHub non lascia
usare workflow o azioni di repo privati. Per lo stesso motivo le azioni che usa stanno in
`GabryXnLab/build-kit`, pubblico anche lui. Qui non c'è niente di segreto: chiavi, token e
configurazioni arrivano dai secret di chi chiama.

## Progetti che lo usano

- [`kagami`](https://github.com/GabryXnLab/kagami) — lettore di manga per archivi MALF: `build.yml` (APK per ABI, universale e IPA su GitHub a ogni push su `main` del repo pubblico) e, dal repo di sviluppo privato, `build-android.yml`, `check.yml` e `update-deps.yml` sul self-hosted.
