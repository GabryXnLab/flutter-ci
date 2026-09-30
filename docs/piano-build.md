# Piano: build più veloci e centralizzate (flutter-ci, expo-ci, desktop-ci)

Documento di passaggio fra sessioni, scritto il 2026-09-26. Raccoglie ciò che è stato
misurato e scoperto sui workflow di build, le modifiche già fatte e le scelte ancora
aperte, per la sessione in cui si rivedranno e centralizzeranno i componenti di build
di tutti i progetti (Kagami su `flutter-ci`, Riftgate mobile su `expo-ci` e desktop su
`desktop-ci`).

## Esito della sessione del 26/09 (sera) — leggere prima questo

Gli obiettivi qui sotto sono stati affrontati. L'architettura che ne è uscita, con il
modello da seguire per ogni build nuova, sta nel `CLAUDE.md` di **`GabryXnLab/build-kit`**
(nuovo repo privato, `~/repos/build-kit`): quello è ora il documento di riferimento; questo
resta come storia e misure.

| Scelta aperta | Decisione |
| --- | --- |
| 1. `max_workers` con parità | Fatto: `auto` (default: CPU libere al momento, minimo 2; su GitHub `nproc`) \| `2` \| `4`, in flutter-ci, expo-ci, desktop-ci (→ `CARGO_BUILD_JOBS`) e nei wrapper di Kagami, Riftgate mobile e desktop, ascend. Applicato da `build-kit/setup` con `GRADLE_OPTS -D`; parallelo di Gradle sempre acceso, limitato da `workers.max` |
| 2. Heap di Gradle | Invariato (4 GB della macchina): due build insieme stanno in RAM |
| 3. Daemon fra i run | No: il runner uccide gli orfani, e l'avvio JVM pesa secondi contro minuti di AOT |
| 4. Configuration cache | Non provata: il guadagno è sulla configurazione (~30 s), non dove sta il tempo |
| 5. Batch e parallelo | Secondo runner `nexus-core-2` (`~/ci/actions-runner-2`, stesso utente, cache in comune): due build insieme, provato con Kagami + Riftgate. `build-kit/bin/ci-batch` lancia più progetti in un colpo |
| 6. Scelta del runner unificata | Nome `runner` nei wrapper; expo-ci senza percorso GitHub (resta EAS): rimandato |
| 7. `clear_cache` unificato | Fatto: pulisce solo il progetto, e per quel run non si fida di build cache/ccache/sccache (`RECACHE`); le cache condivise non si cancellano più (expo-ci cancellava `~/.gradle/caches` e ccache) |
| 8. `flutter_version` nel wrapper | Invariato |
| 9. Repo pubblici | Non toccato |

Scoperte principali:

- **Il tempo della build release di Kagami era per il 60% `gen_snapshot` sotto QEMU**
  (profilo: 545 s, di cui 321 di AOT, 51 di compilazione Dart, ~130 di Gradle quasi tutto
  dalla build cache). Box64 v0.4.4 fa lo stesso in 46 s con output identico, purché il GC
  del Dart VM giri a un thread: con i thread paralleli e la CPU contesa (in CI) cadeva in
  «double free». Il wrapper usa due esecuzioni concordi o ripiega su QEMU. QEMU non
  accelera cambiando modello di CPU (317–328 s).
- Misure dopo le modifiche: build **completamente pulita** in CI (`clear_cache`, niente
  intermedi né build cache) 380 s di step Build contro 659–709 s con QEMU; build locale con
  una modifica al Dart e intermedi presenti: task Gradle **152 s** contro 545.
- Build di Kagami senza modifiche al Dart dopo `clean: false`: **1m20s** l'intero run
  (Build 41 s), contro i 7–15 minuti di prima.
- Riftgate: mobile e desktop condividono la cartella di lavoro sul runner, e il `git clean`
  dell'uno cancellava gli intermedi dell'altro; ora nessuno pulisce se non con
  `clear_cache` e la target di cargo sta in `~/ci/cache/cargo-target/<repo>`. La build
  desktop falliva comunque (`npm ci` su un progetto pnpm): corretto in desktop-ci.
