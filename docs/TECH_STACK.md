# Sonora - Technology Stack Decisions

> **Document purpose:** consolidate the approved technology Architecture Decision Records (ADRs) for the Sonora project.
>
> **Status convention:** every ADR below records an approved technology decision (`DECIDED`). Proposals and unresolved implementation details remain in `TECHNICAL_ARCHITECTURE.md` and must not be treated as approved merely because they appear in a diagram or example.
>
> **Important distinction:** approved ADRs are the selected technology baseline. Architecture patterns labeled `RECOMMENDED` in `TECHNICAL_ARCHITECTURE.md` are non-mandatory guidance. An implementation agent may replace those recommendations when it identifies a concrete technical improvement, while preserving approved product requirements and ADR decisions or explicitly proposing a revision to an ADR if evidence warrants it. The execution model (Main Thread / one Worker / no AudioWorklet) is `DECIDED` in `TECHNICAL_ARCHITECTURE.md` Section 12, not merely recommended.

## Decision summary

| ADR | Decision | Status |
|---|---|---|
| 0001 | TypeScript as primary language | DECIDED |
| 0002 | React as application framework | DECIDED |
| 0003 | Vite as build system | DECIDED |
| 0004 | npm as package manager | DECIDED |
| 0005 | Three.js + WebGL 2 + GLSL ES 3.00 | DECIDED |
| 0006 | Web Audio API for decoding and canonical PCM conversion | DECIDED |
| 0007 | In-house TypeScript audio analysis | DECIDED |
| 0008 | Acquisition backend: Node.js + TypeScript with yt-dlp | DECIDED |
| 0009 | Audio identity from the delivered file bytes (amends 0006) | DECIDED |
| 0010 | Local audio cache in IndexedDB | DECIDED |
| 0011 | Hosting: GitHub Pages (frontend) and Cloud Run (backend), deployed by GitHub Actions | DECIDED |
| 0012 | Audio acquisition through third-party RapidAPI APIs with fallback (amends 0008) | DECIDED |

---

# ADR 0001 - TypeScript as Primary Language

- **Status:** DECIDED
- **Decision date:** 2026-11-13

## Context

Sonora is composed of multiple cooperating systems: audio acquisition and processing, music intelligence, deterministic world identity, procedural generation, scenario-specific interpretation, 3D rendering, and experience orchestration. These systems exchange structured data such as audio features, timeline events, seeds, scenario configuration, segment descriptors, and world state.

The project benefits from explicit contracts between modules and reliable refactoring as the architecture evolves.

## Decision

**TypeScript is the primary programming language for the Sonora application.**

## Rationale

- Static types help define and maintain contracts between the project's systems.
- TypeScript integrates directly with the browser and the JavaScript ecosystem used by React, Vite, and Three.js.
- Types improve editor assistance, navigation, refactoring, and early detection of inconsistent interfaces.
- It allows the project to remain within one primary language while its audio, procedural, and rendering subsystems mature.

## Consequences

- Shared interfaces and domain types should be explicit and owned by appropriate modules.
- Runtime validation is still required at untrusted boundaries; TypeScript types do not validate runtime data.
- TypeScript's types do not inherently make JavaScript execution faster.
- Rust/WebAssembly is not part of the initial baseline. It may be reconsidered only if profiling identifies a workload where it provides a justified benefit.

---

# ADR 0002 - React as Application Framework

- **Status:** DECIDED
- **Decision date:** 2026-11-14

## Context

Sonora needs an application layer for URL input, scenario selection, loading/preparation states, experience lifecycle, and user interaction. The 3D rendering engine has separate requirements and should not be coupled to the UI framework's rendering lifecycle.

The product is client-side and does not currently require server-side rendering, server components, or a full-stack application framework.

## Decision

**React is the application UI framework.** Three.js remains an independent rendering system and will be integrated directly. React Three Fiber is not part of the initial baseline.

## Rationale

- React provides a mature component model for the application's screens and lifecycle states.
- It integrates well with TypeScript and Vite.
- Keeping Three.js independent preserves control over the render loop, scene lifecycle, camera, and GPU resources.
- There is no current product requirement that justifies introducing a server-oriented full-stack framework.

