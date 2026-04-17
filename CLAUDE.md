# Vocali AI — Briefing per Claude Code (Android)

> **Ruolo di Claude Code:** Sei il co-sviluppatore di Vocali AI, un'app Android nativa che riassume i messaggi vocali di WhatsApp usando AI. Il tuo compito è guidare lo sviluppo end-to-end, scrivere codice production-ready, e assicurarti che l'app sia pronta per la pubblicazione sul Google Play Store italiano entro 30 giorni.

---

## 1. Contesto del progetto

### 1.1 Cosa stiamo costruendo
Un'app Android che permette agli utenti italiani di:
1. **Ricevere** un messaggio vocale di WhatsApp tramite Share Intent (o selezionarlo direttamente dalla cartella `WhatsApp/Media/WhatsApp Voice Notes/`)
2. **Trascrivere** automaticamente l'audio in italiano (Whisper API o on-device)
3. **Riassumere** il contenuto in 3 modalità: breve, dettagliato, azioni+punti chiave (GPT-4o-mini)
4. **Estrarre** automaticamente date, luoghi e task → Google Calendar / Google Tasks
5. **Archiviare** localmente tutti i vocali elaborati con ricerca full-text
6. **Generare** risposte suggerite nel tono appropriato (amichevole/lavoro/urgente)

### 1.2 Target utente
- Italiani 18-55 anni, utenti WhatsApp pesanti
- Ricevono 5-30 vocali al giorno tra gruppi famiglia, amici, lavoro
- Pain point: "non ho tempo/voglia di ascoltare 14 vocali da 3 minuti l'uno"
- Device: Android (API 26+ / Android 8.0+)

### 1.3 Perché Android è un'ottima scelta strategica
- **Italia è un mercato Android-dominante**: ~72% market share contro ~28% iOS
- Gli utenti WhatsApp pesanti (famiglie, gruppi amici, 40+) sono prevalentemente su Android
- Play Store ha review più veloce (ore vs giorni) → iterazioni più rapide
- Play Store Italia meno saturo di prodotti AI rispetto all'App Store
- Possibilità di leggere direttamente la cartella WhatsApp con permesso MANAGE_EXTERNAL_STORAGE → UX superiore a iOS

### 1.4 Perché funzionerà
- Italia è #2 in Europa per penetrazione WhatsApp (90.3%)
- I vocali sono strutturalmente più lunghi e frequenti in Italia che in US
- Gap confermato su Play Store IT: esistono solo trascrittori, nessun riassuntore AI con workflow completo
- Monetizzazione validata: Voicenotes ha fatto $100K in poche settimane, AudioPen sostiene $15K MRR

### 1.5 Competitor da studiare
- **AudioPen** (audiopen.ai) — web-first, inglese
- **Voicenotes** (voicenotes.com) — mobile, inglese
- **Transcribe for WhatsApp** — su Play Store, solo trascrizione
- In Italia: solo trascrittori base senza AI layer → da superare con summarization + workflow

### 1.6 Minaccia competitiva
Meta AI Message Summaries e Google Gemini su Android (Circle to Search, Gemini Live) arriveranno in Italia entro 12-18 mesi. La nostra difendibilità NON è la trascrizione/riassunto di base (commodity) ma il **workflow layer**: memoria cross-chat, ricerca semantica sui vocali passati, azioni integrate Android, gestione gruppi familiari. Costruiamo v1 focalizzata sul core, ma v2 deve avere queste feature.

---

## 2. Stack tecnologico (NON negoziabile)

### 2.1 Client Android
- **Kotlin 2.0+** (nessun Java)
- **Jetpack Compose** per tutta la UI (no XML layouts se non per edge cases)
- **Android API 26+** (Android 8.0 Oreo) minimum — copre ~97% dei device attivi
- **Android Studio Hedgehog o più recente**
- **Gradle** con Kotlin DSL (`build.gradle.kts`)
- **Architettura:** MVVM + UDF (Unidirectional Data Flow) seguendo le Android Architecture Guidelines

### 2.2 Librerie Android chiave
- **Jetpack Compose BOM** — UI
- **Hilt** — dependency injection
- **Room** — persistence locale (con supporto FTS per full-text search)
- **DataStore Preferences** — settings e key-value storage
- **WorkManager** — processing background robusto (trascrizioni lunghe)
- **Coroutines + Flow** — async everywhere
- **Navigation Compose** — navigazione
- **Coil 3** — image loading
- **Retrofit + OkHttp + kotlinx.serialization** — networking
- **Accompanist Permissions** — runtime permissions UX

