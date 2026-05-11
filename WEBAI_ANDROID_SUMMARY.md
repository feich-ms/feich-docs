# Web AI APIs on Android — Status & Recommendations

## 1. Language Detection API

### Current Status

| Platform | Engine | Status |
|----------|--------|--------|
| Upstream Desktop | TFLite model | Stable |
| Upstream Android | TFLite model (same as desktop) | Test |
| Edge Desktop | TFLite model (same as upstream) | Stable |
| Edge Android | TFLite model | `BUILD.gn` changes needed |

### Alternatives for Android

#### Option A: MLKit Language Identification

Use `com.google.mlkit:language-id` via Google Play Services.

| Pros | Cons |
|------|------|
| Easy to integrate (similar pattern to MLKit Translate) | Adds unnecessary GMS dependency |
| | Native TFLite solution already exists and works |
| | Cross-process call to GMS — higher latency |
| | Additional native library to package |

#### Option B: Native TFLite (Already Implemented)

The existing TFLite-based language detection that works on desktop already runs on Android — all infrastructure is in place.

| Pros | Cons |
|------|------|
| Already works — no new code needed | Only blocked by feature flag |
| Same model/quality as desktop | |
| Runs in-process (renderer) — lowest possible latency | |
| No additional dependencies | |
| No GMS requirement (works on AOSP devices) | |

### Model Delivery

| Browser | Delivery Mechanism |
|---------|-------------------|
| Chrome | Optimization Guide server |
| Edge | `EdgeComponentModelRegistry` (component-based, immediate local load) |

**Edge-specific code** (in `components/language_detection/core/browser/language_detection_model_service.cc`):
- Guarded by `BUILDFLAG(MICROSOFT_EDGE_BRANDING) && defined(ENABLE_EDGE_LLM)`
- At startup, checks `EdgeComponentModelRegistry::GetModelPath(OPTIMIZATION_TARGET_LANGUAGE_DETECTION)`
- If model is already available locally, loads it immediately — no server round-trip needed