## Consequences

- React owns UI composition, DOM interaction, and application-level state.
- Three.js owns the 3D scene, camera/rendering objects, and frame-by-frame rendering.
- React state should not be updated as a substitute for a high-frequency render loop.
- Communication between the application and renderer must use explicit interfaces and lifecycle boundaries.
- Server-side rendering and full-stack framework features are not part of the initial architecture unless a future requirement justifies them.

---

# ADR 0003 - Vite as Build System

- **Status:** DECIDED
- **Decision date:** 2026-11-14

## Context

The application is a browser-based React experience. Its main workloads—audio processing, music analysis, procedural generation, and 3D rendering—run on the user's device. The project does not currently need a server-rendering build pipeline or a framework-provided application server.

The development workflow should support fast startup and feedback while producing optimized client-side assets for deployment.

## Decision

**Vite is the development server and build system for Sonora.** It will be used with React and TypeScript.

## Rationale

- Vite provides a lightweight client-side development/build workflow suited to a browser application.
- It offers fast development startup and hot module replacement.
- It produces production assets without requiring a full-stack framework.
- It integrates with React and TypeScript.

## Consequences

- Vite is a build/development tool, not a replacement for React.
- The project must include an explicit TypeScript type-checking step (for example, `tsc --noEmit`) because Vite transpiles TypeScript but does not perform full type checking by itself.
- Exact Vite version, plugin configuration, build targets, and scripts remain implementation details to be established.
- The initial application is client-rendered; server-side rendering is not required by this decision.

---

# ADR 0004 - npm as Package Manager

- **Status:** DECIDED
- **Decision date:** 2026-11-14

## Context

The project needs a package manager for installing, updating, and locking JavaScript/TypeScript dependencies. pnpm was considered for its package-store efficiency and strict dependency layout. The user has existing familiarity with npm and Yarn and prefers to use a familiar tool unless the alternative provides a compelling project-specific advantage.

## Decision

**npm is the package manager for Sonora.**

## Rationale

- npm is familiar to the project owner and is broadly supported across the JavaScript ecosystem.
- It is sufficient for the current application architecture and expected dependency structure.
- The additional storage and workspace benefits of pnpm do not justify introducing a less familiar workflow at this stage.

## Consequences

- Commit and maintain `package-lock.json` to make dependency installation reproducible.
- Use npm consistently for project scripts and dependency management.
- Do not mix package managers or commit competing lockfiles.
- Reconsider the choice only if future repository/workspace requirements demonstrate a concrete need.

---

# ADR 0005 - Three.js, WebGL 2, and GLSL ES 3.00

- **Status:** DECIDED
- **Decision date:** 2026-11-14

## Context

Sonora requires procedural 3D environments, camera control, custom geometry and materials, instancing, shader-driven visual effects, and dynamic segment management. The project's visual systems should remain under application-level control rather than depending on a game-engine-style framework that owns the complete experience.

The project owner also identified documentation quality and precision as a key selection criterion, particularly to reduce unsupported assumptions and incorrect API usage during AI-assisted development.

## Decision

The initial 3D rendering stack is:

- **3D library:** Three.js.
- **Graphics API:** WebGL 2.
- **Shader language:** GLSL ES 3.00.
- **Integration:** direct Three.js integration, independent from React's component/render cycle; React Three Fiber is not part of the initial baseline.
- **WebGPU:** not part of the initial implementation baseline; it may be evaluated later.

## Rationale

- Three.js provides a practical abstraction over low-level graphics APIs while retaining control over geometry, materials, shaders, camera, instancing, and rendering.
- WebGL 2 is a well-established browser graphics baseline with formal Khronos specifications and complementary MDN documentation.
- GLSL ES 3.00 matches the WebGL 2 / OpenGL ES 3.0 shader baseline and avoids ambiguity between desktop GLSL and different GLSL ES versions.
- Three.js, WebGL 2, and GLSL ES 3.00 have substantial official documentation, API references, examples, and formal specifications.
- Babylon.js and direct WebGL 2 were considered. Babylon.js offers a broader integrated engine; direct WebGL 2 offers maximum low-level control but requires implementing more rendering infrastructure. Three.js is the selected balance for Sonora's procedural visual workload.