### 2.3 AI / Backend
- **OpenAI Whisper API** (`whisper-1`) per trascrizione — qualità migliore di Google Speech API su italiano casual/dialettale
- **OpenAI GPT-4o-mini** per summarization e action extraction (rapporto qualità/costo ottimale)
- **Opzionale v1.5:** on-device Whisper via `whisper.cpp` compilato con JNI + GGML model piccolo (`ggml-small.bin` ~466MB) per tier privacy
- **Backend proxy:** Node.js + Express o Python FastAPI su **Fly.io** o **Railway** che:
  - Firma le richieste a OpenAI con API key lato server (MAI esporre la key nel client)
  - Enforce dei rate limit per utente (Redis o in-memory per MVP)
  - Logga errori (Sentry)
  - Valida receipt Google Play per server-side subscription validation
- **NON usare Firebase/Supabase per v1** — overhead inutile. Dati su device, sync futuro.

### 2.4 Monetizzazione
- **Google Play Billing Library 7.0+** (libreria nativa, non serve terza parte)
- **OPPURE RevenueCat SDK Android** se vuoi semplificare gestione multi-store futuro (consigliato)
- Prodotti Play Console:
  - Free tier: 3 riassunti/mese
  - `vocali_monthly`: €4.99/mese con 3 giorni trial
  - `vocali_yearly`: €29.99/anno con 7 giorni trial (highlighted come best value)

### 2.5 Analytics & monitoring
- **PostHog Android SDK** o **Mixpanel** (free tier) per product analytics
- **Firebase Crashlytics** per crash reporting (free, best-in-class Android)
- **Firebase Performance** (opzionale) per monitoring performance
- Eventi chiave: `audio_imported`, `summary_generated`, `paywall_shown`, `subscription_started`, `reminder_created`

### 2.6 Stack escluso esplicitamente
- ❌ Java (2026, Kotlin-only)
- ❌ XML Views (tranne Splash Screen theme)
- ❌ React Native / Flutter (serve nativo per Share Intent e gestione file robusta)
- ❌ LiveData (usa Flow/StateFlow)
- ❌ RxJava (usa Coroutines)
- ❌ Firebase Auth per v1 (overkill)

---

## 3. Architettura di sistema

```
┌────────────────────────────────────────────────────────────┐
│                Android App (Compose)                        │
│                                                              │
│  ┌─────────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Share Receiver  │  │ MainActivity │  │ Settings     │  │
│  │ (Intent Filter) │  │ (Compose NAV)│  │ Screen       │  │
│  └────────┬────────┘  └──────┬───────┘  └──────┬───────┘  │
│           │                   │                  │          │
│           └──────────┬────────┴──────────────────┘          │
│                      │                                       │
│          ┌───────────▼────────────┐                          │
│          │   ViewModels (Hilt)    │                          │
│          │   + UseCases           │                          │
│          └───────────┬────────────┘                          │
│                      │                                       │
│  ┌───────────────────▼─────────────────────┐                │
│  │  Domain Layer (Services / Repositories)  │               │
│  │  - TranscriptionRepository              │                │
│  │  - SummaryRepository                    │                │
│  │  - SubscriptionRepository               │                │
│  │  - ActionExtractorUseCase               │                │
│  └───────────────────┬─────────────────────┘                │
│                      │                                       │
│  ┌─────────┬─────────┼────────────┬──────────────┐          │
│  │         │         │            │              │          │
│  ▼         ▼         ▼            ▼              ▼          │
│ Room    DataStore  WorkMgr   Retrofit    Billing Library   │
│ (+FTS)  (prefs)    (bg jobs) (HTTP)      (Play Store)       │
└─────────────────────────────────────────────────────────────┘
                         │ HTTPS
                         ▼
             ┌────────────────────────┐
             │   Backend Proxy        │
             │   (Fly.io / Railway)   │
             │   - API key vault      │
             │   - Rate limiting      │
             │   - Play receipt       │
             │     validation         │
             └───────────┬────────────┘
                         │
             ┌───────────┴────────────┐
             ▼                        ▼
       ┌──────────┐            ┌─────────────┐
       │ Whisper  │            │ GPT-4o-mini │
       │   API    │            │     API     │
       └──────────┘            └─────────────┘
```

---

## 4. Roadmap di sviluppo a 4 settimane

