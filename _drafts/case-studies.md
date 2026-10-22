# Case-study drafts (v1)

Five case studies for the redesigned portfolio. Each follows the same shape:
**Context → Constraints → Options weighed → Decision & architecture → Result → What I'd do differently**, with an evidence table.

How to read the tags:
- `[PDF:n]`: `Mukesh_Kumar_Sharma_iOS_Resume.pdf`, line n of the text extract (the current, canonical resume)
- `[DOCX]`: `MukeshKumarSharma_iOS_Developer-2025 (1).docx` (the long project-by-project resume; `MUKESH_KUMAR_SHARMA_RESUME_iOS_DEVELOPER.docx` is an older copy of the same content and adds nothing new)
- `[SITE]`: the current `index.html` (the existing streaming-assistant write-up)
- `[confirm]`: **my inference, not in any source.** It's plausible reasoning written in your voice. Keep it, correct it, or delete it. Nothing tagged `[confirm]` should go live until you've checked it.

> This folder starts with `_`, so GitHub Pages (Jekyll) won't publish it.

---

## 01 / Fintech AI assistant: streaming chat in a flagship investing app

**Card headline:** *Answers that appear as they're written, without a single data race.*
**One-line how:** Built the real-time LLM streaming layer and chat surface for a generative-AI investing assistant across 8 Swift/SwiftUI modules.
**Chips:** `8 modules` · `8 response types` · `iOS 17 → 26`
**Stack:** Swift 6 · SwiftUI · Combine · actors · Alamofire streaming · Swift Testing
**Role & dates:** Senior iOS Engineer (contract) · Mar 2026 – Aug 2026 · Toronto (hybrid)
**Client naming:** keep it generic ("a top-tier investment firm's flagship app"), to match commit `2f71013`.

### Context
A generative-AI investing assistant was being added to a flagship iOS app used by retail investors. Responses needed to stream token by token, as they do in consumer chat apps, and render more than plain text: tables, lists and charts. [PDF:29–35] [SITE]

### Constraints
- Financial UI, so a half-rendered table or a chart with an out-of-range value is worse than no answer. [SITE: "validation at the parse boundary"]
- The backend schema was still evolving, and new response types could arrive before the app knew about them. [SITE: "graceful degradation on unknown types"]
- Minimum iOS 17, which ruled out iOS 18-only APIs for onboarding tips. [PDF:36]
- Mixed codebase: some host screens were SwiftUI and some legacy UIKit. [SITE: module 01]
- Staged rollout: phase 1 shipped to a small slice of users first. [PDF:30] [SITE: "0.1% Phase 1 rollout"]

### Options weighed
1. **Wait for the full response, then render it.** Simple, but users would stare at a spinner. `[confirm]` Rejected for perceived latency.
2. **Server-Sent Events / generic SSE library.** `[confirm]` Did the backend offer SSE? Your skills line lists both SSE and NDJSON [PDF:19–20], so say why NDJSON won here.
3. **Newline-delimited JSON over an HTTP data stream, parsed by a custom tokenizer.** ✅ Chosen. Each line is a complete, typed component, so the client can render a table as one unit instead of guessing from partial markdown. [PDF:31–32] [SITE: module 07]

On concurrency, the first version `[confirm]` processed chunks on a callback queue, and races showed up under load. Moving the tokenizer into an **actor**, adding a custom **@globalActor** for serialized stream processing, and enforcing **@MainActor** UI boundaries plus full **Sendable** conformance removed them. [PDF:32–33] [SITE: module 06]

### Decision & architecture
Diagram to draw (inline SVG):
```
Alamofire DataStreamRequest ──bytes──▶ actor JSONLTokenizer ──(buffer → split on \n → complete objects)──▶
  PassthroughSubject<Component> ──▶ Orchestration service (@globalActor) ──▶ ChatViewModel (@MainActor) ──▶ SwiftUI Chat UI package
                                         │
                       Decodable on "component" discriminator: text | table | ordered/unordered list | line/bar/pie chart | suggested intents
                       unknown type → skip / fallback (graceful degradation)
```
- **Contract-first:** you co-designed the streaming schema with the backend and AI-platform teams and aligned design on it. [PDF:34–35]
- **8 modules as SPM packages**, each ViewModel paired with a protocol and a mock, so everything could be tested without the network. [PDF:38–39]
- **Reusable chat UI package (50+ files)**, later reused for live-agent chat. [SITE: module 08]
- **Coach-mark framework (17 files)** modeled on TipKit and back-deployed to iOS 17, then adopted by other engineers. [PDF:36–37] [SITE: module 02]
- **Claude commands** that run a first-pass code review against team guidelines. [PDF:39–40] [SITE]