## Documentation and implementation policy

- Use the official Three.js documentation and examples corresponding to the exact installed version.
- Use Khronos specifications and reference pages as the authoritative source for WebGL 2 and GLSL ES 3.00 behavior.
- Use MDN as complementary documentation for browser APIs and compatibility.
- Do not assume that a class, method, property, shader feature, or behavior exists without confirming it against the applicable version and reference.
- Pin dependency versions through the npm lockfile and verify compatibility before adopting examples found online.

## Consequences

- Three.js owns the scene graph and 3D render loop; React owns the UI and application lifecycle.
- Custom shaders must target GLSL ES 3.00 for the WebGL 2 baseline.
- WebGPU-specific renderer behavior, TSL migration, and WebGPU compatibility are future evaluation topics, not current implementation requirements.
- Browser/device support and WebGL 2 context availability must be validated as part of the MVP's compatibility and performance work.
- Installed version (2026-11-18): Three.js 0.187, pinned by `package-lock.json`. No post-processing library is used: Voxel Dreamscape's post-processing is written in-house (`voxel/post.ts`).

---

# ADR 0006 - Web Audio API for Decoding and Canonical PCM Conversion

- **Status:** DECIDED
- **Decision date:** 2026-11-15
- **Amended:** 2026-11-15 by ADR 0009 (the identity hash is no longer computed over the PCM)

## Context

The client receives the original m4a/AAC audio file (or an MP3 fallback) from the acquisition backend and must obtain canonical PCM (44.1 kHz, stereo, Float32, kept as two separate channels) for analysis and playback. Cross-browser byte-identical decoding is not required (see `PRODUCT_SPEC.md` Section 6.6).

## Decision

**Decode with the browser's Web Audio API** (`decodeAudioData` on an `OfflineAudioContext` created at 44,100 Hz), then keep the canonical PCM as two separate Float32 channels (no manual interleaving). The SHA-256 identity is computed over the delivered file bytes, not over this PCM (ADR 0009). No third-party decoding or resampling library is used.

## Rationale

- Native decoding is fast, dependency-free, and available in all target browsers.
- `decodeAudioData` resamples to the context's sample rate, so no separate resampler is needed.
- Output is already Float32 and planar, so no conversion is needed.

## Consequences

- `decodeAudioData` runs on the main thread; the resulting PCM is transferred to the Worker for analysis.
- Mono sources are converted to stereo by duplicating the channel (assumption, to be confirmed in implementation).
- Encoder delay/padding handling and decoder numerics may differ between browsers, and some browsers perturb the decoded samples between sessions (measured in Brave); the resulting analysis differences are accepted, and identity is unaffected (ADR 0009).
- `crypto.subtle` (used for the file hash) requires a secure context (HTTPS or localhost).
- `decodeAudioData` detaches its input buffer, so the file must be hashed before decoding (or copied).
- Measured on a desktop (Brave, 10-minute synthetic track): decode about 0.75 to 1.3 s; canonical PCM 202 MiB; process memory growth about 560 to 600 MiB. Mobile not yet measured.
- Little-endian byte order must be asserted at runtime; big-endian platforms are unsupported.

---

# ADR 0007 - In-house TypeScript Audio Analysis

- **Status:** DECIDED
- **Decision date:** 2026-11-15

## Context

Sonora needs reactive features (RMS, bands, spectrum, transients), rhythmic data (BPM, beats), and practical structural estimates with confidence. Cross-browser determinism of the analysis is not required.

## Decision

**Implement the analysis pipeline in TypeScript** (STFT with a small FFT, band energies, spectral-flux onsets, tempo estimation, dynamic-programming beat tracking, energy/spectral novelty for structure). No analysis library is adopted initially.

## Rationale

- Keeps the dependency footprint small and avoids licensing questions.
- Gives full control over confidence modeling and scenario-facing outputs.
- WASM is not part of the baseline (ADR 0001).

## Consequences