### Settimana 1 — Foundation + trascrizione
**Obiettivo:** l'utente può condividere un audio da WhatsApp e ottenere la trascrizione italiana.

**Giorni 1-2: Setup**
- [ ] Crea progetto Android Studio, package `com.vocaliai.app`
- [ ] Configura Gradle Version Catalog (`libs.versions.toml`)
- [ ] Setup Hilt, Compose, Retrofit, Room (tutte le dipendenze)
- [ ] Setup Git repo + `.gitignore` corretto per Android
- [ ] Setup backend proxy minimal su Fly.io (Node.js + Express, 1 endpoint `/transcribe`)
- [ ] Variabili ambiente, `.env.example`, secret management
- [ ] Configura signing config per debug (release dopo)

**Giorni 3-5: Share Intent + file access**
- [ ] Intent Filter in `AndroidManifest.xml`:
  ```xml
  <intent-filter>
      <action android:name="android.intent.action.SEND" />
      <category android:name="android.intent.category.DEFAULT" />
      <data android:mimeType="audio/*" />
  </intent-filter>
  ```
- [ ] `ShareReceiverActivity` che riceve l'audio e lo copia in cache app
- [ ] Gestione permessi runtime: `READ_MEDIA_AUDIO` (API 33+) o `READ_EXTERNAL_STORAGE` (API <33)
- [ ] Feature bonus: file picker nativo che apre direttamente `WhatsApp/Media/WhatsApp Voice Notes/` se esiste
- [ ] UI loader Compose minimale con stato

**Giorni 6-7: Pipeline trascrizione**
- [ ] `TranscriptionRepository` interface + implementazione `OpenAIWhisperRepository`
- [ ] Retrofit service con multipart upload del file audio
- [ ] Gestione errori tipizzata (sealed class `TranscriptionError`)
- [ ] Loader state con animazione elegante (Compose)
- [ ] Test end-to-end: WhatsApp → share → trascrizione mostrata

**Deliverable settimana 1:** Demo video di un vocale WhatsApp trascritto correttamente in italiano in <10 secondi.

### Settimana 2 — Summarization + persistence
**Obiettivo:** ogni audio processato viene riassunto, archiviato, e ricercabile.

**Giorni 8-10: Summarization**
- [ ] `SummaryRepository` con 3 modalità (`Breve`, `Dettagliato`, `Azioni`)
- [ ] Prompt engineering in italiano (vedi sezione 6)
- [ ] Streaming response via SSE (OkHttp EventSource) per UX progressiva
- [ ] Toggle UI per switchare modalità senza riprocessare l'audio

**Giorni 11-12: Room schema + FTS**
- [ ] Entity: `VocaleProcessato` (id, data, durata, trascrizione, riassunto, azioni JSON, contattoId, tono)
- [ ] Entity: `Contatto` (id, nome, telefono?)
- [ ] Entity: `Tag`
- [ ] `VocaleFts` virtual table per full-text search su trascrizione+riassunto
- [ ] DAO con query `@Query("SELECT * FROM vocale_fts WHERE vocale_fts MATCH :query")`
- [ ] Migration strategy pianificata

**Giorni 13-14: History UI**
- [ ] `LazyColumn` Compose con ricerca full-text in tempo reale
- [ ] Filter chips: oggi / settimana / tutti / per contatto
- [ ] Detail screen con tab trascrizione/riassunto/azioni (HorizontalPager)
- [ ] Swipe-to-delete con `ModalBottomSheet` di conferma
- [ ] Share action: condividi riassunto come testo

**Deliverable settimana 2:** App usabile per uso personale beta interno.

### Settimana 3 — Monetization + workflow integration
**Obiettivo:** paywall funzionante + integrazione Android nativa con Google Tasks/Calendar.

**Giorni 15-16: Billing**
- [ ] Setup Play Console: 2 prodotti subscription + offer (trial)
- [ ] Se usi **RevenueCat**: SDK init + paywall Compose
- [ ] Se usi **Google Play Billing nativo**: `BillingClient`, `purchasesUpdatedListener`, acknowledgment flow
- [ ] Paywall a schermata intera (NON modal fastidiosi)
- [ ] Entitlement check: `premium` → unlock tutto
- [ ] Free tier counter: 3 riassunti/mese con reset automatico (WorkManager periodico)
- [ ] Restore purchases flow (`queryPurchasesAsync`)