### Result
- Shipped through the phase 1 production release (staged rollout). [PDF:30]
- Data races under load eliminated. [PDF:32]
- Coach-mark framework adopted by other engineers and validated through iOS 26. [PDF:37]
- `[confirm]` Any number to add, e.g. time to first token on screen, crash-free rate, or number of unit tests on the tokenizer?

### What I'd do differently `[confirm]`
Version the stream schema explicitly (a `schemaVersion` field) from day one. Graceful degradation hides unknown types, but it also hides when the client is out of date.

---

## 02 / Enterprise payments: taking payments with no signal

**Card headline:** *Field sales kept taking payments with no signal, and offline failures dropped to effectively zero.*
**One-line how:** Led the 0→1 SwiftUI checkout hub with an offline transaction queue that syncs on reconnect.
**Chips:** `72-hour offline retention` · `3 apps on one SPM package` · `20–50% faster releases`
**Stack:** SwiftUI · Combine · async/await · Core Data · BGTaskScheduler · Bluetooth/MFi · Fastlane · Xcode Cloud
**Role & dates:** Lead iOS Engineer · Tech Mahindra · Dec 2024 – Mar 2026 [PDF:41–42]
**Naming:** the PDF calls it "an enterprise payments platform"; the DOCX names it **TruPOS**. `[confirm]` Which name can go public?

### Context
The client needed one payments app to act as the checkout hub for several business lines. Field sales staff take card payments at customer sites, often where there's little or no signal. [PDF:43–47] [DOCX: TruPOS]

### Constraints
- A failed payment is lost revenue and a bad customer moment, and these payments happen in low-signal areas. [DOCX: "intermittent POS connectivity… low-signal areas"]
- Hardware: Bluetooth card readers and MFi accessories that must keep working in the background. [PDF:47] [DOCX]
- Several apps needed the same payment, receipt and deep-link logic. [PDF:48]
- Integration with external apps (e.g. Field Service 365) via universal links and URL schemes. [DOCX]

### Options weighed
1. **Block checkout when offline.** Simplest, but it fails the core use case. `[confirm]`
2. **Hold failed requests in memory and retry.** `[confirm]` Rejected because the app can be killed or the device rebooted before signal returns.
3. **Durable queue in Core Data, drained by background tasks, with a bounded retention window.** ✅ Chosen. [PDF:45–46]

`[confirm]` Why **72 hours**? My guess is it covers a weekend of field work and matches the processor's limit for submitting offline authorizations. Please give the real reason; it's the most interesting detail in this story.
`[confirm]` Idempotency: how did you avoid charging the same payment twice on retry (client-generated transaction ID? server dedupe?). Interviewers will ask.

### Decision & architecture
Diagram to draw:
```
Checkout (SwiftUI, MVVM) ─▶ PaymentService ─online?─yes─▶ API ─▶ live status polling (Combine + URLSession) ─▶ UI
                                     └─no──▶ Core Data queue (status, attempts, createdAt ≤ 72h)
                                                  ▲            │
                         BGTaskScheduler / reconnect trigger ──┘   retry layer with backoff ─▶ API ─▶ mark synced
Shared SPM package: payments · receipts · deep linking ──▶ used by 3 internal apps
```
- MVVM with dynamic, form-based SwiftUI screens for checkout, history and merchant onboarding. [PDF:43–44] [DOCX]
- Bluetooth card readers plus a retry layer for flaky accessory connections. [PDF:47]
- App Intents shortcuts for frequent payment operations. [DOCX]
- CI/CD owned end to end: Fastlane, GitHub Actions and Xcode Cloud automate TestFlight and App Store releases. [PDF:49–50]

### Result
- Offline transaction failures reduced to effectively zero. [PDF:46]
- The shared Swift package was adopted by 3 internal apps, ending per-app rewrites. [PDF:48]
- Release cycle time cut by 20–50%. [PDF:49]
- 80% unit-test coverage on business logic; retain cycles fixed and cold start and memory pressure improved with Instruments. [PDF:50–51]
- `[confirm]` A cold-start number (before → after) would make the Instruments claim concrete.

### What I'd do differently `[confirm]`
Surface the queue to the user, for example an "3 payments waiting to sync" badge, earlier. Trust in an offline system comes from seeing it work.

---

## 03 / Dr Godly: on-device clinical AI where patient audio never leaves the phone