- `hermesc` di Riftgate con Box64: 43 s contro 95 s del QEMU di binfmt, identico.
- La macchina non è più condivisa con GitLab (`gitlab-runner` rimosso il 25/09).

- Riftgate desktop dopo le correzioni (runner `nexus-core-2`): prima build 888 s a freddo
  (riempie sccache e `~/ci/cache/cargo-target/Riftgate`), seconda **370 s**: frontend 61 s,
  solo il crate dell'app ricompilato (119 s, profilo release), poi i bundle — **rpm 106 s,
  AppImage 78 s**. Se servono solo `.deb` e AppImage, togliere `rpm` dai `bundle.targets`
  del progetto è il prossimo guadagno (scelta del progetto, non della CI).

Da misurare nelle prossime build normali: il tempo in CI della build Kagami con modifiche al
Dart (atteso ~3–4 minuti di run), la build mobile di Riftgate a macchina scarica (la
prova del 26/09 era a carico 12–14).

## Obiettivi della sessione (26/09, pomeriggio)

1. **Rivedere e centralizzare** ciò che i tre reusable fanno ognuno a modo suo: scelta
   del runner, cache di Gradle, pulizia del checkout, firma, taratura della memoria.
2. **Componenti comuni condivisi**, a partire da Gradle: stessa cache, stesse
   impostazioni, stesso modo di scegliere worker e memoria fra Flutter ed Expo.
3. **Build in parallelo, anche di più progetti in batch**, per la massima velocità ed
   efficienza sulla macchina e fuori.
4. **Opzione `max_workers` nelle opzioni di avvio**, con **parità** fra tutti i
   workflow di build: Kagami (Build Android) e Riftgate (build mobile e build desktop)
   devono esporre la stessa scelta, con lo stesso nome, gli stessi valori e lo stesso
   significato.

## Com'è oggi

### I tre reusable

| | `flutter-ci/flutter-build.yml` | `expo-ci/expo-build.yml` | `desktop-ci/tauri-build.yml` |
| --- | --- | --- | --- |
| Chi lo usa | Kagami (`build-android.yml`) | Riftgate (`build-mobile.yml`, job APP) | Riftgate (`build-desktop.yml`) |
| Scelta del runner | input `runner`: `self-hosted` \| `github` (aggiunto il 26/09) | nessuna: `runs-on: [self-hosted, nexus-core]` cablato; il wrapper sceglie `build_target: local \| eas` (EAS = cloud Expo, non GitHub) | input `runner_type` (il wrapper lo chiama `runner`: `self-hosted` \| `github`) più override per piattaforma in JSON |
| Checkout | `clean: false` sul self-hosted (dal 26/09) | `clean` solo con `run_prebuild` o `clear_cache` | da verificare |
| Build cache Gradle | dalla macchina (`~/.gradle/gradle.properties`: `org.gradle.caching=true`) | `--build-cache` esplicito, tolto con `clear_cache` | n/a (Tauri/Rust) |
| Parallelo Gradle | spento dalla macchina | `--parallel` solo nei task di codegen | n/a |
| Firma | debug.keystore del runner; su GitHub dal secret `ANDROID_DEBUG_KEYSTORE` | input `signing_keystore` (keystore in `/home/ubuntu/secrets/`) con verifica dello SHA-1 | chiavi Tauri dai secret |
| `clear_cache` | toglie `build/`, `.dart_tool/`, `android/.gradle` | toglie anche `~/.gradle/caches` e ccache | — |