**Giorni 17-18: Azioni → Android nativo**
- [ ] `ActionExtractorUseCase` che parsa azioni da GPT JSON response
- [ ] Integrazione **Google Tasks** via Calendar Provider o Intent a Google Tasks
- [ ] Integrazione **Google Calendar** via `CalendarContract.Events`
- [ ] Prompt utente: "Aggiungi 'chiamare Luca domani' ai Tasks?"
- [ ] Gestione permessi: `WRITE_CALENDAR`, `READ_CALENDAR`
- [ ] Fallback: se utente non ha Google Tasks, usa Intent generico `ACTION_INSERT` che apre qualsiasi app calendar/reminders installata

**Giorni 19-21: Feature differenzianti**
- [ ] "Riassumi tutta la chat" — raggruppamento multi-vocale per contatto/giorno
- [ ] Risposta suggerita (bottone "Copia risposta" → clipboard + toast)
- [ ] Auto-tagging tono: amichevole/lavoro/urgente (GPT classification)
- [ ] **Android-exclusive:** Quick Settings Tile per trascrizione veloce (opzionale ma wow-factor)

**Deliverable settimana 3:** Internal testing track su Play Console pronto per 20-50 beta tester.

### Settimana 4 — Polish + launch
**Obiettivo:** approvata da Google e live sul Play Store italiano.

**Giorni 22-23: Polish UX**
- [ ] Onboarding a 3 step (con video demo integrato usando ExoPlayer)
- [ ] Empty states, loading states, error states curati
- [ ] Haptic feedback con `HapticFeedbackConstants`
- [ ] Dark mode completo + dynamic color (Material You, Android 12+)
- [ ] Accessibility: TalkBack su tutti gli elementi, `contentDescription` ovunque
- [ ] Edge-to-edge display (Android 15+ requirement)
- [ ] Predictive Back gesture support

**Giorni 24-25: Play Store assets**
- [ ] Screenshot in italiano (minimo 2, massimo 8, risoluzione 1080x1920+) — NO mockup inglese
- [ ] Feature graphic (1024x500)
- [ ] Promo video YouTube (30-60 secondi) — opzionale ma aumenta install rate
- [ ] Descrizione breve (80 char) e completa (4000 char) ottimizzate ASO (vedi sezione 7)
- [ ] Icon finale (512x512 PNG, design professionale, NON IA-generata banale)
- [ ] Privacy policy + Terms of Service (host su Notion/Carrd/sito tuo)

**Giorni 26-27: Closed/Open testing + bug fixing**
- [ ] Upload su Closed Testing track (consigliato, 12+ tester per 14+ giorni per passare requisito di Play Store)
- [ ] Reclutamento beta tester italiani (amici, X, Reddit r/Italia, r/italy)
- [ ] Iterazione su feedback più critico
- [ ] Performance tuning: cold start <2s, trascrizione <10s
- [ ] Crash-free user rate >99.5% (vedi Crashlytics)
- [ ] APK size check: <50MB ideale

**Giorni 28-30: Production submission**
- [ ] Build AAB (Android App Bundle, obbligatorio dal 2021)
- [ ] Data Safety form su Play Console (vedi sezione 8)
- [ ] Compilazione contenuto rating
- [ ] Target audience: 18+ (per evitare compliance COPPA/complicazioni)
- [ ] App Access: fornisci credenziali test se richiesto da reviewer
- [ ] Submit per review (Google review: 2-7 giorni tipicamente per app nuova)
- [ ] Press kit pronto per lanci coordinati
- [ ] Lista 30 creator/giornalisti italiani per outreach

**Deliverable settimana 4:** App live sul Play Store Italia.

**⚠️ NOTA CRITICA Play Store 2024+:**
Per nuovi developer personali, Google richiede **Closed Testing track con 12 tester per 14 giorni continuativi** prima di poter pubblicare in produzione. PIANIFICA QUESTO DALL'INIZIO DEL MESE, non aspettare la settimana 4. Per developer business (LLC/Srl) questo requisito non si applica.

---

## 5. Struttura del progetto