**Card headline:** *Clinical AI that never sends a patient's voice to a server.*
**One-line how:** Fine-tuned Apple Foundation Models with LoRA adapters for clinical vocabulary and built live two-party transcription into video consults, shipped solo across 6+ repositories.
**Chips:** `100% on-device inference` · `6+ repos, solo` · `limited production pilot`
**Stack:** Apple Foundation Models · LoRA · Apple Speech · Twilio Video · SwiftUI · Node.js/TypeScript on App Engine
**Role & dates:** Founder & Lead Engineer · Jan 2024 – present [PDF:54–55]
**Naming:** the PDF says **Dr Godly**, the site says **Doctor AI**, and the DOCX describes **MedPal AI** (Jun 2023 – Jan 2025, under Tech Mahindra). `[confirm]` Are these the same product line? Pick one public name and make the dates match everywhere. A sharp interviewer will notice the mismatch.

### Context
Doctors spend consult time writing notes. An AI that transcribes and summarizes helps, but in healthcare, sending patient audio to a third-party model raises privacy problems. [PDF:56–57]

### Constraints
- Patient audio and transcripts should never reach a server for AI processing. [PDF:57]
- General-purpose models handle clinical vocabulary poorly. [PDF:56: "LoRA adapters for clinical vocabulary"]
- Solo founder: iOS, Android, a shared package and the backend are all one person. [PDF:60–61]
- Real-time: transcription has to keep up with a live video call between two people. [PDF:58–59]

### Options weighed
1. **Cloud LLM (OpenAI or similar) for transcription and summaries.** Fastest to build. Your earlier telehealth work used the OpenAI client [DOCX: MedPal]. `[confirm]` Rejected for privacy and compliance; this contrast between your earlier approach and this one is a strong story.
2. **Base on-device model only.** Private, but weak on clinical terms. `[confirm]`
3. **On-device Foundation Model plus a LoRA adapter tuned for clinical vocabulary and note summarization, with default safety guardrails.** ✅ Chosen. [PDF:56–57]

### Decision & architecture
Diagram to draw:
```
Twilio Video call ──▶ Apple Speech (doctor track) ─┐
                 └──▶ Apple Speech (patient track) ─┴─▶ speaker-tagged transcript (on device)
                                                         ─▶ Foundation Model + clinical LoRA adapter ─▶ note summary
                                                         ─▶ TTS playback
Node.js backend (App Engine): auth · sessions · orchestration only — no patient audio or transcripts
```
`[confirm]` "LLM orchestration" sits on the backend [PDF:61] while inference runs on device. Clarify what the backend orchestrates, so the privacy claim stays clean.

### Result
- Running as a limited production pilot with early customers. [PDF:55]
- Full stack shipped solo: iOS client, Android companion, shared Swift package and Node.js backend. [PDF:60–61]
- `[confirm]` Pilot numbers: how many doctors, how many consults, minutes saved per note, or transcription accuracy on clinical terms before vs after LoRA?

### What I'd do differently `[confirm]`
Build an evaluation set of real (consented, de-identified) clinical phrases before fine-tuning, so each adapter version can be scored rather than judged by feel.

---

## 04 / NEOM Smart City: one core framework under ~20 apps

**Card headline:** *One framework, about 20 apps, and no team rebuilding networking or auth.*
**One-line how:** Designed the core SPM framework (networking, OAuth, SSL pinning, shared UI) and shipped the Smart City app's first App Store release.
**Chips:** `~20 adopting apps` · `first App Store release` · `3 engineers mentored`
**Stack:** Swift · SwiftUI · SPM · Apollo iOS (GraphQL) · OAuth 2.0 · SSL pinning · OpenAI · Twilio · Apple Speech
**Role & dates:** Senior / Lead iOS Engineer · NEOM via Tech Mahindra · Jun 2023 – Jan 2024 [PDF:62–63]

### Context
NEOM was building many apps for residents, visitors and service providers. Without a shared foundation, each team would re-implement networking, auth and security on its own. [PDF:64–66]

### Constraints
- Security baseline (OAuth, SSL pinning) had to be identical across every app. [PDF:65]
- Many consuming teams, so breaking changes would affect ~20 apps at once. [PDF:66: "interface contracts and versioning strategy"]
- GraphQL backends needed type safety on the client. [PDF:66]

### Options weighed
1. **Copy-paste a template project per app.** Fast at first, then the security fixes drift apart. `[confirm]`
2. **CocoaPods / binary framework.** `[confirm]` You'd used CocoaPods for TD's foundation framework [PDF:86]; say why SPM won here (first-party Xcode support, no extra tooling, source visibility).
3. **SPM framework with explicit interface contracts and semantic versioning.** ✅ Chosen. [PDF:65–66]