For Edge Android, the model would be delivered via `EdgeComponentModelRegistry` (Edge's component updater infrastructure), so it does **not** depend on Google's Optimization Guide server.

### Verification

Verified working on a physical Android device (Pixel 10) with Chromium debug build:
1. Changed `"Android": "test"` → `"Android": "stable"` in `runtime_enabled_features.json5`
2. Used `--optimization-guide-model-override=OPTIMIZATION_TARGET_LANGUAGE_DETECTION:<path>` to provide the TFLite model locally (needed only for Chromium builds without Google model server)
3. `ai.languageDetector.create()` + `detect()` works end-to-end

Also verified on Edge Android build with BUILD.gn changes in [commit 939fca1f6877](https://dev.azure.com/microsoft/Edge/_git/chromium.src/commit/939fca1f687707e8f791d2d62315382afb22c3a5?refName=refs%2Fheads%2Fuser%2Ffeich%2Fsupport_language_detection_on_android).

### Recommendation

**Native TFLite (Option B) — no new implementation needed.**

The solution already exists and has been verified. Only two changes are required:

1. **Upstream Chromium**: Change runtime feature flag from `"Android": "test"` → `"Android": "stable"` in `runtime_enabled_features.json5`
2. **Edge Android**: Ensure `EdgeComponentModelRegistry` delivers the TFLite model on Android (BUILD.gn changes to include the model component)

---

## 2. Translator API

### Current Status

| Platform | Engine | Model Delivery | Status |
|----------|--------|---------------|--------|
| Upstream Desktop (Mac/Win/Linux/CrOS) | TranslateKit (native C library) | Component Updater (38 language packs) | Stable |
| Upstream Android | — | — | Not supported |
| Edge Desktop | ONNX Runtime + OGA (`EdgeTranslateKitClient`) | Edge-specific model server (18 language packs) | Shipping |
| Edge Android | — | — | Not supported |

### Alternatives for Android

#### Option A: Port TranslateKit to Android

TranslateKit is Google's **closed-source** native C library (`libtranslatekit.so`/`.dll`) delivered as a prebuilt binary via Component Updater. It runs in a sandboxed utility process and exposes a C-style interface (CreateTranslateKit, Translate, SplitSentences, etc.).

**Current state:**
- Explicitly excluded from Android: `enable_on_device_translation = is_mac || is_win || is_linux || is_chromeos`
- TODO in source: `crbug.com/364795294` — "This class does not support Android yet"
- Library source is Google-internal — cannot be built from Chromium source

| Pros | Cons |
|------|------|
| Same engine as desktop — consistent quality/behavior | **Blocked**: closed-source binary, no Android/ARM64 variant currently available |
| Reuses Component Updater model delivery | Sandboxed utility process needs Android service launcher |
| No GMS dependency | Closed-source binary delivered via Google's Component Updater server — only available to Chrome, not Chromium or downstream browsers (e.g., Edge) unless separately distributed |

#### Option B: Port Edge ONNX/OGA to Android

Edge desktop uses its own translation engine based on ONNX Runtime + OGA (ONNX Generator API).

| Pros | Cons |
|------|------|
| Independent of Google services | ONNX Runtime adds significant binary size on Android |
| Microsoft controls model quality & updates | Not available in upstream Chromium (Edge-proprietary) |

#### Option C: MLKit Translation

Java API via `com.google.mlkit:translate`, models dynamically downloaded and stored in app-local storage.

| Pros | Cons |
|------|------|
| Minimal code surface (~4 new files: Java bridge + C++ controller) | Requires Google Play Services runtime dependencies (Tasks API, Firebase components) |
| MLKit handles model download/cache in app-local storage | In-process Java API (no sandboxing) |
| Edge Android already depends on GMS — works as downstream consumer | Dependency on Google's model release cadence |
| Same NMT models as Google Translate (high quality) | Models stored per-app (not shared across apps) |
| Native lib `libtranslate_jni.so` is small (~2MB) | |
| No Component Updater integration needed | |

### Known Limitations (MLKit on Android)

| Limitation | Details |
|------------|---------|
| **No download progress monitoring** | MLKit's `downloadModelIfNeeded()` only provides success/failure callbacks — no progress events. The Translator Web API's `monitor` option (which reports `loaded`/`total` download progress on desktop via TranslateKit) cannot be supported on Android. `ai.translator.create()` will block until model download completes with no intermediate progress updates. But we can use two non-continous status 0% and 100% to mitigate |
| **Google Play Services required** | MLKit Translation depends on GMS runtime libraries (Tasks, Firebase components) — not available on AOSP-only devices |
| **No sandboxing** | MLKit runs in-process (Java API) unlike desktop TranslateKit which runs in a sandboxed utility process |
| **Per-app model storage** | Models are dynamically downloaded to app-local storage, not shared via GMS across apps |

### Recommendation

**MLKit Translation (Option C) is the best choice for Android.**

1. TranslateKit (Option A) is **blocked** — it's a closed-source prebuilt binary with no ARM64 build available. Only Google can provide one (tracked in `crbug.com/364795294`).
2. Edge ONNX/OGA (Option B) is proprietary to Edge and not viable for upstream Chromium.
3. MLKit (Option C) is production-ready, lightweight, and works for both Chromium and Edge Android since both already ship with GMS dependencies.

---

## 3. Prompt API (AILanguageModel)

### Current Status

| Platform | Backend | Status |
|----------|---------|--------|
| Upstream Desktop | On-device model via Optimization Guide | Stable |
| Upstream Android | MLKit GenAI Prompt (`com.google.mlkit:genai-prompt`) via AICore | Backend mapped, runtime flag disabled |
| Edge Desktop | Same as upstream | Stable |
| Edge Android | MLKit GenAI Prompt via AICore | Backend mapped, runtime flag disabled |

### What's Working

- Backend mapping exists: `kPromptApi` → `kAICorePrompt` in `GetAICoreFeatureFor()` (`components/optimization_guide/core/model_execution/android/model_broker_android.cc`)
- Same backend as Summarization API (which already works on Android)

### Web API vs MLKit GenAI Prompt Feature Gap

| Web API Feature | MLKit GenAI Prompt | Gap |
|---|---|---|
| `prompt(input)` → string | ✅ `generateContent(prompt)` | Mappable |
| `promptStreaming(input)` → ReadableStream | ✅ `generateContentStream(prompt)` | Mappable |
| `measureContextUsage(input)` → token count | ✅ `countTokens(request)` | Mappable |
| `temperature` / `topK` | ✅ Supported as config options | Mappable |
| `maxOutputTokens` | ✅ Supported | Mappable |
| Image input (`{type: "image", value}`) | ✅ `ImagePart(bitmap)` supported | Mappable — but Chromium impl hits `NOTREACHED()` |
| `initialPrompts` / system prompts | ⚠️ No multi-turn session API | **Gap**: MLKit is single-turn only; Chromium manages context locally |
| `clone()` (duplicate session state) | ❌ No session concept | **Gap**: Must be handled entirely in Chromium layer |
| `contextUsage` / `contextWindow` properties | ❌ Not exposed by MLKit | **Critical gap**: Cannot track token budget accurately |
| `contextoverflow` event | ❌ No equivalent | **Gap**: Chromium must estimate overflow locally |
| Audio input (`{type: "audio", value}`) | ❌ Not supported | **Missing**: MLKit Prompt only supports text + image |
| Tool use (function calling) | ❌ Not supported | **Missing**: MLKit has no tool/function calling API |
| Response constraints (JSON schema / regex) | ❌ Not supported | **Missing**: No structured output enforcement |
| `append()` (add context without response) | ❌ No session/conversation API | **Gap**: Must be managed in Chromium layer |
| `seed` (deterministic output) | ✅ Supported | Mappable — but not exposed in Web API |
| `candidateCount` (multiple responses) | ✅ Supported | Not in Web API spec |
| Download progress (`monitor`) | ✅ `download()` with progress | Mappable |
| Availability check | ✅ `checkStatus()` | Mappable |

### MLKit GenAI Prompt Limitations

| Limitation | Details |
|------------|---------|
| **Single-turn only** | No multi-turn conversation or session API — each `generateContent()` is independent |
| **4000-token input cap** | ~3000 English words max per request |
| **256-token recommended output cap** | "Use cases that require long output (more than 256 tokens) should be avoided" |
| **Language support** | Validated only for English and Korean |
| **Per-app inference quota** | AICore enforces rate limiting per app |
| **No unlocked bootloaders** | Blocked on rooted/unlocked devices |
| **Requires warmup** | `warmup()` recommended to preload model — first inference has cold-start latency |

### Desktop vs Android Feature Alignment (Chromium Implementation)

Chromium's Android backend bridges the gap between the stateful Web API and MLKit's stateless single-turn API by managing conversation context locally. The table below shows what works in the **Chromium implementation** (not raw MLKit):

| Feature | Desktop | Android (Chromium) | Gap Type | Notes |
|---------|---------|-----------------|----------|-------|
| Streaming responses | ✅ | ✅ | — | |
| Token counting (`measureContextUsage`) | ✅ | ✅ | — | Via MLKit `countTokens()` |
| Session cloning (`clone()`) | ✅ | ✅ | — | Chromium manages session state locally |
| Image input | ✅ | ❌ | Missing impl | `NOTREACHED()` when encountering bitmap in `Generate()` (`crbug.com/425408635`); MLKit supports `ImagePart` |
| Initial/system prompts | ✅ | ✅ | — | Chromium prepends to each request (MLKit has no native session) |
| Temperature / topK | ✅ | ✅ | — | Already passed to MLKit via `MlKitAiCoreSessionBackendImpl` |
| **Tool use (function calling)** | ✅ | ❌ | Missing impl | Mojo interface ready; MLKit has no tool API (`crbug.com/422803232`) |
| **Response constraints (JSON schema/regex)** | ✅ | ❌ | Missing impl | Received in `GenerateOptions` but discarded at JNI boundary; MLKit has no constraint API |
| **Audio input** | ✅ | ❌ | Missing impl | `AsrStream()`/`AsrAddAudioChunk()` return NOTIMPLEMENTED; MLKit doesn't support audio (`crbug.com/425408635`) |
| **Context window tracking** | ✅ Accurate | ❌ Returns 0 | MLKit limitation | MLKit is stateless per call; `OnComplete(tokens_processed=0)` is hardcoded; would need MLKit API enhancement |

### Reducible Gaps

These can be fixed with Chromium implementation work (MLKit already supports the underlying capability):

1. **Image input** — MLKit supports `ImagePart(bitmap)`. Chromium just hits `NOTREACHED()` on bitmap input — needs to convert to MLKit image format and pass through JNI.

### Non-Reducible Gaps (MLKit API Limitations)

These require MLKit API enhancements or alternative approaches:

2. **Tool use** — MLKit GenAI Prompt has no function calling / tool use API. Would need MLKit to add this, or Chromium to implement tool dispatch purely in the C++ layer with multiple model calls.
3. **Response constraints** — MLKit has no structured output enforcement (JSON schema, regex). Would need MLKit enhancement or post-processing validation in Chromium.
4. **Audio input** — MLKit GenAI Prompt supports text + image only, not audio. Would need MLKit to add audio modality.
5. **Context window tracking** — MLKit is stateless; cannot report tokens consumed per session. `countTokens()` exists but Chromium would need to call it on every accumulated context to approximate `contextUsage`.
6. **Multi-turn conversation** — MLKit has no session API. Chromium already works around this by accumulating `context_input_pieces_` and resending full context each call, but this is inefficient and hits the 4000-token input cap quickly.

### What's Blocking

1. **Runtime feature flag**: `AIPromptAPI` has `"default": ""` in `runtime_enabled_features.json5` — not enabled on Android
2. **chrome://flags**: Uses `kOsDesktop` — cannot manually enable on Android
3. **Chrome-branded builds**: `kAICorePrompt` is `FEATURE_DISABLED_BY_DEFAULT` on Google Chrome branded builds (enabled by default on non-branded/Chromium builds)

### What's Needed

1. Enable `AIPromptAPI` for Android: add `"Android": "stable"` in `runtime_enabled_features.json5`
2. For Chrome-branded builds: ensure `kAICorePrompt` feature is enabled via Finch or flag override

---

## 4. Writer API

### Current Status

| Platform | Backend | Status |
|----------|---------|--------|
| Upstream Desktop | On-device model via Optimization Guide | Experimental |
| Upstream Android | — | **No backend mapping**, runtime flag disabled |
| Edge Desktop | Same as upstream | Experimental |
| Edge Android | — | **No backend mapping**, runtime flag disabled |

### What's Blocking

1. **Backend mapping MISSING**: `kWritingAssistanceApi` is **not** in the `GetAICoreFeatureFor()` switch statement — returns `nullopt` on Android
2. **Runtime feature flag**: `AIWriterAPI` has `"default": ""` — not enabled on Android
3. **chrome://flags**: Uses `kOsDesktop`

### Desktop vs Android Feature Alignment

**Key architecture:** On desktop, Writer/Rewriter/Proofreader pass structured proto requests (`WritingAssistanceApiRequest` with tone/format/length enums) to the on-device model. The model's built-in execution config (substitution templates) converts these protos into actual prompts. **Android AICore uses the same model execution config**, so structured options are interpreted identically.

| Feature | Desktop | Android (AICore) | Notes |
|---------|---------|-----------------|-------|
| `write()` / `writeStreaming()` | ✅ | ✅ | Same proto + same execution config |
| `measureInputUsage()` | ✅ | ✅ | Token counting supported |
| Tone (formal/neutral/casual) | ✅ | ✅ | Proto enum → model config handles conversion |
| Format (plain-text/markdown) | ✅ | ✅ | Proto enum → model config handles conversion |
| Length (short/medium/long) | ✅ | ✅ | Proto enum → model config handles conversion |
| `sharedContext` | ✅ | ✅ | Field in proto, handled by config |
| `outputLanguage` | ✅ | ✅ | Language display name in proto options |

### What's Needed

1. Add `kWritingAssistanceApi` → `kAICorePrompt` mapping in `GetAICoreFeatureFor()`
2. Enable `AIWriterAPI` for Android in `runtime_enabled_features.json5`

---

## 5. Rewriter API

### Current Status

| Platform | Backend | Status |
|----------|---------|--------|
| Upstream Desktop | On-device model via Optimization Guide | Experimental |
| Upstream Android | — | **No backend mapping**, runtime flag disabled |
| Edge Desktop | Same as upstream | Experimental |
| Edge Android | — | **No backend mapping**, runtime flag disabled |

### What's Blocking

Same as Writer API — both use `kWritingAssistanceApi` feature.

1. **Backend mapping MISSING**: Same `kWritingAssistanceApi` not in `GetAICoreFeatureFor()`
2. **Runtime feature flag**: `AIRewriterAPI` has `"default": ""` — not enabled on Android
3. **chrome://flags**: Uses `kOsDesktop`

### Desktop vs Android Feature Alignment

Same architecture as Writer — passes structured `WritingAssistanceApiRequest` proto. Android AICore uses same model execution config.

| Feature | Desktop | Android (AICore) | Notes |
|---------|---------|-----------------|-------|
| `rewrite()` / `rewriteStreaming()` | ✅ | ✅ | Same proto + same execution config |
| `measureInputUsage()` | ✅ | ✅ | Token counting supported |
| Tone (as-is/more-formal/more-casual) | ✅ | ✅ | `as-is` maps to `NEUTRAL` in proto |
| Format (as-is/plain-text/markdown) | ✅ | ⚠️ | `as-is` maps to `NOT_SPECIFIED` (marked NOTIMPLEMENTED on desktop too) |
| Length (as-is/shorter/longer) | ✅ | ✅ | `as-is` maps to `MEDIUM` in proto |
| `sharedContext` | ✅ | ✅ | Field in proto, handled by config |
| `outputLanguage` | ✅ | ✅ | Language display name in proto options |

### What's Needed

1. Same backend fix as Writer (single change covers both — shared `kWritingAssistanceApi`)
2. Enable `AIRewriterAPI` for Android in `runtime_enabled_features.json5`

---

## 6. Proofreader API

### Current Status

| Platform | Backend | Status |
|----------|---------|--------|
| Upstream Desktop | On-device model via Optimization Guide (Gemini Nano) | Experimental |
| Upstream Android | — | **No backend mapping**, runtime flag disabled |
| Edge Desktop | Same as upstream | Experimental |
| Edge Android | — | **No backend mapping**, runtime flag disabled |

### What's Blocking

1. **Backend mapping MISSING**: `kProofreaderApi` is **not** in the `GetAICoreFeatureFor()` switch statement
2. **Runtime feature flag**: `AIProofreadingAPI` has `"default": ""` — not enabled on Android
3. **chrome://flags**: Uses `kOsDesktop`

### Alternatives for Android

#### Option A: AICore via Generic Prompt (Same as Desktop)

Desktop Proofreader uses Gemini Nano with a `ProofreaderApiRequest` proto and two execution modes:
1. **Proofread mode**: Input text → fully corrected text
2. **Label mode**: Original + corrected text → correction type label (uses `ResponseConstraint`)

Android AICore uses the same model and execution config, so this is the natural path.

| Pros | Cons |
|------|------|
| Same engine as desktop — consistent quality | Label mode requires `ResponseConstraint` (regex) — not yet supported on AICore |
| No new dependencies — AICore already used by Prompt API | AICore availability limited (Pixel 8+, select Samsung) |
| Full Web API compliance (structured corrections with offsets, types, explanations) | Requires AICore + Gemini Nano on device |
| Same model execution config handles all options | |

#### Option B: MLKit GenAI Proofreading (`com.google.mlkit:genai-proofreading`)

MLKit offers a dedicated proofreading API backed by Gemini Nano + LoRA adapter via AICore.

| Pros | Cons |
|------|------|
| Purpose-built API optimized for proofreading | **No structured corrections** — returns only corrected text, no per-correction offsets/types/explanations |
| Supports 7 languages (en, ja, fr, de, it, es, ko) | **256-token input limit** — restrictive for long text |
| Download progress callback available (`DownloadCallback`) | Same AICore/Gemini Nano dependency as Option A |
| Input type distinction (keyboard vs voice) | Not available on devices with unlocked bootloaders |
| Streaming variant available | Would require custom diff algorithm to approximate `ProofreadCorrection[]` — unreliable for complex corrections |

### Web API vs MLKit Feature Gap

| Web Proofreader API Feature | MLKit GenAI Proofreading | Gap |
|---|---|---|
| `ProofreadCorrection[]` with `startIndex`/`endIndex` | ❌ Not exposed — only full corrected text | **Critical**: cannot produce per-correction offsets |
| `CorrectionType` enum (spelling, grammar, punctuation, etc.) | ❌ Not exposed | **Critical**: no correction categorization |
| `includeCorrectionExplanations` | ❌ Not supported | **Major**: no explanation generation |
| Streaming (`proofreadStreaming()`) | ✅ `runInference(request, callback)` | Mappable |
| Download progress (`monitor`) | ✅ `DownloadCallback` with byte counts | Mappable (byte-based → normalized 0–1) |
| Availability check | ✅ `checkFeatureStatus()` | Mappable (same states) |
| Language selection | ✅ `setLanguage()` — 7 languages | Minor: Web allows array of BCP-47, MLKit allows single enum |
| Input length | Not specified | **< 256 tokens** | Android limitation |
| Destroy/close | `destroy()` | `close()` | Mappable |

### Desktop vs Android Feature Alignment (Option A — AICore)

| Feature | Desktop | Android (AICore) | Notes |
|---------|---------|-----------------|-------|
| `proofread()` → corrected text | ✅ | ✅ | Same proto + same execution config |
| Correction type labeling (label mode) | ✅ | ❌ | Requires `ResponseConstraint` (regex/JSON schema) — not supported on AICore |
| `includeCorrectionTypes` | ✅ | ⚠️ | Proofread works, but label inference blocked |
| `includeCorrectionExplanations` | ✅ | ✅ | Boolean in proto options, handled by config |
| Structured `ProofreadCorrection` output | ✅ | ✅ | Model config handles response format |
| Language options | ✅ | ✅ | Handled by model config |

### Recommendation

**Option A (AICore via Generic Prompt) — same architecture as desktop.**

1. MLKit GenAI Proofreading (Option B) has **critical gaps**: no structured corrections, no correction types, no explanations. The Web API requires `ProofreadCorrection[]` with precise offsets and type labels — MLKit cannot provide these. Reverse-engineering corrections via text diff is fragile and unreliable.
2. Option A reuses the same Gemini Nano model + execution config as desktop. The backend mapping is trivial (same pattern as Writer/Rewriter).
3. The only limitation is label mode (`includeCorrectionTypes`) which requires `ResponseConstraint` support on AICore — this is the same gap as desktop's advanced features and can be addressed when AICore adds constraint support.

### What's Needed

1. Add `kProofreaderApi` → `kAICorePrompt` mapping in `GetAICoreFeatureFor()`
2. Enable `AIProofreadingAPI` for Android in `runtime_enabled_features.json5`
3. **Known limitation**: Label mode (correction type inference) blocked by missing `ResponseConstraint` support on AICore

---

## 7. Summary

| API | Recommended Backend | MLKit GenAI Package | Complexity | Key Limitation |
|-----|-------------------|-------------------|-----------|----------------|
| Language Detection | Native TFLite (same model as desktop) | N/A (not GenAI) | Low — feature flag only | None — already fully functional |
| Translator | MLKit Translation (NMT models) | `com.google.mlkit:translate` | Medium — JNI bridge + controller | No download progress monitoring |
| Prompt API | MLKit GenAI Prompt | `com.google.mlkit:genai-prompt` | Low — feature flag only | Single-turn only (no session API), no tool use, no response constraints, no audio, 4000-token input cap, context window tracking returns 0 |
| Writer API | MLKit GenAI Prompt | `com.google.mlkit:genai-prompt` (no dedicated Writer API) | Low — add backend mapping | None — same execution config as desktop |
| Rewriter API | MLKit GenAI Prompt | `com.google.mlkit:genai-rewriting` exists but Chromium uses generic prompt path | Low — add backend mapping | None — same execution config as desktop |
| Proofreader API | MLKit GenAI Prompt | `com.google.mlkit:genai-proofreading` exists but lacks structured corrections; Chromium uses generic prompt path | Low — add backend mapping | Label mode blocked by missing `ResponseConstraint` support |