```
VocaliAI/
├── app/
│   ├── build.gradle.kts
│   ├── proguard-rules.pro
│   └── src/
│       ├── main/
│       │   ├── AndroidManifest.xml
│       │   ├── java/com/vocaliai/app/
│       │   │   ├── VocaliAiApplication.kt       # @HiltAndroidApp
│       │   │   ├── MainActivity.kt              # @AndroidEntryPoint
│       │   │   │
│       │   │   ├── core/
│       │   │   │   ├── di/                       # Hilt modules
│       │   │   │   │   ├── NetworkModule.kt
│       │   │   │   │   ├── DatabaseModule.kt
│       │   │   │   │   └── RepositoryModule.kt
│       │   │   │   ├── network/
│       │   │   │   │   ├── ApiService.kt
│       │   │   │   │   ├── NetworkError.kt
│       │   │   │   │   └── ErrorInterceptor.kt
│       │   │   │   ├── database/
│       │   │   │   │   ├── VocaliDatabase.kt
│       │   │   │   │   ├── dao/
│       │   │   │   │   │   ├── VocaleDao.kt
│       │   │   │   │   │   └── ContattoDao.kt
│       │   │   │   │   └── entity/
│       │   │   │   │       ├── VocaleEntity.kt
│       │   │   │   │       ├── VocaleFtsEntity.kt
│       │   │   │   │       ├── ContattoEntity.kt
│       │   │   │   │       └── TagEntity.kt
│       │   │   │   ├── billing/
│       │   │   │   │   └── BillingManager.kt
│       │   │   │   ├── analytics/
│       │   │   │   │   └── AnalyticsTracker.kt
│       │   │   │   └── util/
│       │   │   │       ├── AudioFileUtils.kt
│       │   │   │       └── DateFormatters.kt
│       │   │   │
│       │   │   ├── data/
│       │   │   │   ├── repository/
│       │   │   │   │   ├── TranscriptionRepository.kt
│       │   │   │   │   ├── TranscriptionRepositoryImpl.kt
│       │   │   │   │   ├── SummaryRepository.kt
│       │   │   │   │   ├── SummaryRepositoryImpl.kt
│       │   │   │   │   └── VocaleRepository.kt
│       │   │   │   └── remote/dto/
│       │   │   │       ├── TranscriptionResponse.kt
│       │   │   │       └── SummaryResponse.kt
│       │   │   │
│       │   │   ├── domain/
│       │   │   │   ├── model/
│       │   │   │   │   ├── Vocale.kt
│       │   │   │   │   ├── Summary.kt
│       │   │   │   │   ├── Azione.kt
│       │   │   │   │   └── Tono.kt
│       │   │   │   └── usecase/
│       │   │   │       ├── ProcessAudioUseCase.kt
│       │   │   │       ├── ExtractActionsUseCase.kt
│       │   │   │       └── AddToCalendarUseCase.kt
│       │   │   │
│       │   │   ├── ui/
│       │   │   │   ├── theme/
│       │   │   │   │   ├── Theme.kt
│       │   │   │   │   ├── Color.kt
│       │   │   │   │   └── Typography.kt
│       │   │   │   ├── navigation/
│       │   │   │   │   └── VocaliNavHost.kt
│       │   │   │   ├── share/
│       │   │   │   │   ├── ShareReceiverActivity.kt
│       │   │   │   │   └── ShareProcessingScreen.kt
│       │   │   │   ├── home/
│       │   │   │   │   ├── HomeScreen.kt
│       │   │   │   │   ├── HomeViewModel.kt
│       │   │   │   │   └── components/
│       │   │   │   ├── detail/
│       │   │   │   │   ├── VocaleDetailScreen.kt
│       │   │   │   │   └── VocaleDetailViewModel.kt
│       │   │   │   ├── paywall/
│       │   │   │   │   ├── PaywallScreen.kt
│       │   │   │   │   └── PaywallViewModel.kt
│       │   │   │   ├── onboarding/
│       │   │   │   │   └── OnboardingScreen.kt
│       │   │   │   └── settings/
│       │   │   │       └── SettingsScreen.kt
│       │   │   │
│       │   │   └── worker/
│       │   │       └── MonthlyResetWorker.kt       # WorkManager per reset free tier
│       │   │
│       │   └── res/
│       │       ├── drawable/
│       │       ├── mipmap-anydpi-v26/              # Adaptive icons
│       │       ├── values/
│       │       │   ├── strings.xml                  # Fallback EN
│       │       │   └── themes.xml
│       │       ├── values-it/
│       │       │   └── strings.xml                  # PRIMARY italiano
│       │       └── xml/
│       │           ├── file_paths.xml
│       │           └── data_extraction_rules.xml
│       │
│       ├── test/                                    # Unit tests
│       └── androidTest/                             # Instrumentation tests
│
├── backend/
│   ├── server.js                                    # Express proxy
│   ├── routes/
│   │   ├── transcribe.js
│   │   └── summarize.js
│   ├── middleware/
│   │   ├── rateLimit.js
│   │   └── playReceiptValidator.js
│   ├── package.json
│   └── fly.toml
│
├── gradle/
│   └── libs.versions.toml                           # Version Catalog
├── build.gradle.kts                                 # Project-level
├── settings.gradle.kts
├── .gitignore
├── README.md
└── CLAUDE.md                                        # Questo file
```