### Decision & architecture
Diagram to draw:
```
            ┌──────────── NEOM Core (SPM, semver) ─────────────┐
            │ Networking · OAuth 2.0 · SSL pinning · Apollo GraphQL · Shared UI │
            └───────────────────────────────────────────────────┘
               ▲            ▲            ▲              ▲
          Smart City    App 2        App 3   …     App ~20
             app (+ AI assistant: OpenAI + LangChain prompts · Twilio A/V · Apple Speech)
```
- `[confirm]` How was versioning enforced: tagged releases, a changelog, a deprecation window?
- In-app assistant for utilities, transport and support. [PDF:67–68]

### Result
- Smart City app's first App Store release shipped. [PDF:64]
- Core framework adopted by ~20 internal apps. [PDF:65–66]
- Mentored 3 engineers and led the team to production launch. [PDF:69–70]

### What I'd do differently `[confirm]`
Publish a small sample app alongside the framework from week one. Adoption by ~20 teams is mostly a documentation problem.

---

## 05 / Kroger: one release pipeline for four banner apps

**Card headline:** *The build, sign and deploy pipeline the whole iOS org shipped with.*
**One-line how:** Authored the CI/CD pipelines used across 4 Kroger banner apps and delivered the Gamification module to the full customer base.
**Chips:** `4 banner apps` · `whole iOS org adopted` · `v1.0 → v2.0`
**Stack:** Fastlane · GitHub Actions · Ruby · shell · Node.js · Firebase Remote Config · Crashlytics · Grafana
**Role & dates:** Senior / Lead iOS & Release Engineer · Kroger via Tech Mahindra · Jan 2023 – Jun 2023 [PDF:71–72]

### Context
Kroger runs one codebase family across several grocery banners (Kroger, Ralphs, Fred Meyer, Harris Teeter). Each banner is a separate App Store app with its own signing and release. [PDF:73–76]

### Constraints
- 4 apps × signing identities × environments (dev / prod). [PDF:75–76] [DOCX: "Dev and production"]
- A new feature (Gamification) going to the full customer base, so rollout had to be reversible. [PDF:73–77]

### Options weighed
1. **Manual Xcode archives per banner.** Slow and error-prone. `[confirm]`
2. **Per-banner scripts.** Duplicated logic that drifts. `[confirm]`
3. **One parameterized pipeline (Fastlane lanes + GitHub Actions) with the banner as an input.** ✅ `[confirm]` That this was the shape; the resume says "adopted by the entire iOS org across 4 Kroger banner apps" [PDF:75–76].

For feature risk, Gamification shipped behind **Firebase Remote Config flags with A/B experiments**, monitored in Grafana and Crashlytics. [PDF:77]

### Decision & architecture
Diagram to draw:
```
PR ─▶ GitHub Actions ─▶ fastlane lane(banner: kroger|ralphs|fredmeyer|harristeeter, env: dev|prod)
                         ├─ build ─▶ sign (per-banner identity) ─▶ TestFlight / App Store
                         └─ flags via Remote Config ─▶ A/B ─▶ Crashlytics + Grafana
```

### Result
- Pipeline adopted by the entire iOS org across 4 banner apps. [PDF:75–76]
- Gamification v1.0 and v2.0 released to the full customer base across all four banners. [PDF:73–74]
- `[confirm]` Release time before → after, or releases per month?

### What I'd do differently `[confirm]`
Add automated screenshot diffs per banner. Most multi-brand bugs are visual (the wrong logo or colour), and only a human was catching them.

---

## Not chosen (alternates, if you'd rather swap one in)
- **TD Bank (2021–22):** Face ID/Touch ID sign-in, jailbreak and tamper hardening, and the iOS Banking Foundation Framework on CocoaPods. A strong security story but thin on numbers. [PDF:83–86] [DOCX]
- **GroAssist (Pfizer, 2015–16):** sync conflict resolution when one child has several caregivers on several devices. An early proof that you think about data correctness. [DOCX: Project 10]
- **Gap (2022):** Notification System built from scratch across 4 storefronts. [PDF:80–81] [DOCX]
- **Penny:** goes in **Products**, not here (already on the site with TestFlight).

## Inconsistencies to fix before anything goes live
1. **Dr Godly / Doctor AI / MedPal AI:** three names, and the dates overlap with the DOCX's Tech Mahindra telehealth project. (Case 03)
2. **Where the OpenAI/LangChain/Twilio work happened:** the PDF puts it at NEOM [PDF:67–68]; the DOCX puts it under MedPal [DOCX: Project 2].
3. **Years of experience:** the PDF says 15+, the site says fourteen, and the older DOCX says 12+ / 14.
4. **TruPOS vs "enterprise payments platform":** decide what's public. (Case 02)