- Beat-tracking and structure quality must be validated with a short spike on a varied set of tracks before the pipeline is considered final.
- If the spike fails systematically, a library such as Essentia.js (WASM; license and ADR 0001 implications to be verified) may be proposed, but would require a new explicit approval.
- No quantization of outputs for cross-browser stability is required.

---

# ADR 0008 - Acquisition Backend: Node.js + TypeScript with yt-dlp

- **Status:** DECIDED
- **Decision date:** 2026-11-15
- **Amended:** 2026-11-16 by ADR 0012 (production acquisition goes through RapidAPI providers; yt-dlp remains the local and fallback extractor)

## Context

The browser cannot fetch YouTube audio directly: YouTube pages and media servers do not allow cross-origin access from arbitrary sites, stream URL resolution requires frequently changing extraction logic, and resolved stream URLs are commonly bound to the requester's IP. A small Sonora-owned backend is therefore required for acquisition and delivery only. This is an experimental, non-commercial, unpublished prototype; the decision does not imply platform approval or long-term reliability.

## Decision

- **Extractor:** yt-dlp, a third-party executable invoked as a subprocess behind the replaceable source adapter.
- **Runtime:** Node.js + TypeScript (consistent with ADR 0001), allowing HTTP contract types to be shared with the frontend.
- **Format:** deliver the original m4a/AAC stream without re-encoding; ffmpeg is used only as a fallback conversion when no m4a stream exists.
- **State:** asynchronous jobs identified by `job_id`, held in memory only; temporary files are deleted after the first completed download, with a safety TTL and cleanup at startup.

## Rationale

- yt-dlp is the most actively maintained extractor and is corrected quickly when YouTube changes.
- Keeping the backend in TypeScript avoids a second application language.
- Skipping conversion removes the ffmpeg dependency from the common path and avoids a second lossy encoding step.
- In-memory job state matches the no-persistence rule.

## Consequences

- yt-dlp must be kept up to date; extractor breakage and datacenter-IP blocking are accepted operational risks.
- Operational viability must be validated by the feasibility spike in `TECHNICAL_ARCHITECTURE.md` Section 5.1 before the backend is built.
- Hosting requirements are long-running processes, an installable yt-dlp, and temporary disk; the provider is chosen in ADR 0011.
- Restarting the service loses in-flight jobs.
- Operational limit values (TTL, concurrency, file size, timeout) are starting points to be calibrated, not decisions.

---

# ADR 0009 - Audio Identity from the Delivered File Bytes

- **Status:** DECIDED
- **Decision date:** 2026-11-15
- **Amends:** ADR 0006, `PRODUCT_SPEC.md` Sections 6.1 and 6.3 to 6.6.

## Context

The identity was defined as SHA-256 over canonical PCM produced by `decodeAudioData`. The acquisition spike measured, for the same m4a file:

- Edge: decoded samples bit-identical across separate sessions.
- Brave: decoded samples differ in about 99.65% of values between sessions (maximum absolute difference 1.7e-4, RMS difference 3.4e-5, about 76 dB below the signal RMS of 0.214). The PCM hash therefore changes at every session, which breaks same-environment reproducibility (`PRODUCT_SPEC.md` Section 6.6). The cause is probably a browser privacy feature that perturbs Web Audio output; this was not confirmed.
- The m4a file bytes were identical across repeated downloads of 3 videos with the same yt-dlp version.

## Decision

**The audio identity is the SHA-256 of the exact bytes of the audio file as delivered by the backend**, computed before decoding. The canonical PCM remains the input of analysis and playback but is not hashed. The world seed formula is otherwise unchanged.

## Rationale

- The file hash is stable in every browser, including those that perturb decoded audio.
- It removes the interleave-and-hash step from the identity path (less memory and time on mobile) and makes the identity available before decoding.
- It requires no dependency and no change to the acquisition design.

## Consequences

- The identity is only as stable as the acquired bytes. yt-dlp applies a container fixup (`FixupM4a`); a different yt-dlp version or a changed source stream may change the bytes. The yt-dlp version should be recorded or pinned.
- Container and metadata bytes are part of the identity. The MP3 fallback has its own identity.
- Stability of the bytes over days and from other IPs or hosts has not been verified (3 videos, one machine, one day).
- Analysis still runs on decoded PCM, which may vary slightly in browsers that perturb it. The effect on analysis output is not measured and must be checked in the analysis spike (ADR 0007).
- The product's "preserve silence" and "same canonical audio" rules now apply to the delivered file as a whole.