---

## 6. Prompt engineering per GPT-4o-mini

### 6.1 System prompt base (per riassunto)

```
Sei un assistente italiano che riassume messaggi vocali di WhatsApp.
Rispondi SEMPRE in italiano, con tono naturale e caldo ma conciso.
Non usare mai termini inglesi se esiste l'equivalente italiano.
Non aggiungere preamboli tipo "Ecco il riassunto:" — vai dritto al contenuto.
Non inventare informazioni non presenti nell'audio.
Se il vocale è confuso o incomprensibile, dillo esplicitamente.
```

### 6.2 Prompt per modalità "breve"
```
Riassumi questo messaggio vocale in UNA frase di massimo 20 parole.
Cattura solo l'essenza.

Trascrizione: {transcript}
```

### 6.3 Prompt per modalità "dettagliato"
```
Riassumi questo messaggio vocale in un paragrafo di 2-4 frasi.
Mantieni il tono originale (amichevole, formale, urgente).
Preserva dettagli concreti importanti (nomi, date, luoghi).

Trascrizione: {transcript}
```

### 6.4 Prompt per modalità "azioni + punti chiave"
```
Analizza questo messaggio vocale ed estrai:

1. **Punti chiave** (max 3 bullet, ciascuno ≤15 parole)
2. **Azioni richieste** — SOLO azioni concrete che l'ascoltatore deve fare.
   Formato JSON array: [{"azione": "...", "quando": "YYYY-MM-DD o null", "urgenza": "alta|media|bassa"}]
3. **Tono generale** — uno tra: "amichevole", "lavoro", "urgente", "informativo", "emotivo"

Rispondi SOLO con un oggetto JSON valido:
{
  "punti_chiave": ["...", "..."],
  "azioni": [{"azione": "...", "quando": "...", "urgenza": "..."}],
  "tono": "..."
}

Trascrizione: {transcript}
```

### 6.5 Prompt per risposta suggerita
```
L'utente ha ricevuto questo messaggio vocale. Suggerisci UNA risposta testuale
breve (≤30 parole), naturale, appropriata al tono.
NON essere formale se il messaggio non lo è.
NON usare emoji a meno che il mittente ne abbia usati.

Messaggio ricevuto: {transcript}
Tono rilevato: {tono}
```

---

## 7. Play Store Optimization (ASO)

### 7.1 Nome app
**Primary:** `Vocali AI`
**Subtitle (short desc, 80 char):** `Riassumi i vocali WhatsApp con l'intelligenza artificiale in italiano`

### 7.2 Keywords
Google Play NON ha un campo keywords dedicato (a differenza di App Store). Le keyword vanno nel **titolo** (30 char) e nella **descrizione completa**. Strategie:

**Titolo ottimizzato (max 30 char):**
- Opzione A: `Vocali AI - Riassumi WhatsApp` (29 char)
- Opzione B: `Vocali AI: Trascrivi & Riassumi` (31 char, troppo lungo)

**Keywords da inserire organicamente nella descrizione:**
- vocali, whatsapp, trascrivere, riassumere, audio, ai, intelligenza artificiale, messaggi vocali, trascrizione, assistente vocale

