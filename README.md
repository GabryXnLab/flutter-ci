# flutter-ci

Workflow GitHub Actions **riusabili** per i progetti Flutter/Dart dell'organizzazione,
eseguiti sul runner self-hosted `nexus-core` (Ubuntu 24.04 **ARM64**).

È la controparte Flutter di [`expo-ci`](https://github.com/GabryXnLab/expo-ci) (Expo /
React Native) e [`desktop-ci`](https://github.com/GabryXnLab/desktop-ci) (Tauri): stesso
schema, un repo centrale con la logica e thin wrapper nei progetti.

| Workflow | Cosa fa |
| :--- | :--- |
| `flutter-build.yml` | APK/AAB Android o bundle Linux desktop, artifact del run + invio su Telegram |
| `flutter-update.yml` | `flutter pub upgrade` con verifica e commit del lockfile |
| `flutter-check.yml` | formato, analisi statica, test, controllo del codice generato |

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
| QEMU 10 x86-64 | `/home/ubuntu/qemu-x86_64-10/bin/qemu-x86_64` | Serve all'AOT Android: Flutter non pubblica `gen_snapshot` per host linux-arm64 e quello x86-64 sotto il QEMU di sistema (8.2) va in SIGSEGV. È l'unico che il workflow **installa da sé** se manca, sotto `$HOME` e senza sudo. |

Tutti questi percorsi sono input con quel default: un runner diverso li sovrascrive
senza toccare questo repo.

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

Questo repo è **privato**: perché gli altri repo dell'organizzazione possano usarne i
workflow, le impostazioni Actions del repo devono consentire l'accesso ai repository
dell'organizzazione (`Settings → Actions → Access → Accessible from repositories in the
GabryXnLab organization`).

## Progetti che lo usano

- [`kagami`](https://github.com/GabryXnLab/kagami) — lettore di manga per archivi locali.