---

# ADR 0010 - Local Audio Cache in IndexedDB

- **Status:** DECIDED
- **Revises:** the earlier no-persistence rule, for the audio file only.

## Context

Every submitted URL triggered a full backend download, even for a video already processed. The project owner authorized persisting the downloaded audio locally to avoid repeated waits.

## Decision

The client stores the delivered audio bytes in IndexedDB, keyed by YouTube video id, with a 500 MB limit and least-recently-used eviction. The backend stays stateless. No new dependency is used (native IndexedDB API).

## Rationale

- Removes repeated download time for the same video.
- Keeping the cache in the client leaves the server unchanged and avoids server-side storage and privacy concerns.
- Video-id keys make different URL forms of the same video hit the same entry.

## Consequences

- Cache is per browser and can be cleared by the browser or the user; a failure is just a miss.
- The audio identity is still computed from the bytes on each execution; cached bytes make it stable for that browser.
- Catalog, history, and identity/analysis caches remain prohibited.
- No UI to inspect or clear the cache yet (not in scope).

---

# ADR 0011 - Hosting: GitHub Pages and Cloud Run, Deployed by GitHub Actions

- **Status:** DECIDED
- **Decision date:** 2026-11-16
- **Resolves:** the OPEN hosting provider in `TECHNICAL_ARCHITECTURE.md` Section 5.1 and ADR 0008.

## Context

The project owner chose to host the backend on Google Cloud Run within the free tier and the static frontend on GitHub Pages, with deployment on every push to `main`.

## Decision

- Frontend: `vite build` output published to GitHub Pages (`.github/workflows/deploy-frontend.yml`), served under `/<repo-lowercase>/` (`BASE_PATH`). The workflow lowercases the repository name because GitHub Pages normalizes project-site paths that way. The backend origin is injected at build time as `VITE_API_BASE_URL` from the `BACKEND_URL` repository variable; empty means same origin (development proxy).
- Backend: Docker image built by Cloud Build and deployed to Cloud Run in `us-central1`, `max-instances=1` (`.github/workflows/deploy-backend.yml`). Authentication uses Workload Identity Federation, with no stored keys. `ALLOWED_ORIGIN` restricts browser reads (CORS) to the Pages origin.
- Each workflow runs only when its side changes (path filters). Free-tier guards are in `deploy/gcp-free-tier-guards.sh`. Operational details are in `DEPLOYMENT.md`.

## Rationale

- Both are free at personal-project scale and need no server administration.
- The backend stays a plain Node service in a container, so the architecture remains portable.

## Consequences

- Cross-origin calls depend on CORS and on `ALLOWED_ORIGIN` matching the Pages origin.
- Cloud Run throttles CPU outside requests and keeps `/tmp` in memory: background yt-dlp work and the egress counter (reset on cold start) still need validation.
- yt-dlp on a datacenter IP remains unvalidated (ADR 0008). Update: it is blocked (see ADR 0012).

---

# ADR 0012 - Audio Acquisition Through Third-Party RapidAPI APIs, With Fallback

- **Status:** DECIDED
- **Decision date:** 2026-11-16
- **Amends:** ADR 0008 (extractor) and the "original m4a without re-encoding" delivery rule.

## Context

On Cloud Run, YouTube rejects yt-dlp with its bot check ("Sign in to confirm you're not a bot"), so every job fails with `BLOCKED`. The project owner ruled out a residential host, PO-token providers, paid proxies, account cookies, and file upload, and chose the RapidAPI API "YouTube Audio Downloader & Converter" (provider `nbdy0`, host `api-youtubemp3.paulius.pro`) on its free plan. The project is expected to run only a few times a year.

## Decision