### 7.3 Descrizione completa (prima riga = conversion critical)
```
Stanco di ascoltare vocali infiniti su WhatsApp? Vocali AI li riassume per te in 5 secondi con l'intelligenza artificiale.

━━━━━━━━━━━━━━━━━━━
COSA PUOI FARE:
━━━━━━━━━━━━━━━━━━━

📝 RIASSUMI all'istante — condividi un vocale WhatsApp e ottieni il riassunto in italiano
✅ AZIONI automatiche — date, appuntamenti e task direttamente in Google Calendar e Tasks
💬 RISPOSTE suggerite — rispondi senza riascoltare nulla
🔍 CERCA ovunque — ritrova qualsiasi vocale del passato con una parola
🇮🇹 TUTTO IN ITALIANO — AI addestrata per capire le sfumature del nostro modo di parlare

━━━━━━━━━━━━━━━━━━━
PERFETTO PER:
━━━━━━━━━━━━━━━━━━━

• Chi riceve vocali al lavoro e non può ascoltarli
• Chi è in gruppi famiglia con 50+ messaggi al giorno
• Chi vuole archiviare conversazioni importanti
• Chi odia i vocali da 4 minuti

━━━━━━━━━━━━━━━━━━━
COME FUNZIONA:
━━━━━━━━━━━━━━━━━━━

1. Apri WhatsApp e tieni premuto sul vocale
2. Tocca "Condividi" e seleziona Vocali AI
3. In pochi secondi ottieni trascrizione e riassunto

━━━━━━━━━━━━━━━━━━━
PRIVACY AL PRIMO POSTO
━━━━━━━━━━━━━━━━━━━

I tuoi vocali non vengono mai salvati sui nostri server.
Trascrizione e riassunto avvengono in tempo reale, poi tutto resta SOLO sul tuo telefono Android.

━━━━━━━━━━━━━━━━━━━
ABBONAMENTI:
━━━━━━━━━━━━━━━━━━━

• Piano Mensile: €4,99/mese (3 giorni gratis)
• Piano Annuale: €29,99/anno (7 giorni gratis — risparmi 50%)

Gratis: 3 riassunti al mese.

Puoi disdire in qualsiasi momento dalle impostazioni di Google Play.

Termini: [link] | Privacy: [link]
```

### 7.4 Categoria Play Store
- **Primary:** Produttività
- **Tags:** AI assistant, Note taking, Transcription

---

## 8. Checklist pre-submission Play Store

- [ ] Tutti i testi in italiano nel `values-it/strings.xml`, inglese come fallback in `values/`
- [ ] Privacy Policy URL attivo e menziona OpenAI come data processor
- [ ] Terms of Service URL attivo
- [ ] **Data Safety form** su Play Console compilato correttamente:
  - Audio files: raccolti, NON condivisi, processati in tempo reale e non memorizzati
  - Device or other IDs: raccolti per analytics
  - App activity: raccolto per analytics
  - Encryption in transit: Yes
  - Data deletion request: fornisci email o form
- [ ] **Content rating** completato (IARC questionnaire) → dovrebbe risultare PEGI 3 / Everyone
- [ ] No crash al primo launch senza rete
- [ ] Paywall NON blocca onboarding completamente (Google richiede free features visibili)
- [ ] Bottone "Ripristina acquisti" visibile e funzionante
- [ ] Link ai termini visibile sul paywall
- [ ] Trial period chiaramente indicato ("3 giorni gratis, poi €4,99/mese, rinnovo automatico")
- [ ] Cancellation instructions visibili in Settings + link diretto a Play Store subscriptions
- [ ] Screenshot senza elementi ingannevoli o feature non presenti
- [ ] Target API level aggiornato (API 34 Android 14 minimo per nuovi app nel 2025)
- [ ] `android:allowBackup="false"` O regole `data_extraction_rules.xml` ben configurate
- [ ] ProGuard/R8 abilitato in release build + keep rules per Retrofit/Room
- [ ] AAB firmato con upload key + Play App Signing abilitato
- [ ] Test su device fisico minimo 2 modelli diversi (es: Pixel + Samsung con One UI)
- [ ] Closed Testing con 12 tester per 14 giorni completato (requirement per personal developer account nuovi)
- [ ] Declarations completate: Government apps, Financial apps, Health apps = No

---

## 9. Note operative per Claude Code

### 9.1 Come lavorare insieme
- **Scrivi codice production-ready**, non pseudocodice. Include error handling, logging, e KDoc dove serve.
- **Un file alla volta**, con path completo, così posso copiarlo direttamente in Android Studio.
- **Testa mentalmente ogni funzione** prima di scriverla: edge cases, null handling, thread safety, coroutine scope corretto.
- **Proponi sempre alternative** quando una decisione architetturale è discutibile.
- **Segnala i rischi** quando usi API alpha/beta Compose o pattern non canonici.

### 9.2 Convenzioni di codice Kotlin
- **Code style:** Kotlin Official (configurato in `.editorconfig`)
- Nomi di classi, funzioni e proprietà **in inglese** (convenzione Kotlin/Java).
- **Stringhe user-facing sempre in Italiano**, gestite via `strings.xml` in `values-it/`.
- KDoc comments in inglese.
- `// region Xxx` / `// endregion` per organizzare sezioni lunghe.
- Preferisci `data class` e `sealed class/interface` dove possibile.
- `suspend fun` + Flow over callback.
- `@Immutable` e `@Stable` Compose annotations dove rilevanti.
- NO `!!` (non-null assertion) — usa `?.let`, `requireNotNull`, o sealed error handling.