Gli altri job di Riftgate (`build-mobile.yml`, oltre all'app) girano già su
`ubuntu-latest` / `ubuntu-24.04-arm`, non su nexus-core.

### nexus-core: la configurazione di Gradle vince su tutto

`~/.gradle/gradle.properties` della macchina **prevale** sul `gradle.properties` dei
progetti, per scelta: la macchina è condivisa con GitLab e altri servizi.

```properties
org.gradle.jvmargs=-Xmx4g -Xms256m -XX:MaxMetaspaceSize=512m -XX:+UseG1GC ...
org.gradle.daemon=false
org.gradle.workers.max=2          # 4 CPU disponibili, condivise con GitLab
org.gradle.parallel=false
org.gradle.caching=true
org.gradle.configureondemand=true
```

Conseguenze da sapere:

- Kagami chiede `-Xmx8G -XX:MaxMetaspaceSize=4G` nel suo `android/gradle.properties`,
  ma su nexus-core ottiene 4 GB. Riftgate chiede `-Xmx2048m`, e anche lui ottiene 4 GB.
- **`GRADLE_OPTS` con `-D` prevale sul file della macchina.** Verificato il 26/09:
  `GRADLE_OPTS="-Dorg.gradle.workers.max=4 -Dorg.gradle.parallel=true" ./gradlew help
  --info` riporta «Using 4 worker leases» e `parallelProjectExecution=true`; con il
  solo file, «Using 2 worker leases». È la leva per un `max_workers` scelto al lancio,
  valido solo per quel run e senza toccare la configurazione della macchina.
- Il daemon è spento: ogni build paga l'avvio della JVM di Gradle. Il runner, a fine
  job, uccide comunque i processi rimasti («Cleaning up orphan processes»).

Macchina: 4 CPU, 23 GB di RAM (circa 16 GB disponibili quando è stata misurata).

### Tempi misurati (Kagami, APK release arm64)

Self-hosted, run 36183873807 (25/09), 7m28s in tutto:

| Fase | Tempo |
| --- | --- |
| checkout, `flutter doctor`, `pub get` | ~7 s |
| `flutter analyze` | 9 s |
| `flutter test` | 22 s |
| Build: dal comando all'avvio di `assembleRelease` | 30 s |
| Build: compilazione Dart fino al tree-shaking dei font | 50 s |
| Build: resto di Gradle (AOT sotto QEMU, Kotlin/Java dei plugin, nativo, R8) | 5m10s |
| upload dell'artifact, Telegram | ~8 s |

Un altro run dello stesso giorno (36177383912) ha impiegato 652 s nel solo step Build:
la variabilità dipende dal carico della macchina.

Il checkout di quei run eseguiva `git clean -ffdx` e toglieva `build/`, `.dart_tool/`,
`android/.gradle/`, `android/.kotlin/`: ogni build ricompilava da zero plugin e nativo.
Dal commit `2e8d430` di flutter-ci non succede più sul self-hosted; **il guadagno non
è ancora stato misurato** (la prima build dopo il cambio è comunque completa).

GitHub, run 36252509943 (26/09), **fallito** dopo 15 minuti:

| Fase | Tempo |
| --- | --- |
| Set up Flutter (download dell'SDK) | 65 s |
| `flutter doctor` + `pub get` | 22 s |
| `flutter analyze` + `flutter test` | 66 s |
| Gradle prima di compilare (download di AGP, Kotlin, librerie; installazione di CMake) | ~4 min |
| Compilazione, poi il runner spento | ~8 min |

Log: «The runner has received a shutdown signal», Gradle uscito con 143. Causa più
probabile, non dimostrata dal log: **memoria esaurita**. Il runner standard di un repo
privato ha 2 CPU e 7 GB, e Kagami chiedeva 8 GB di heap più 4 di Metaspace, con il
demone Kotlin accanto.

## Già fatto il 26/09 (tutto su `main`)

| Repo | Commit | Cosa |
| --- | --- | --- |
| flutter-ci | `35ed7b6` | input `runner` (`self-hosted` \| `github`) e `flutter_version`; su GitHub `setup-java` 17, `subosito/flutter-action`, niente QEMU; l'ambiente del self-hosted passa da `GITHUB_ENV` invece che dall'`env:` del job (non si può omettere a condizione); secret `ANDROID_DEBUG_KEYSTORE` ripristinato in `~/.android/debug.keystore` |
| flutter-ci | `3cc6e55` | solo su GitHub: `gradle.properties` dell'utente con heap 3 GB, Metaspace 1 GB, Kotlin 1,5 GB sotto i 12 GB di RAM; swap da 8 GB; `gradle/actions/setup-gradle` per la cache; CPU e RAM stampate nel log |
| flutter-ci | `2e8d430` | solo self-hosted: `actions/checkout` con `clean: false`, build incrementale |
| kagami | `8ff986d` | input `runner` nel lancio di Build Android, `flutter_version: '3.47.4'`, secret passato al reusable |
| kagami (secret) | — | `ANDROID_DEBUG_KEYSTORE` = debug.keystore di nexus-core in base64; SHA-1 `43:A3:81:…:A7:36`, uguale a `certificate_hash` di `google-services.json` |

**La build su GitHub con le correzioni di `3cc6e55` non è ancora stata provata.**

## Scoperte che valgono per tutti i progetti

- **Runner più grandi: non disponibili.** L'organizzazione GabryXnLab è sul piano
  **Free**; i larger runner (4–64 CPU) esistono solo per Team/Enterprise
  (`gh api orgs/GabryXnLab/actions/hosted-runners` → «GitHub hosted runners are not
  supported for this organization»).
- **GitHub Education** dà GitHub Pro all'account personale **GabryXn** (più minuti e più
  spazio per i repo personali), **non** all'organizzazione, e non dà runner più grandi.
  I repo privati dell'organizzazione usano la quota Free: 2 000 minuti al mese.
- **Runner standard**: repo privato 2 CPU / 7 GB; repo **pubblico** 4 CPU / 16 GB e
  minuti illimitati. Rendere pubblico un repo è l'unico modo gratuito di avere una
  macchina più grande (da decidere repo per repo).
- **ARM64 hosted** (`ubuntu-24.04-arm`) è disponibile anche ai privati (Riftgate lo usa,
  voce «Actions Linux ARM» nel billing). Per Flutter è peggio: non esiste
  `gen_snapshot` per host linux-arm64 e l'AOT tornerebbe sotto QEMU come su nexus-core.
- **Firma su GitHub**: un runner nuovo genera un debug.keystore nuovo, quindi l'APK non
  si installa sopra quello del telefono e Google Sign-In non ne riconosce l'impronta. Il
  keystore va passato da secret (fatto per Kagami). Riftgate ha già `signing_keystore`
  in expo-ci, ma legge da `/home/ubuntu/secrets/`, quindi funziona solo sul self-hosted.
- **Il costo dominante sul self-hosted è l'AOT sotto QEMU più Gradle a freddo.** L'AOT
  non si evita su ARM64; il resto si recupera con build incrementali e build cache.

## Scelte aperte

1. **`max_workers` al lancio, con parità.**
   - Proposta: input `max_workers` in tutti i reusable Android (flutter-ci, expo-ci) e
     nei wrapper (Kagami Build Android, Riftgate build mobile), stessa descrizione e
     stessi valori.
   - Valori proposti: `macchina` (default: quello di `~/.gradle/gradle.properties`,
     cioè 2) \| `2` \| `4`. Con `4` si accende anche `org.gradle.parallel`. Si passa con
     `GRADLE_OPTS="-Dorg.gradle.workers.max=N -Dorg.gradle.parallel=true"`, valido per
     quel run.
   - Riftgate desktop (Tauri/Rust): l'equivalente è `CARGO_BUILD_JOBS`. Decidere se la
     parità vuol dire lo stesso input anche lì, mappato su cargo.
   - Su GitHub il valore sensato è `nproc`; l'input ha senso soprattutto sul self-hosted.
   - Rischio da pesare: con 4 worker la macchina è piena per tutta la build, ed è
     condivisa con GitLab e altri servizi.
2. **Heap di Gradle sul self-hosted.** Oggi vince il 4 GB della macchina. Con 4 worker
   e parallelo potrebbe non bastare: misurare prima di alzarlo.
3. **Daemon di Gradle fra i run.** Tenerlo vivo farebbe risparmiare l'avvio della JVM e
   la JIT a ogni build, ma occuperebbe memoria sulla macchina condivisa anche a riposo,
   e il runner lo uccide a fine job (servirebbe aggirare la pulizia degli orfani).
   Probabilmente no; da decidere.
4. **Configuration cache di Gradle** (`org.gradle.configuration-cache`): non provata,
   compatibilità incerta con il plugin Gradle di Flutter e con Expo.
5. **Build in batch e in parallelo di più progetti.**
   - Sul self-hosted c'è **un solo runner** (`nexus-core`): due progetti lanciati
     insieme si mettono in fila. Per il parallelismo vero servono più istanze del
     runner sulla stessa macchina (con etichette), da pesare contro CPU e RAM
     condivise, o spostare una parte dei job su GitHub.
   - Un workflow «batch» (per esempio in un repo centrale) che con un solo lancio
     chiama le build di più progetti, ognuna col suo runner. Oggi i reusable sono per
     progetto; il batch avrebbe bisogno di `workflow_call` verso i wrapper o di
     `gh workflow run` con un token dell'organizzazione.
   - **Gradle in comune**: la cache `~/.gradle/caches` (e la build cache
     `build-cache-1`) è già condivisa fra i progetti sulla macchina. Due build Gradle
     in parallelo la contendono con un lock: da misurare.
   - Su GitHub la cache di `setup-gradle` è per repo: non c'è un Gradle comune fra
     progetti diversi.
6. **Unificare la scelta del runner.** flutter-ci ha `runner`, desktop-ci ha
   `runner_type` (con `runner` nel wrapper), expo-ci non ce l'ha (EAS è un'altra cosa).
   Proposta: `runner: self-hosted | github` ovunque, stessa descrizione, e expo-ci con
   un percorso GitHub analogo a quello di flutter-ci (setup-java, cache Gradle,
   memoria tarata, keystore da secret).