- When `RAPIDAPI_KEY` is set, the backend acquires audio through `server/rapidapi.ts` instead of yt-dlp: `POST /api/v2/jobs` (`format=m4a`, `quality=128`, `Idempotency-Key` = Sonora job id), long-poll `GET /api/v2/jobs/{id}?waitSeconds=25`, then stream `GET /api/v2/jobs/{id}/download` to the temp dir. The rest of the pipeline (Sonora jobs, limits, delivery, HTTP contract) is unchanged.
- **Providers, in priority order** (owner's choice, 2026-11-16):
  1. "YouTube MP3 M4A Converter" (`server/ytMp3M4a.ts`, added 2026-11-16 at the owner's request after the two others failed: quota exhausted and `SERVICE_TEMPORARILY_UNAVAILABLE`): `GET /audio?id=<videoId>` returns JSON with `lengthSeconds`, `isLiveContent` and `linkDownload`, a plain URL that serves the m4a (observed: AAC 128 kbps, duration matching the video). The provider's documented `/download` and `/stream` endpoints did not work when tested; `/audio` did, so it is undocumented and may change. Free plan: 100 requests per month (observed in the response headers).
  2. "YouTube to MP3 Converter API" (GatiMan, `server/ytToMp3.ts`): `POST /output.php` with multipart `url` returns JSON `medias[]` with relay links to YouTube's own streams; the m4a/AAC audio (YouTube format 140, the stream yt-dlp downloads) is picked and fetched from the relay. Free plan: 4 requests per month (observed in the response headers).
  3. "YouTube Audio Downloader & Converter" (nbdy0, `server/rapidapi.ts`): the job API described above.
- **Fallback chain** (`server/sources.ts`): the backend tries the configured providers in priority order (`PROVIDERS` in `server/jobs.ts`). Provider failures (generic, quota, blocked) move to the next provider; outcomes about the video itself (unavailable, live, too long, too large) stop the chain. One RapidAPI key is shared by all providers. The job timeout covers the whole chain.
- The key is stored in GCP Secret Manager (`rapidapi-key`) and exposed to Cloud Run as `RAPIDAPI_KEY`. It never passes through GitHub and is never sent to the frontend.
- Local development uses yt-dlp (no key set, residential IP); only the Cloud Run deployment uses the API.
- Provider errors map to existing codes: 413 `TOO_LONG`, 422 `UNAVAILABLE`, 429 `QUOTA_EXCEEDED`, others `DOWNLOAD_FAILED`, including the provider's job error `SERVICE_TEMPORARILY_UNAVAILABLE` (retried with a new idempotency key, because the provider returns the original failed job for a repeated key; failed provider jobs did not spend a conversion when observed).

## Rationale

- The provider downloads from its own infrastructure and hosts the result, so the YouTube block on the Cloud Run IP no longer applies, and the file is fetched from the provider (not from a YouTube URL bound to the provider's IP).
- The free plan's limits are hard limits, so it cannot generate charges.
- The change is confined to the source adapter, as ADR 0008 intended.

## Consequences

- The first provider delivers YouTube's stream as is, but in its DASH container (no `FixupM4a` as yt-dlp applies), so its bytes, and the identity, differ from a yt-dlp download of the same video; Edge decodes it. The second provider's file is re-encoded (AAC 128 kbps in m4a), not YouTube's original stream. Identity stays the SHA-256 of the delivered bytes (ADR 0009); the same video may hash differently from a yt-dlp download, and across provider runs if its encoding is not stable (not verified). The project owner accepted this.
- The video URL is sent to a third party. The provider's error codes and long-term availability are not guaranteed; job-error classification is a best guess.
- The first provider's relay is run by unknown third parties (a Cloudflare Worker in front of another proxy service). Free plans: 4 requests per month (first provider) and 25 conversions / 300 requests per month (second). The local audio cache (ADR 0010) avoids spending conversions on repeated videos.
- Processing third-party content through this service is outside YouTube's Terms of Service, as with yt-dlp.

---

## Related project documents

- `AGENTS.md` — agent working rules and documentation authority.
- `PRODUCT_SPEC.md` — product behavior and MVP requirements.
- `TECHNICAL_ARCHITECTURE.md` — system boundaries, data flow, execution strategy, and unresolved architecture questions.
- `DEPLOYMENT.md` — hosting, CI/CD workflows, and operations.