### 9.3 Convenzioni Compose
- Composable function names in PascalCase (`HomeScreen`, non `homeScreen`)
- Modifier sempre come primo parametro opzionale con default `Modifier`
- State hoisting: stateless composables sotto, stateful ViewModels sopra
- `collectAsStateWithLifecycle()` invece di `collectAsState()` per sicurezza lifecycle
- NO logica in `@Composable` — tutto in ViewModel
- Preview functions per ogni composable pubblico

### 9.4 Quando non sei sicuro
- Chiedi prima di fare assunzioni su package name, keystore, Google Play account.
- Suggerisci ricerca su documentazione Android developer se una API sembra instabile.
- NON inventare metodi RevenueCat/OpenAI/Billing Library — verifica sempre con docs ufficiali.
- Attenzione particolare a permessi runtime che sono cambiati: `READ_EXTERNAL_STORAGE` è deprecato da API 33, usa `READ_MEDIA_AUDIO`.

### 9.5 Priorità quando le cose vanno storte
1. **Funzionalità core** (trascrizione + riassunto) > feature secondarie
2. **Stabilità** > features nuove
3. **UX italiana impeccabile** > features tecniche avanzate
4. **Time to market** > perfezione del codice
5. **Compatibilità Samsung/Xiaomi/Oppo** > Pixel-only polish (l'Italia usa tanti Samsung)

### 9.6 File da creare IMMEDIATAMENTE all'inizio
1. `README.md` — setup instructions
2. `.gitignore` — con `local.properties`, `*.keystore`, `.env`, `build/`, `.gradle/`, `.idea/` (parziale)
3. `.env.example` — variabili ambiente template per backend
4. `CLAUDE.md` — questo file, come memoria persistente
5. `gradle/libs.versions.toml` — Version catalog centralizzato
6. `backend/server.js` — proxy minimo funzionante
7. Progetto Android Studio base con Hilt + Compose + Navigation già cablati
8. `TranscriptionRepository.kt` — prima implementazione

### 9.7 Gotchas Android specifici da evitare
- **Scoped Storage (Android 10+)**: NON assumere di poter scrivere ovunque. Usa `MediaStore` o `Context.filesDir`.
- **Foreground services**: se fai trascrizioni lunghe in background, serve foreground service type `dataSync` (Android 14+).
- **Battery optimization**: non chiedere `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` — Play Store rejeta se non strettamente necessario.
- **Notification permission (Android 13+)**: richiedi `POST_NOTIFICATIONS` runtime, non assumere concesso.
- **Edge-to-edge (Android 15)**: è OBBLIGATORIO per target API 35. Usa `enableEdgeToEdge()` in `onCreate`.
- **Predictive back**: supportalo con `BackHandler` Compose o `OnBackPressedCallback`.

---

## 10. Definition of Done — v1.0

L'app è pronta per il lancio quando:

- ✅ Un utente può installare l'app, fare onboarding, e processare il suo primo vocale in <90 secondi
- ✅ La trascrizione è accurata (>95% word accuracy su italiano parlato)
- ✅ Il riassunto cattura il senso del vocale nel 90% dei casi testati
- ✅ Le azioni vengono estratte correttamente nell'80% dei casi testati
- ✅ La subscription si attiva correttamente e il paywall rispetta le policy Google Play
- ✅ Zero crash su 50 sessioni di test su device Samsung + Pixel
- ✅ Crash-free user rate >99.5% in Crashlytics dopo closed testing
- ✅ APK size <50MB, cold start <2 secondi
- ✅ Play Store screenshots, descrizione, e feature graphic sono in italiano nativo
- ✅ Privacy policy e terms live e linkati correttamente
- ✅ Data Safety form compilato onestamente
- ✅ Closed Testing 14 giorni + 12 tester completato (se personal account)
- ✅ Analytics funnel tracciato: install → onboarding complete → first summary → paywall view → trial start → paid conversion

---

**Claude Code, ora inizia dal passo 1 della Settimana 1: setup del progetto Android Studio + backend proxy minimale. Mostrami:**
1. **Il `libs.versions.toml` completo** con tutte le dipendenze che useremo
2. **La struttura directory esatta** che devo creare
3. **Il primo file di codice critico**: `VocaliAiApplication.kt` + `MainActivity.kt` scheletro

**Iniziamo.**