7. **Unificare `clear_cache`.** flutter-ci toglie solo le cache del progetto; expo-ci
   toglie anche `~/.gradle/caches` e ccache, che sono **condivise** con gli altri
   progetti: su una macchina condivisa, pulire le cache di tutti per una build sola va
   ripensato.
8. **`flutter_version` nel wrapper di Kagami** è fissato a `3.47.4`, da aggiornare a
   mano insieme al clone su nexus-core. Alternativa: leggere la versione dal clone,
   esposta in un file del repo (`.flutter-version` per `flutter-version-file`).
9. **Repo pubblici** per avere 4 CPU / 16 GB su GitHub, per i progetti dove non ci sono
   segreti nel codice. Decisione per progetto.

## Da verificare all'inizio della prossima sessione

- Un run di Kagami con `runner=github` dopo `3cc6e55`: riesce? Quanto dura con la cache
  vuota e con la cache calda? Il log di «Fit Gradle to the runner» dice CPU e RAM.
- Un run self-hosted di Kagami dopo `2e8d430`: tempo dello step Build alla prima build e
  alla seconda (quella che trova gli intermedi).
- Che `clean: false` non lasci residui sbagliati passando da un branch all'altro.
- Le stesse misure per Riftgate mobile (expo-ci) e desktop (desktop-ci), come base per
  la parità.

## Collegamenti

- `flutter-ci/CLAUDE.md` e `README.md`: vincoli di nexus-core (SDK clonato, QEMU 10,
  SDK Android condiviso) e percorso GitHub.
- `kagami/.github/workflows/CLAUDE.md`: il wrapper di Kagami e il secret della firma.
- `Riftgate/.github/workflows/CLAUDE.md`, `expo-ci`, `desktop-ci`: gli altri due reusable.
- Wiki dell'infrastruttura: nexus-core, runner self-hosted, bot Telegram della CI.
