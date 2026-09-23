# Changelog

All notable changes to MusePAI are documented here.

---

## [0.5.40] — 2026-09-23 — Lyrics aligned against the audio

### Added
- **`daily/align_lyrics.py` gives generated lyrics real timings**, using torchaudio's MMS_FA forced aligner: a multilingual CTC model that takes the *known* text and finds where each word is in the audio. That is a far easier problem than transcription, which is what makes it work on singing. Timings go into `lyrics_timed` in the track's own JSON; `lyrics` is untouched, and a track that cannot be aligned keeps the even spread.
  - All three existing tracks were aligned. The first sung line is at 22.2 s, 25.4 s and 23.4 s — every one of them behind an instrumental intro the even spread knew nothing about.
  - Non-Latin scripts are romanized with `uroman` first. Without it the Thai track produced 3 alignable lines out of 34; with it, 34 of 34. Chinese, Korean and Japanese are in the rotation too and would have failed the same way.
  - A track is **refused** rather than partially aligned when fewer than 60% of its sung lines survive romanization. Timing three lines out of thirty-four would scatter them across the song and leave the rest untimed — worse than an even spread, which is at least wrong everywhere by a little.
  - `run_daily.sh` aligns after a clean run, while the audio is still local.

### Fixed
- **Every generated track had a duration of zero.** `fromJson` read duration and bpm only from the nested `metas`, but the catalogue lifts them to the top level and drops `metas`. Two symptoms, one cause: the list showed "- BPM" for every track, and the lyric spread fell back to a flat three seconds a line regardless of how long the song was.

Verified on the live site: at 0:39 the highlighted line is "Yang dulu pernah ada", whose aligned start is 38.32 s.

### Known gaps
- MMS_FA is 1.18 GB, downloaded on first use to `~/.cache/torch`.

**Git tag:** `v0.5.40-lyric-alignment`

---

## [0.5.39] — 2026-09-22 — Structure markers no longer eat the lyric timeline

### Fixed
- **Generated lyrics ran ahead of the singing.** ACE-Step's sheet has no timings, so the lines were spread evenly — including the structure markers. On a real track nine of thirty-five lines are `[Intro]`, `[Verse 1]`, `[Chorus]` and the like: nobody sings them, but each took a 5.77 s slot, putting the first sung line eleven seconds late and spending a quarter of the timeline on unsung text. Markers now take no time and carry the timestamp of the line they head, so they still show while the highlight lands on the sung line. The first sung line starts at 0:00 instead of 0:11.

### Known gaps
- This is still an even spread, now over the sung lines only. It drifts across an instrumental break, because nothing here knows where the singing actually is. Real timings need forced alignment against the audio at generation time, written into the track's own JSON — which is exactly the kind of enrichment the metadata repo exists for.

**Git tag:** `v0.5.39-lyric-timing`

---

## [0.5.38] — 2026-09-22 — Two bugs the deployment exposed

### Fixed
- **Generated tracks played silence through the equaliser.** `MediaElementAudioSource outputs zeroes due to CORS access restrictions`: a cross-origin media element without `crossOrigin=anonymous` is tainted, and a tainted element in a Web Audio graph emits silence rather than failing. The rule for setting it was "is this a proxied unlock URL", which was the same thing only while every media URL was same-origin. It is now "is this cross-origin", which covers the unlock proxy, the generated library and any CDN.
- **Generated tracks showed 暂无歌词.** Two causes, in sequence:
  - `catalog/all.json` summarises and deliberately leaves lyrics out, but the catalogue was the app's only source of them. Lyrics are now fetched per track, from the track's own JSON, once, for the track being played — so the file the app loads at startup stays small as the library grows. Both servers serve a track's JSON from the metadata tree, since the audio lives in a release.
  - The fetch then parsed that JSON back through `fromJson`, which requires a `url` the server only synthesises for catalogue entries. It returned null and the player showed no lyrics while the file had 35 lines in it. The sheet now parses on its own.

Verified in a browser on the live site: 35 lines render and scroll with playback, the current line highlighted.

**Git tag:** `v0.5.38-cors-and-lyrics`

---

## [0.5.37] — 2026-09-22 — Deployed, with no server anywhere

### Added
- **A live deployment**: the app on GitHub Pages at <https://clkhoo5211.github.io/musepai-web/>, three functions on Vercel `sin1` at `musepai-api.vercel.app`, and the library in the private `clkhoo5211/musepai-library`. No server, no disk, no rsync. Verified from that public origin in a browser: a generated track plays and seeks (177 s of 190 s), and a Netease track plays and seeks (247 s of 260 s, a real 4.16 MB file).
- `deploy/vercel/` — the Netease API, the unlock proxy and the generated library as Vercel functions, reading the library straight from GitHub. `deploy/publish-web.sh` rebuilds and republishes the frontend, and refuses to publish a bundle that does not carry the backend's address.
- CORS is locked to the Pages origin. The backend's URL is compiled into a published bundle and is therefore public; with `*` it would be an open proxy for anyone who read the JavaScript.

### What measurement changed
- **kuwo rate-limits by egress IP.** It resolved from Vercel a handful of times, then refused for eight minutes while the same call from a home connection kept working — throttling, not a block. A single failing request used to cost eight upstream calls (two retries × four providers). Now one provider is tried, resolved URLs are cached for fifteen minutes, and a retry backs off a second instead of 150 ms.
- **Only kuwo resolves at all.** pyncmd, migu and kugou fail from home and from a datacenter alike; trying them cost ~5 s per attempt and brought the throttle closer.
- **Restricted tracks come back as a 2-second stub** — exactly 16,009 bytes at 64 kbps for every id tried, from every host, including locally. Tracks Netease serves itself are unaffected. "Unlock succeeded" is not the same as "the song plays".
- Bare Vercel Functions are not Next.js: `api/[...path].js` did not populate `req.query`, and a two-segment path matched no function at all. The paths come through explicit rewrites instead.

**Git tag:** `v0.5.37-live`

---

## [0.5.36] — 2026-09-21 — Media on disk is a cache, not a requirement

### Added
- **`MUSEPAI_MEDIA_GITHUB_REPO` streams audio straight out of the metadata repo's releases, with nothing on the server's disk.** The token stays on the server, so the browser still never talks to GitHub, and the browser's `Range` is passed upstream so seeking works. The asset index is refreshed only on a miss, and at most once a minute, so a request for something that does not exist cannot hammer the API.
  - Verified in a browser against the real private repo with **zero mp3 files on disk**: all three tracks reached `canplaythrough`, played, and seeked to within twelve seconds of their end.
  - The trade is disk for latency, and it is a trade, not an upgrade: every play fetches from GitHub rather than reading a local file.

### Fixed
- **No audio or video type was in the MIME table**, so every mp3 this server has ever sent went out as `application/octet-stream` and only played because browsers sniff. Cover art and MVs would not have been so lucky. Pre-existing, and it applied to the on-disk path too.
- GitHub answers the asset API with `application/octet-stream` because that is what it is asked for; the extension is now preferred over that generic answer.

**Git tag:** `v0.5.36-stream-from-github`

---

## [0.5.35] — 2026-09-21 — Older releases were being skipped

### Fixed
- **`fetch-media.sh` only ever fetched one asset per run.** `curl` inside the `while read` loop consumed the loop's own stdin, so every asset after the first silently disappeared — on a server that means yesterday's tracks are there and the day before's never arrive, with nothing in the log to say so. Found by publishing a second day and checking that *both* came down, not just the new one. Fixed with `</dev/null`.
- `test-github-source.sh` raced its own subject: it waited for the first mp3 and then asserted the full count, so it passed or failed on download timing. It now waits for as many files as the catalogue declares.

### Verified
A new day creates its own `media-<date>` release automatically, and a fresh sync pulls every release, old and new — 4 of 4 across two days before the probe day was removed.

**Git tag:** `v0.5.35-fetch-all-releases`

---

## [0.5.34] — 2026-09-21 — GitHub can be the only source of truth

### Added
- **`library-sync` now pulls both halves from GitHub**: the tree over SSH with the read-only deploy key, and the audio out of the repository's release assets with a read-only token. A new server needs `META_REPO_URL`, `MEDIA_GITHUB_REPO` and `GITHUB_TOKEN` and nothing else — no rsync, no bucket. Media fetching runs every cycle rather than only after the tree moves, because a release can gain a cover or an MV without any commit.
- The sidecar has its own image: `alpine/git` carries neither `curl` nor `python3`, which `fetch-media.sh` needs.
- `test-github-source.sh` — opt-in, because it needs the network and real credentials. Five checks that the tree and every track's audio come down, that no audio leaks into the tree, that the catalogue joins them, and that release-sourced audio seeks.

### Verified
End to end against the real private repository, with no byte of this machine's own library involved: the full compose stack served a catalogue of three tracks assembled from a git-pulled tree and release-pulled audio, and all three played in a browser — Vietnamese, Indonesian and Thai, each reaching `canplaythrough` and advancing past 1.7 s.

Measured, since it decides the architecture: a private repository's release assets return **404 without a token**, and 200 with one, supporting `Range` and sending `access-control-allow-origin: *`. So the server fetches them and the browser never does — a token in a page is a published token.

**Git tag:** `v0.5.34-github-source`

---

## [0.5.33] — 2026-09-21 — The Pages bundle actually plays

### Fixed
- **A frontend hosted apart from the backend could not see the generated library at all.** `GeneratedCatalogApi` asked for a relative `/generated/catalog.json`, which resolves against the *frontend's* host — and a static host has no backend, so it 404'd. The base is now configurable and falls back to the origin of `UNLOCK_API_URL`, since the two are always the same backend; the catalogue's relative track URLs are made absolute client-side, so the server's output stays independent of knowing its own public address.
- **`test-split-origin.sh` had the gap that let this through**: it proved the backend sends the right headers, never that the app asks the backend at all. curl cannot see that. It now also checks the bundle carries the backend's origin and points nowhere at the frontend's own host for `/generated`.

### Verified
Cross-origin playback, measured in a real browser with the page on one origin and the backend on another: the audio reached `canplaythrough`, `currentTime` advanced to 2.44 s, and a seek to 171.92 s of a 190 s track succeeded — which exercises `Range` and CORS together, not just the headers in isolation.

**Git tag:** `v0.5.33-pages-playable`

---

## [0.5.32] — 2026-09-21 — amd64, TLS, a deploy key, classification, and a frontend that can live anywhere

### Added
- **The metadata repo is real and populated**: `clkhoo5211/musepai-library`, private, holding three tracks' metadata and a read-only deploy key for the server. `make-deploy-key.sh` creates the key, registers it, and pins GitHub's host keys after checking them against the fingerprints GitHub publishes — no trust-on-first-use. Verified by having the real `library-sync` container clone the real private repo over SSH.
- **Classification as derived indexes.** `sync/reindex-library.py` rebuilds each day's `index.json` plus `catalog/all.json` and `catalog/by-{genre,language,mood,persona}/`. Days stay the storage key because a generation date never changes, so reclassifying a genre edits one field instead of moving files and breaking their history; classification is an index rather than a tree because a track is in several categories at once. The catalogue reads `all.json` when present — one file instead of one per day, which after a year is 365 reads per request.
- **Media can live in GitHub releases.** `publish-library.sh --archive <owner>/<repo>` uploads a day's audio as release assets, which never enter git history and do not count against repository size, and `sync/fetch-media.sh` pulls them onto the server with a read-only token. Measured: a private repo's assets need a token (404 without), support `Range` (206) and send `access-control-allow-origin: *`. So GitHub can be the only source of truth — but the *server* fetches, never the browser, because a browser would have to carry the token.
- **TLS and access control**, as a `tls` compose profile running Caddy. Verified locally against its internal CA: 401 without credentials, 200 with, 401 with the wrong password, `/health` open for probes, and a byte range still 206 through TLS.
- **`CORS_ALLOW_ORIGIN`**, so the frontend can be hosted anywhere — GitHub Pages included. `test-split-origin.sh` builds the bundle against a backend on one origin, serves it from a static-only host on another, and checks all ten of: CORS on `/api`, `/unlock`, the catalogue and the audio, seeking, preflight, and that a different origin is refused.
- amd64 images: cross-building `linux/amd64` on this Apple Silicon machine works, so the arm64-only images are no longer a blocker.

### Fixed
- **`send()` was passing the content type into `writeHead`'s status-message slot**, so every JSON response went out with no `Content-Type` at all — `HTTP/1.1 200 application/json; charset=utf-8`. Browsers sniff JSON and hid it; a cross-origin frontend would not be so forgiving. Pre-existing.
- **`.gitignore`'s `/MusePAI/*` had been silently excluding the entire `deploy/` tree.** `git add -A` says nothing about ignored paths, so v0.5.27 through v0.5.31 described files that were never committed. They are in the repository now, with the secret and scratch rules placed after the un-ignore where they take effect.
- Media responses and the OPTIONS preflight now cover `/generated`, which they did not when it was always same-origin.
- `fetch-media.sh` retries: one transient failure left a track unplayable until the next sync.
- `BASIC_AUTH_HASH` was declared `:?` and compose interpolates every service regardless of profile, so adding the TLS profile made the variable required for every invocation. The Caddyfile refuses to start without it instead.

### Known gaps
- Whether the Netease API works from a datacenter IP is still unknown and untestable from here.
- A login cookie travels as a query parameter, so it lands in access logs. Pre-existing.

**Git tag:** `v0.5.32-deployable`

---

## [0.5.31] — 2026-09-21 — A seed for the metadata repo

### Added
- `deploy/library-repo-template/` — a `.gitignore` and README to start the metadata repo from. The ignore list is a backstop against a stray `git add .`, not a preference: one MV outweighs a decade of metadata, and git keeps every version forever. Verified by staging an mp3, a webp and a json into a fresh repo and confirming only the json goes in.
- Setup steps in `deploy/README.md`. The repo is independent of the server, so it can start collecting metadata before a deployment exists.

**Git tag:** `v0.5.31-library-repo-seed`

---

## [0.5.30] — 2026-09-21 — Media can live on object storage

### Added
- **`MUSEPAI_GENERATED_MEDIA_BASE_URL` puts the media on a CDN.** The catalogue then points at it and the server never touches the audio — the browser fetches it straight from the bucket, which is the only reason to use one. `publish-library.sh --r2 <rclone remote>` uploads with `rclone copy --immutable`, and `--prune` reclaims the local copy **after verifying the destination matches**, which is the point when the generating Mac is short on disk.
- `deploy/sync/cdn-stub.mjs` — a CORS-enabled stand-in for the bucket, carrying exactly the policy R2 needs (`Range` in, `Content-Range`/`Accept-Ranges` out). `./rehearse-local.sh --cdn` runs the whole stack against it, so the object-storage path can be rehearsed before a bucket exists. Both rehearsals stay offline: 14 checks and 7 checks.

### Changed
- **`publish-library.sh` now publishes media before metadata.** With object storage there is no local file for the catalogue to gate on, so the ordering is the guarantee rather than the read-time check.
- **A generated track's URL is no longer sent through `ensureWebPlayable`.** On web that wraps every absolute http(s) URL in the unlock proxy, which for a CDN means paying for the bytes twice and discarding the reason to use one. A generated URL is already what the browser should load, same-origin or not. Pinned with a test that shows what the wrapping would otherwise do.

### Fixed
- `test-sync-chain.sh` hung instead of failing: under `set -e`, `wait` on a job killed by a signal takes the script down, and re-entering `wait` from the `EXIT` trap deadlocks. All stops now go through one helper.
- The same test would pass or fail depending on whether something else held port 5498 — a squatter answered the readiness probe and then 404'd every file. It now refuses to run rather than blaming the code.

### Known gaps
- Serving media from a bucket needs CORS on that bucket; without it the browser blocks the fetch even though `curl` succeeds. The policy is in `.env.example` and the stand-in enforces it, but nothing checks the real bucket.

**Git tag:** `v0.5.30-object-storage`

---

## [0.5.29] — 2026-09-21 — Both metadata transports, written down

### Changed
- `deploy/.env.example` and `deploy/README.md` now document both shapes for the metadata repo, which differ only in `META_REPO_URL`:
  - **a bare repo on the server** — the smaller setup and the default recommendation, since the Mac already needs SSH for the media rsync and can push over the same connection;
  - **a private GitHub repo** — worth it for an off-site copy of the history and for filling in lyrics, cover art or background from another machine. Private, because GitHub's private repos are free and unlimited, so a public one costs the same and merely publishes every track.
- Documented what belongs in the repo and what does not, with sizes. Media is referenced **by filename**, never by URL, so the same repo works against any media root without an edit — which is what makes moving to another server free. One MV is larger than a decade of metadata, and git keeps every version forever.

**Git tag:** `v0.5.29-transport-docs`

---

## [0.5.28] — 2026-09-21 — Metadata rides in git, media rides in rsync

### Added
- **The library now travels as two halves.** `MUSEPAI_GENERATED_META_ROOT` and `MUSEPAI_GENERATED_MEDIA_ROOT` split the catalogue's sources; both default to `MUSEPAI_GENERATED_ROOT`, so local development is unchanged. Metadata (~40 KB/day, and the half that keeps changing as lyrics, cover art, MV and genre are filled in) rides in a git repo the server pulls; audio (~9.3 MB/day, written once) rides in rsync.
  - **No coordination is needed between the two.** The catalogue checks the media root before listing an entry, so a track appears only once both halves are present, whichever arrives first.
- `deploy/sync/pull-loop.sh` and a `library-sync` container that clones and pulls the metadata repo on a timer. It uses `fetch origin HEAD` + `reset --hard`, so it neither needs to know the remote's branch name nor can drift.
- `run_daily.sh` publishes after a clean run when `MUSEPAI_PUBLISH` is set, so a partial day never reaches the server.
- **Two local rehearsals, both offline**: `deploy/test-sync-chain.sh` (11 checks — a bare repo standing in for GitHub, both arrival orders, a metadata edit landing long after the audio, and that no audio reaches the repo; it drives the real `library-sync` container too) and `deploy/rehearse-local.sh` (5 checks — the whole compose stack against a local bare repo and the real generated library).

### Fixed
- **`publish-library.sh` was deleting the metadata repo's `.git`.** `rsync --delete` with `--include '*/'` matched `.git` and its subdirectories, so the include set emptied the repository's own database; git then reported "not a git repository". Caught the first time the sync rehearsal ran. Rules are first-match-wins, so `--exclude '.git/'` now comes first.
- `unlock-proxy` referenced `musepai-backend:latest` without a `build:` block, which is unresolvable on a first `up --build` under buildx's image store.
- `test-sync-chain.sh` exited 143 even when every check passed: killing the background server in the `EXIT` trap leaked the signal status, and under `set -e` a false `[ … ] && cmd` aborted the trap.

**Git tag:** `v0.5.28-metadata-sync`

---

## [0.5.27] — 2026-09-21 — A deployable stack, and a rewrite that dropped its path

### Added
- **`MusePAI/deploy/` — a three-container stack for a CPU-only server**: the site, the Netease API and the unlock proxy, with a `library` bind mount for generated tracks. Verified end to end in Docker: the site loads, `/api` and `/unlock` proxy, `/generated/catalog.json` lists all three tracks, and a resolved unlock URL streams `audio/mpeg` with a 206.
  - ACE-Step is deliberately absent — it needs a GPU, and Docker on macOS cannot pass Metal through either, so containerising it locally would only take the GPU away. Generation stays on the Mac; `publish-library.sh` rsyncs the result, audio first and each day's `index.json` last so the server never advertises a track whose mp3 is still in flight.
  - `netease-api` and `unlock-proxy` share one image and differ only in entrypoint; the web image builds Flutter in a multi-stage and ships a slim Node runtime.
  - The `web` healthcheck hits `/unlock/health`, not `/`: a site that serves its index but cannot reach the proxy looks healthy and plays nothing, which is exactly how playback broke in 0.5.24.

### Fixed
- **`rewriteUnlockProxyHost` discarded the unlock base's path prefix.** `.replace(path: …)` overwrote the whole path, so a base of `https://host/unlock` produced `/stream` rather than `/unlock/stream`. Same-origin deployments never saw it, because the same-origin conversion put the prefix back; a cross-origin unlock base would have failed every stream. Found by the first test written against the deployed shape.

### Known gaps
- Nothing in the stack terminates TLS; put a reverse proxy in front for a real domain.
- `PUBLIC_ORIGIN` is compiled into the bundle, so changing it needs `up -d --build web`, not a restart.

**Git tag:** `v0.5.27-deploy-stack`

---

## [0.5.26] — 2026-09-21 — A superseded playback stops instead of racing the next one

### Fixed
- **`_loadAndPlay` reported "a newer request took over" and "this track has no source" as the same `false`.** `playQueueWithFallback` could not tell them apart, so a superseded attempt made it walk to the next queue entry and start another playback — which superseded the newer one in turn. Queueing the three generated tracks produced a cascade of `stale resolve` / `skipping` pairs and an uncaught `TypeError` from a player being stopped mid-load. It now returns a three-way `_PlaybackAttempt`, and the loop stops on `superseded`. `播放全部` logs one line.
- **Skipped tracks were written to listening history.** The loop recorded every entry it walked past before knowing whether it would play; history is now written only for the track that actually started.

**Git tag:** `v0.5.26-playback-supersede`

---

## [0.5.25] — 2026-09-21 — Seeking works, and `--daemon` stops lying

### Added
- **`/generated/*` honours `Range`.** Seeking a generated track now sends one `206` for the bytes it needs instead of re-downloading the file; `bytes=N-`, `bytes=-N` and an over-long end are all handled, a request past the end gets a `416`, and a multi-range request falls back to a `200` rather than a wrong answer. Verified in the browser: a seek to 3:10 of a 3:22 track issued a single 206 and playback did not stall.
- `scripts/test-generated-range.mjs` — spins the real server on a throwaway root and asserts the status, `content-range` and body of each case.

### Fixed
- **`--daemon` on its own started `flutter run`.** It warned that the flag was "ignored" and then ran the dev server anyway: foreground, blocking, and serving without the `/unlock` proxy — the exact configuration that broke playback in 0.5.24. `--daemon` now implies `--build`, because there is nothing about `flutter run` to daemonise.
- Tightened the `/generated/*` traversal guard to compare against `GENERATED_ROOT` plus a separator, so a sibling directory sharing the prefix can no longer be read.

**Git tag:** `v0.5.25-generated-range`

---

## [0.5.24] — 2026-09-21 — The ACE-Step library plays inside MusePAI

### Added
- **The locally generated ACE-Step library is a playable source.** `flutter-web-server.mjs` serves `/generated/*` from `~/Downloads/ACE-Step-1.5/daily/library` (override with `MUSEPAI_GENERATED_ROOT`), `GeneratedCatalogApi` reads the catalogue, and `PlaybackResolver` short-circuits `source == 'generated'` the way it already did local files — same-origin, permanent, no unlock fallback to attempt.
- **Generated tracks carry their lyrics.** ACE-Step writes a lyric sheet with no timings, so `GeneratedTrack.parsedLyrics` spreads the lines evenly across the track and parks the result in a registry the player reads before falling back to the Netease API, which has no record of these tracks.
- An "AI 生成" section in the local music screen, listing every track the pipeline has produced.

### Fixed
- **Playback was completely broken and `dev-all.sh check` reported all green.** `:5199` was being served by `flutter run`, which has no proxy, so `/unlock/*` fell through to the SPA's `index.html` and every resolve failed. The check only *warned* about the missing `/unlock/health`; it is now a hard failure that names the cause and the fix (`restart --build --daemon`).
- `GENERATED_ROOT` resolved one directory too deep, so the catalogue was always empty.
- `GeneratedMusicProvider` was registered lazily, so its constructor — and the lyric registry it fills — never ran unless you first visited the library screen. Registered with `lazy: false`.

### Known gaps
- `/generated/*` ignores `Range` and always returns the whole file with a 200. Playback works; seeking within an unbuffered region will re-download.
- Queueing several generated tracks at once logs a burst of stale-resolve/skip lines. Harmless — the winning resolve still plays — but noisy.

**Git tag:** `v0.5.24-acestep-in-musepai`

---

## [0.5.23] — 2026-09-21 — Eight more screens, and a full bilingual sweep

### Changed
- **164 literals wired** across podcast, settings_profile, notifications_preferences, accessibility, import, about and shazam. `about_screen` and `accessibility_screen` are now at 0.
- 1,764 → 1,602 hardcoded literals.

### Fixed
- **A literal `\n` was reaching the UI.** Moving a Dart string with a real newline into an ARB value wrote it as a backslash followed by `n`, so the About subtitle rendered as `…music player\nSynced playback…` on one line. Fixed, and an audit found three more ARB values with the same defect left over from the first i18n wave — all three were unreferenced and were dropped.
- Two field initializers became getters again (`about_screen._items`, `import_screen._tabs`) — a field initializer runs before `context` exists. The shared const-strip had additionally left them with no declaration keyword, which is how they surfaced.
- Two threaded helpers were passed as tear-off callbacks (`onPressed: _startImport`), so adding a parameter broke the callback type; those call sites wrap in a closure now.

### Verified
A full sweep in the browser: **17 routes × 2 locales = 34 loads, zero runtime exceptions.** Screenshots of About, Accessibility, Podcast and Shazam checked by eye in English, which is how the `\n` defect and three stragglers (`大`, `设置`, and two tech-stack badges) were caught — the analyzer and the literal count both considered those files done.

### Known gaps
- `search_screen` keeps 31 literals and is deliberately left: they are seeded demo content — fake artist names, album titles and playlists with made-up usernames — the same category as the design-port screens.
- 754 literals remain in the 13 demo/design-QA screens and 61 in the platform-preview screens, both out of scope.

**Git tag:** `v0.5.23-settings-batch`

---

## [0.5.22] — 2026-09-21 — Collaborative and Concerts

### Changed
- **`collaborative_provider.dart`: 9 → 0**, **`collaborative_screen.dart`: 14 → 2**, **`concerts_screen.dart`: 11 → 0**.
- `CollaborativeProvider` follows the pattern the other providers now use: a `CollabError` code instead of a Chinese `_error` string, a `CollabActivityKind` for the two locally generated activity rows, `totalDurationMs` instead of a pre-rendered `durationLabel`, and an empty `playlistName` that the UI fills from `l10n.collabDefaultName`.
- `FormatL10n.totalDuration` absorbs the provider's private `'$h 小时 $m 分'` formatter.
- 13 new placeholder keys cover the interpolated copy on both screens — `concertsMatchReason(artist, count, pct)`, `concertsSeeAll(count)`, `collabInviteCap(name)` and so on.

### Known gaps
- `concerts_provider.dart` still holds 23 literals: venue names, festival names and badge text for generated demo listings. Mock content, left alone.
- `collaborative_screen.dart` keeps 2 literals inside a decoration block.

35 new ARB keys (1,915 → 1,950). 1,811 → 1,764 hardcoded literals. `flutter analyze` 0 errors, `flutter test` 35/35. Verified in a browser: the concerts screen reads "Based on your last 7 days · artists like 陈奕迅 first", "See all 6 →", "City 上海" and dates as "Sat, Nov 14" — artist and venue names correctly left as the Netease data they are.

**Git tag:** `v0.5.22-collab-concerts`

---

## [0.5.21] — 2026-09-21 — Friend activity: kinds instead of string matching

### Fixed
- **Two matchers were keyed on rendered Chinese text and would have broken silently once it was localized.** `models/user_insights.dart` *builds* the friend-activity action text, and two consumers then pattern-matched it:
  - `NotificationsProvider._categoryForAction` matched `'正在听'`, `'歌单'`, `'评论'` to pick a notification category
  - `CollaborativeProvider` did `if (!event.actionText.contains('歌单')) continue;`

  `FriendActivityItem` now carries a `FriendEventKind` enum plus the resource name, both matchers switch on the kind, and `lib/l10n/friend_activity_l10n.dart` renders the wording. A correction to v0.5.15, where a comment claimed `_categoryForAction` matched "the Chinese action text the Netease feed returns" — it matched text this app generates.

### Changed
- `models/user_insights.dart`: 11 → 0 hardcoded literals. Free text from the API (`msg` / `title` / `actName`) is still passed through verbatim — it is the user's own words — and only the generated wording is localized.
- `CollaborativeActivity`, `PartyChatMessage` and `AppNotificationItem` carry the event instead of pre-rendered text.

### Added
- `test/friend_event_kind_test.dart` — event type → kind and resource name for the six types that matter, the free-text passthrough, and that an event with neither a known type nor text is dropped. Suite 32 → 35.

### Verification note
The friend-activity feed only renders for a signed-in account, and the party and collaborative screens are login-gated, so this change is covered by the unit tests and the analyzer rather than a browser screenshot. Faking a session to reach those screens is what produced the false "white screen" bug in v0.5.18, so it was not repeated.

11 new ARB keys (1,904 → 1,915). 1,840 → 1,811 hardcoded literals. `flutter analyze` 0 errors, `flutter test` 35/35.

**Git tag:** `v0.5.21-friend-activity-kinds`

---

## [0.5.20] — 2026-09-21 — Shared locale-aware formatting

### Added
- **`lib/l10n/format_l10n.dart`** — one place for relative time, track counts and compact numbers, all taking an `AppLocalizations`.
  - **Compact counts are genuinely locale-specific, not a suffix swap.** Chinese groups by 万 (10⁴) and 亿 (10⁸); English by K/M/B (10³/10⁶/10⁹). The thresholds differ, so the helper branches on locale rather than substituting a word. `45300` now renders `4.5万` in Chinese and `45.3K` in English.

### Changed — three duplicated implementations collapsed
- `models/comment.dart` had its **own** relative-time formatter, a near-copy of `utils/relative_time.dart` with different cut-offs. It now exposes a raw `DateTime` and the UI calls `FormatL10n.relativeTimeLong`.
- `mv_screen.dart` and `podcast_screen.dart` each carried a private `_formatCount` with 万/亿 baked in. Both deleted.
- 9 `'$n 首'` sites across 6 files go through `FormatL10n.trackCount`.
- `CollaborativeProvider` carries `addedAt` / `whenMs` instead of pre-rendered text; `NotificationsProvider` feed items carry `timeMs`.

### Known gaps
- `messages_provider.dart` is the **only** remaining caller of the Chinese `utils/relative_time.dart`. `Conversation.timeLabel` holds a clock on some paths and a relative time on others, so separating them is its own change. `relative_time.dart` now carries a header saying so, and that `formatChatClock` / `formatDurationMs` beside it are locale-neutral and fine.
- 1,863 → 1,840 hardcoded literals; 162 of them interpolated. 136 of the 160 interpolated shapes outside the demo screens are one-offs, so the remaining work there is per-string rather than clusterable.

13 new ARB keys (1,891 → 1,904). `flutter analyze` 0 errors, `flutter test` 32/32.

**Git tag:** `v0.5.20-shared-formatting`

---

## [0.5.19] — 2026-09-21 — Shared catalog actions localized

### Changed
- **`lib/utils/catalog_actions.dart`: 46 → 0 hardcoded literals.** This is the shared action layer — download, favorite playlist/album, share, save-queue-as-playlist, follow artist, shuffle, open MV, and the song context sheet — reached from a dozen screens, so its toasts were leaking Chinese everywhere regardless of which screen the user was on.
- 8 of those needed real ARB `@placeholders` rather than being skipped: `catQueuedForDownload(added, total, label)`, `catSavedAsPlaylist(name)`, `catShufflePlaying(count)`, `catLinkCopiedNamed(title)`, `catPrivateFmName(date)` and three `{error}` variants.
- `queueForDownload`'s `label` parameter defaulted to the literal `'列表'`; it is now nullable and falls back to `l10n.catDefaultListLabel`.

### Translation notes
Three of the machine translations were wrong in context and were corrected by hand:
- `已插入下一首` and `已加入队列` both came back as "Added to queue"; the first is now "Playing next".
- `已取消收藏` is used for **both** a playlist and a song, so it cannot say "Playlist unfavorited" — it is the context-neutral "Removed from favorites".
- `已取消关注` → "Unfollowed" rather than the literal "Following cancelled".

39 new ARB keys (1,852 → 1,891). `flutter analyze` 0 errors, `flutter test` 32/32. Verified in a browser in both `zh_TW` and `en`: the favorite toast reads 請先登錄後再收藏歌單 / and the download toast renders all three placeholders — "10 queued for download (10 total) · 飙升榜", with the chart name correctly left as the Netease data it is.

### Known gaps
1,909 → 1,863 hardcoded literals. `context_menu_screen.dart` still passes a hardcoded label to `queueForDownload`; it is one of the six design-QA screens deliberately out of scope.

**Git tag:** `v0.5.19-catalog-actions`

---

## [0.5.18] — 2026-09-21 — Boot survives an unreadable stored session

### Fixed
- **`AuthProvider.initialize()` could throw before `runApp`.** It read the stored cookie and profile with `prefs.getString(...)` unguarded. A stored value that is not a valid encoded String makes `getString` throw a cast error; `main()` awaits `initialize()` before `runApp`, so the app never rendered and the only way out was clearing site data. It now drops the unreadable entries and starts as a guest — the same tolerance the profile `jsonDecode` one line below already had.
- New `test/auth_provider_corrupt_prefs_test.dart` pins both paths. Verified the test is load-bearing: it fails when the guard is reverted. Suite 30 → 32.

### Not a bug, for the record
This was investigated as "the app white-screens on an invalid Netease cookie". **It does not.** An expired or revoked cookie is a well-formed String and the app boots, signs in, and degrades on the API response — confirmed in a browser with 0 exceptions. The original white screen came from a hand-written `localStorage` value: `shared_preferences` on web stores a String **JSON-encoded** (`"\"MUSIC_U=...\""`), and a bare `MUSIC_U=...` written by hand is not a valid encoded String. The crash was in the diagnostic, not the product.

The guard above is still worth having — a partially-written or migrated entry produces the same unrecoverable boot — but it is defensive hardening, not a fix for the reported scenario.

**Git tag:** `v0.5.18-boot-resilience`

---

## [0.5.17] — 2026-09-21 — Paid funnel localized

### Changed
- **`vip`, `billing`, `checkout` and `gift_card` screens: 182 → 14 hardcoded literals.** The provider half was already done in v0.5.14, so these four now render fully from ARB in all three locales.
- **`gift_card_screen`** needed three State fields turned into getters — a field initializer runs before `context` exists, so `final _tabs = [... l10n ...]` cannot compile.
- **`gift_card_screen` selections are indices, not labels.** `_sendTime` / `_cover` compared the *rendered string* to decide which chip was selected, so switching language would silently drop the selection. They are `_sendTimeIndex` / `_coverIndex` now.

### Fixed
- **`billing_screen._statCard` double-wrapped its card in `Expanded`** — the non-compact branch returned `Expanded(...)` and `_buildStatsStrip` wrapped it again, producing "Competing ParentDataWidgets" at runtime. Pre-existing; surfaced by the new test. `_statCard` now returns a plain container in both branches, matching the compact path.

### Added
- **`test/paid_funnel_screens_test.dart`** — 8 cases pumping all four screens in `en` and `zh`. These routes are login-gated, so they cannot be reached in a browser as a guest; this is how they get checked after an i18n change. Scope is deliberate: it asserts the tree **builds** (no missing getter, no null, no bad interpolation), not that it lays out. Pumped bare they lack `AppShell`'s constrained centre column, so RenderFlex overflows here are a harness artifact and are ignored. Suite is 22 → 30.
- **`scripts/i18n_wire_llm.py`** — the per-file LLM wiring pass, kept alongside `i18n_thread_l10n.py` now that both are proven.

### Fixed in the tooling
- `i18n_thread_l10n.py` treated a *call site* whose last argument is a closure — `_row(context, 'x', () {` — as a declaration, because the line ends in `{`. It threaded such "helpers" and corrupted every call. The return type must now start with a word character. It also deduplicates by name, so a helper matched twice is not threaded twice.

### Known gaps
2,077 → 1,909 hardcoded literals.

**Git tag:** `v0.5.17-paid-funnel`

---

## [0.5.16] — 2026-09-21 — Settings and podcast subtitles

### Fixed
- **`settings_display_screen.dart` passed the same key as both `title` and `subtitle`** to `ShellScreenTopBar`, so the standalone Language & Display header would render its title twice. Added `settingsLanguageDisplaySubtitle` ("Interface language & Chinese display"), matching the sibling style of `settingsSubtitle` ("Account & preferences") and `playbackSubtitle` ("Quality & behavior"). Predates the current i18n work — already present at v0.5.10.
  **Not user-visible today:** both call sites construct the screen with `embedded: true`, and the top bar only renders on the non-embedded path. That path is a supported-but-unused mode shared by five settings sub-screens, so it is fixed rather than deleted.
- **`podcast_screen.dart` had a hardcoded Simplified fallback subtitle** (`'发现 · 分类 · 电台'`), which showed as a Simplified island in the Traditional UI. Now `l10n.podcastSubtitle`, with a real Traditional value. This one *is* user-visible.

2 new ARB keys. `flutter analyze` 0 errors, `flutter test` 22/22. Verified in a browser at mobile width.

**Git tag:** `v0.5.16-subtitles`

---

## [0.5.15] — 2026-09-21 — Providers localized; Traditional Chinese repaired

### Fixed
- **Traditional Chinese was rendering as a mix of scripts.** 691 of the `app_zh_TW.arb` values were still byte-identical Simplified, so a screen showed Traditional chrome with Simplified islands — visible on the home screen as 工作时段推荐 sitting among 繼續 / 佇列 / 專輯. `tool/fill_zh_tw.dart` converts them with the same `ChineseHelper` the app already ships and leaves hand-translated values alone.
- **The script converter emitted a tofu character.** `ChineseHelper` maps 跨 to the rare extension character 𫏥; 跨 is unchanged in Traditional. A scan of all 922 distinct hanzi in `app_zh.arb` found this to be the only bad mapping. Corrected in the fill tool **and** in `DisplayPreferencesProvider.formatDisplayText`, so the runtime 统一为繁体 setting stops producing it for users.

### Changed
- **DisplayPreferencesProvider** — `appLanguageLabel` / `chineseScriptLabel` deleted; they duplicated the screen's already-localized chip helpers and only two toasts used them.
- **HomeProvider** — `timeBasedRecommendLabel` (the 今晚推荐 heading) moves to `lib/l10n/home_bucket_l10n.dart`.
- **NotificationsProvider** — generated notifications carry a `NotificationTemplate` enum; feed-derived items still pass their own text through.
- **AiDjProvider** — status and commentary become an `AiDjStatus` enum plus the data they interpolate; queue items carry `isLive`; notes carry an `AiDjNoteKind`.
- **ConcertsProvider** — `_nextMonthLabel` returns a `DateTime`; `concerts_screen` formats it with `DateFormat.MMMEd(locale)` so weekday and month names follow the active locale.

### Deliberately not translated
Now commented in place so they are not "fixed" later — these are matching data, and translating them breaks behaviour:
- `HomeProvider._keywordsForBucket` — matched against Netease playlist names and copywriter text
- `NotificationsProvider._categoryForAction` — matches the Chinese action text the feed returns
- `AiDjProvider` fallback songs — real track, artist and album metadata

### Added
- `tool/fill_zh_tw.dart` — refills `app_zh_TW.arb` from `app_zh.arb`, skipping hand-translated values.
- 46 new ARB keys (1,806 → 1,852).

### Known gaps
2,107 → 2,077 hardcoded literals. Providers are down to 152, most of it demo content in `concerts_provider` and the deliberate matching data above.

**Git tag:** `v0.5.15-providers-and-zh-tw`

---

## [0.5.14] — 2026-09-21 — CommerceProvider localized; login and library screens

### Changed
- **`CommerceProvider`: 122 → 0 hardcoded literals.** The provider was composing user-visible strings, so its Chinese leaked to the UI no matter how many screens were localized. Providers have no `BuildContext`, so this is a data/presentation split, not a translation:
  - `PaymentMethod` keeps only `iconBg`; `SubscriptionPlan` keeps tier / cycle / days / prices; a new `PlanCycle` enum replaces the `'月'/'季'/'年'` strings
  - `PlanCatalogEntry` drops subtitle, compareNote and both feature lists
  - `CouponResult` / `RefundResult` carry a `CouponCode` / `RefundCode`; `CheckoutResult` carries the method; Netease tier is a `NeteaseTier` enum; seeded demo orders carry a `MockOrderKey`
  - `accountLabel` splits into `accountNickname` + `neteaseTier`
  - CSV export writes the stable enum name, not a localized status
  - All display text moves to **`lib/l10n/commerce_l10n.dart`**, following the existing `*_l10n.dart` convention
- **Login screen fully localized.** `lib/widgets/auth/auth_ui.dart` had no `AppLocalizations` at all. The gradient headline splits into prefix + highlight, with the trailing space carried only by English so "and leave it to AI" spaces correctly and 剩下的交给 AI does not gain a gap.
- **Local Music, Toplist, Lyric, MV shell, bottom nav, queue drawer, sleep timer, playback quality, search, artist, album, settings display** wired the same way.

### Added
- **`scripts/i18n_thread_l10n.py`** — adds `AppLocalizations l10n` as a first parameter to private `_helper` methods holding UI literals and rewrites their call sites. Private helpers are file-local, so this is safe. This was the missing transformation: it is why earlier batch passes could only reach ~4% of sites.
- 90 new ARB keys (1,716 → 1,806), all three locales in sync.

### Fixed
- The shared const-strip turned `const sample = '…'` into `sample = …`, dropping the declaration keyword; it now leaves a `final`. It also no longer strips `const` from a `const Foo({super.key})` constructor declaration, which previously broke every `const Foo(...)` call site in other files.

### Known gaps
- **2,178 → 2,132 hardcoded literals.** The count moves slowly because the remaining sites are mostly interpolated strings needing ARB placeholders, demo content in the eight design-port screens, and logic strings that must not be replaced.
- `app_en.arb` remains 0% Chinese; every wired surface is correct in both locales.

**Git tag:** `v0.5.14-commerce-and-hot-screens`

---

## [0.5.13] — 2026-09-20 — i18n tooling repaired; hardcoded-literal work scoped

### Fixed
- **`scripts/i18n_wire_screens.py` could never process Chinese at all.** Three bugs, each hiding the next:
  - `main()` skipped every file already containing `app_localizations` — 87 of 95 screens, so the script was a no-op on almost the whole codebase
  - `slug()` deliberately kept CJK, emitting keys like `scr_equalizer_预设_9cdfce`, not a legal Dart identifier
  - `save_arb()` wrote `@metadata` keys unquoted, producing invalid JSON that made `flutter gen-l10n` fail outright
- Widened the replacer set to `showAppToast(context, …)`, `hintText:` and `labelText:`.

### Added
- **`payment_success_screen.dart` fully localized** as a reference closed loop — zero hardcoded Chinese, including 3 interpolated strings converted to real ARB `@placeholders`, and `_buildPerk` threaded with `AppLocalizations`. Verified in a browser in both locales.
- ARB grew 1,456 → 1,668 keys, all three locales in sync, `app_en.arb` still 0% Chinese.

### Known gaps — the headline number did not move much
**2,361 → 2,300 hardcoded Chinese literals (61 removed, 2.6%).** Measured, not estimated:
- **No mechanical path exists.** Position-based replacement reaches ~4% of sites. The blocker is that private helper methods (`_buildBreakdownRow`, `_AboutNavItem`, …) take no `AppLocalizations`, so literals in their bodies have no `l10n` in scope. Threading it through every helper signature is the actual work.
- **Measured cost: ~45 min per 314-line file / 17 edit sites**, nearly all of it hand-verification, not translation. Extrapolated to 125 files: 80–100 hours.
- **324 literals live in `lib/providers/`** with no `BuildContext`. Even with every screen done, these still leak to the UI (e.g. the payment screen still shows `Sona Plus 连续季度` and `微信` from `CommerceProvider`). Needs an architectural decision — return keys and localize at the widget layer — not a rewrite.
- **623 of the literals sit in 8 demo-heavy design-port screens** (`modals_screen` 144, `achievements_screen` 111, `theme_store_screen` 83). This is mock content; translating it has little value.
- 124 literals need ARB placeholders; ~47 are logic (map keys, `case` labels, `==` comparisons) and must never be replaced.

Suggested next scope, by value rather than count: the high-traffic providers first, then the 15–20 screens users actually visit — roughly 500–700 literals, ~20 hours — and skip the demo screens.

**Git tag:** `v0.5.13-i18n-tooling`

---

## [0.5.12] — 2026-09-20 — English locale completed (ARB)

### Changed
- **`lib/l10n/app_en.arb`** — all **486** values that were still Simplified Chinese are now English (1,456/1,456 keys translated). Regenerated `AppLocalizations` / `AppLocalizationsEn` with `flutter gen-l10n`.
- Translated locally with **Ollama `qwen3.5:9b`** (`think: false`, batches of 25, ~1.3 s/key, ~10 min total). Nothing left the machine.

### Verified
- 0 CJK characters remain in `app_en.arb`; structural validation passed on all 486 (escape sequences, `·` separators, `$` interpolations and product names — MusePAI, Sona, AlgerMusicPlayer, Flutter, Netease, CarPlay — all preserved)
- `flutter gen-l10n` clean; `flutter analyze` unchanged at 327 issues; `flutter test` 22/22
- Checked in a browser: switching to **English** in 设置 → 语言与显示 produces a correct English shell, settings, player bar and guest banner

### Known gaps
- **~2,361 hardcoded Chinese string literals remain across 125 Dart files** — these were never in the ARB, so this release does not touch them. Worst offenders: `modals_screen.dart` (144), `commerce_provider.dart` (122), `achievements_screen.dart` (111), `theme_store_screen.dart` (83). The English locale is correct wherever a screen goes through `AppLocalizations`, and still Chinese wherever it does not (e.g. the home screen's 今晚推荐 heading).
- `settings_display_screen.dart` — the 预览 / 原文 labels on the script-conversion card are hardcoded

**Git tag:** `v0.5.12-i18n-en-complete`

---

## [0.5.11] — 2026-09-20 — Architecture docs, green tests, runnable clone

### Added
- **`docs/INDEX.md`** — master documentation index, linked from `README.md`; classifies every doc under `architecture/`, `reference/`, `operations/`, `project/`
- **`docs/architecture/00-OVERVIEW.md`** — three-process runtime, architectural shape, and the repo-layout trap
- **`docs/architecture/diagrams/`** — four self-contained interactive HTML diagrams (system architecture, playback URL resolution, data/persistence flow, user session lifecycle)
- **`docs/reference/01-06`** — screens and the 86-index shell switch, all 30 providers, audio engine, an exhaustive ~100-endpoint API table, data models and the 22 `SharedPreferences` keys, design tokens and the five conditional-export dispatchers
- **`docs/project/02-KNOWN-GAPS.md`** — adversarially verified gaps register; overrides conflicting statements elsewhere
- **`docs/project/03-OLLAMA-OPTIMIZATION-PLAN.md`** — forward plan for local-model work
- **`scripts/bootstrap.sh`** — clone the ref backend pinned to `v5.1.0`, install the vendored harness, `npm install`, `flutter pub get`; idempotent
- **`scripts/ref-backend/`** — `dev-web.sh`, `music-unlock-proxy.mjs`, `start-netease-api.mjs` vendored into this repo. These 624 lines existed in **no git history anywhere** — not upstream, not committed here — only as untracked files inside the gitignored `AlgerMusicPlayer_ref` checkout. This repo is now their only home.

### Fixed
- **`flutter test` was red** — `test/widget_test.dart` was still the stock counter template, which pumps `MusePaiApp` bare and throws (`MusePaiApp` is a `Consumer<DisplayPreferencesProvider>`). Replaced with a real smoke test asserting the shell mounts. **22/22 pass.**
- **`scripts/dev-all.sh` / `scripts/dev-stack.sh`** — missing ref checkout now fails with a message pointing at `bootstrap.sh` instead of an opaque `cd` error under `set -e`

### Changed
- **`lib/main.dart`** — async-free providers extracted to `appProviders()` so the test does not duplicate the list; the list itself is unchanged
- **`README.md`** — first-time setup section

### Verified
- End to end in a browser: all three services up, playlist and playback working, the unlock chain falling through to GD Music, guest trial counting down
- The `desktop_player_bar.dart:252` RenderFlex overflow is a **widget-test artifact only** — the real browser lays out cleanly at 800×600

### Known gaps
- `scripts/screen_audit.py` still exits 1 with 3 issue buckets; no accepted baseline recorded
- `flutter analyze`: 327 issues (0 errors), mostly deprecated `withOpacity`
- No CI gates any of the three checks
- `app_en.arb`: 486 of 1,456 values still contain Chinese

**Git tag:** `v0.5.11-docs-bootstrap`

---

## [0.5.10] — 2026-06-09 — i18n (zh default), dev-only QA nav

### Added
- **Flutter gen-l10n** — `lib/l10n/app_en.arb`, `app_zh.arb`, `app_zh_TW.arb` (~1450 keys); `l10n.yaml` + generated `AppLocalizations`
- **Screen wiring** — bulk `scr_*` keys + `scripts/i18n_*.py` helpers; shell chrome, settings, player, home, playlists, and most `lib/screens/*` use `AppLocalizations`
- **Web equalizer bridge** — `web_audio_equalizer.dart` + platform `equalizer_service` stub/IO split

### Changed
- **Default app language** — `DisplayPreferencesProvider` defaults to **简体中文** (`zh-Hans`); legacy `system` prefs migrate to `zhHans` on load; `MaterialApp` falls back to `Locale('zh')`
- **Sidebar nav** — **设计 QA** and **平台预览** sections only when `kDebugMode` (`ShellMenu.showDeveloperMenuSections`); production/release builds show product nav only
- **Settings** — CarPlay / Android Auto / lockscreen / widgets / wearables preview tiles hidden outside debug; **Driving mode** stays visible
- **Design QA strip** — banner clarifies preview pages are not production entry points
- **Docs** — `MODULE-ROADMAP.md`, `DEV-RUNBOOK.md`, `PRODUCTION-BUILD.md`, `DOCKER.md`, `AGENTS.md` updated for i18n + debug nav

### Known gaps
- Many `scr_*` keys in `app_en.arb` still contain Chinese placeholders — English locale may show Chinese on some screens until translated
- Platform preview routes (shell 73–85) are **in-app mockups**, not native CarPlay / Android Auto auto-detection
- Hardcoded copy remains in some widgets (e.g. design QA galleries, brand tokens)

**Git tag:** `v0.5.10-i18n-zh-default`

---

## [0.5.9] — 2026-06-08 — Ref depth: download, local music, sleep timer

### Added (continued)
- **Settings playback tab** — `settings_playback_screen`: default quality, crossfade/gapless, reparse current song, EQ link
- **PlayerProvider** — `playbackQualityLevel`, `reparseCurrentSong()`, quality passed to resolver

### Added
- **Download pipeline** — `DownloadProvider` task queue, native `download_service_io`, live progress on `download_screen`
- **Local music** — `LocalMusicProvider`, native folder/file scan, `Song.fromLocalFile`, web stub
- **Sleep timer** — `PlayerProvider` timer modes, `sleep_timer_sheet`; player transport + modals gallery hooks
- **Context menu actions** — `catalog_actions` + `context_menu_screen` wired to play/queue/favorite/download
- **Settings cache tab** — `settings_cache_screen` (storage stats, clear completed, link to download manager)

### Changed
- **Settings → 缓存 / 下载** — opens cache panel instead of jumping straight to download screen
- **Docs** — `MODULE-ROADMAP.md`, `INTEGRATION-PLAN.md` reflect Phase 6 status

### Known gaps (unchanged)
- Web: no disk download or local file scan
- LxMusic strategy, reparse UI, donation tab still deferred

---

## [0.5.8] — 2026-06-08 — Platform preview screens, web playback fix, QA

Session end: dev stack stopped; resume with `./scripts/dev-all.sh restart --build --daemon`.

### Added
- **Platform preview screens (shell 73–85)** — CarPlay (73–76), wearables player (77), context menu (78), Android Auto home/player (79–80), lockscreen (81), Dynamic Island (82), iOS widgets (83), wearables home/notifications (84–85)
- **`lib/screens/platform/`** — `auto_home_screen`, `auto_player_screen`, `lockscreen_screen`, `dynamic_island_screen`, `widgets_screen`, `wearables_home_screen`, `wearables_notifications_screen`, `platform_preview_frames.dart`
- **Shell deep linking** — `?shell=N` query param in `main.dart` for direct route QA (e.g. `http://localhost:5199/?shell=82`)
- **QA script** — `.gstack/qa-platform-test.sh` + report under `.gstack/qa-reports/` (18 platform routes smoke-tested)
- **Web playback stack** — `scripts/flutter-web-server.mjs` (static `build/web` + `/unlock/*` proxy); `scripts/dev-all.sh` with `--daemon` mode
- **Unlock URL helpers** — `play_url_utils.dart`: same-origin `/unlock/stream`, `127.0.0.1` → `localhost` migration on web
- **Docs** — `docs/WEB-PLAYBACK-DEV.md`; `AGENTS.md` agent restart rules; unlock notes in `lib/services/README.md`

### Fixed
- **Web audio `ERR_SSL_PROTOCOL_ERROR`** — Chromium HTTPS-upgrades `127.0.0.1`; app now uses `http://localhost:5199` + relative `/unlock/stream?url=…`
- **`Failed to load URL`** on play — unlock proxy reached via same-origin path through Flutter web server
- **`dev-all.sh check`** — Flutter health probe uses `localhost` (server binds localhost only)
- **PlayerProvider** — `togglePlay()`, `skipNext()`, `skipPrevious()` for platform preview transport controls
- **Providers wired** — concerts, party/collaborative controls, rewards/family/gift_card commerce screens

### Changed
- **Daily dev command** — prefer `./scripts/dev-all.sh restart --build --daemon` over foreground `flutter run` for stable web playback
- **`.dev/dart_defines.json`** — `UNLOCK_API_URL=http://localhost:5199/unlock` (same-origin on web)

### Known gaps (unchanged — see `MODULE-ROADMAP.md`)
- Platform screens are **in-app design previews**, not native CarPlay / Android Auto / OS widgets
- Commerce checkout: demo flow
- Sidebar click QA unreliable on Flutter web — use `?shell=N` deep links
- Agent shell may kill `nohup` daemons (exit 137); run restart in your own terminal for persistence

---

## [0.5.7] — 2026-06-07 — Route hygiene, auth refresh, AI pulse rhythm

Tagged as `v0.5.7-rhythm-routes-auth`.

### Added
- **Shell route manifest** — registry + `designQaRoutes` (empty, error, loading, modals, mini-player, context menu)
- **Route hygiene audit** — `scripts/screen_audit.py --route-only`; `test/shell_route_hygiene_test.dart`
- **Audio visualizer** — Web Audio FFT on web; onset-aligned `RhythmEngine` fallback on native
- **Auth UI kit** — shared `auth_ui.dart`; refreshed login, register, and guest screens

### Fixed
- **AI 情绪脉冲** — bars follow real tempo (FFT + lyric onsets) instead of fake `songId % 64` BPM
- **Playback clock** — interpolated `playbackPosition` for smooth rhythm UI between stream ticks
- `context_menu_screen` and gallery QA screens reachable from browse drawer
- **Docs** — `docs/DEV-RUNBOOK.md` full cold-start / shutdown procedure

---

## [0.5.0] — 2026-06-07 — Phase 5: Shell navigation, guest mode, catalog wiring

Tagged as `v0.5.0-phase5` with batch tags `v0.5.0-docs` … `v0.5.6-tests`.

### Added
- **Shell navigation** — `ShellRoutes` (0–77), `shell_menu`, mobile browse drawer, queue drawer, mini player bar
- **Guest mode** — `GuestSessionProvider`, `GuestPlaybackGuard`, `guest_policy.dart`, onboarding flow
- **73 screens** — commerce, social, AI shells, CarPlay/wearables in-app UI, settings sub-pages
- **Providers** — concerts, party, collaborative, messages, comments, download queue, recognition, etc.
- **Playback source keys** — live play/pause on toplist, artist, podcast radio
- **Commerce sync** — `userPointBalance` from `/user/detail`; rewards/family derived data
- **Services** — `comment_api`, `recognition_api`, `gd_music_api`, audio fingerprint (native/stub)
- **Tests** — lyric parser, equalizer presets, WAV PCM; `scripts/screen_audit.py`
- **Docs** — `DATA_SOURCES.md`, `MODULE-ROADMAP.md`, `INTEGRATION-PLAN.md` refresh

### Fixed
- Onboarding `RenderFlex` overflow on narrow viewports (content-width layout)
- Home desktop layout uses effective content width, not full viewport
- Toplist / artist / podcast play buttons now reflect real playback state

### Known gaps (documented, not in this release)
- Download and local music: UI only — no file I/O (ref parity pending)
- Commerce checkout: demo flow
- OS design surfaces (lockscreen, widgets, Android Auto): not implemented

*(Context menu + design QA routes added in v0.5.7 — see entry above.)*

---

## [0.2.0] — 2026-06-06 — Phase 2: App Shell & Core Screens

### Added
- **App Shell** (`main.dart`) — `AppShell` with `IndexedStack` routing, desktop sidebar (248px) and mobile bottom nav.
- **HomeScreen desktop layout** — full 3-column layout mirroring `web/home.html`:
  - Hero card with AI 情绪脉冲 preview and mode tags
  - 4-col responsive album grid (今晚推荐)
  - Queue list with AI-curated track reasons
  - Activity stats grid (3 cards)
  - Right rail: friends panel, active lyric box, device/quality panel
- **SearchScreen desktop layout** — mirroring `web/search.html`:
  - Semantic search headline with description
  - Stateful filter chip row (7 filters) + suggestion chips (4)
  - Best Match card with album art and action buttons
  - Numbered results table (title, artist, album, duration, match score)
  - Intent + artist side panels
  - Right aside (queue mini-cards, device relay, search history)
- **PlayerScreen desktop layout** — mirroring `web/player.html`:
  - Split hero: album art panel + meta panel with resume strip + track stats
  - Versions table + team credits module grid
  - Right rail with lyric lines, queue, tabs
  - Full bottom transport bar (shuffle / prev / play / next / repeat + seeker)
- **Google Fonts Inter** — added `google_fonts: ^8.1.0`, wired into `ThemeData`.
- **`MusePaiTheme.aiGradient`** — `#7C3AED` → `#DB2777` gradient constant.
- **Material3** — `useMaterial3: true` in `ThemeData`.

### Fixed
- Replaced all `print()` calls with `debugPrint()` across `player_provider.dart` and `netease_api.dart`.
- Replaced all deprecated `.withOpacity()` calls with `.withValues(alpha:)` in `account_security_screen.dart`.
- Replaced deprecated `Switch.activeColor` with `Switch.activeThumbColor`.
- `flutter analyze` — **0 issues** (was 19 infos/warnings).

### Changed
- `theme.dart` — switched to `Material3`, added `GoogleFonts.interTextTheme`, added `aiGradient`, standardised `SliderTheme`.
- `main.dart` — replaced standalone scaffold with `AppShell` (desktop sidebar + mobile bottom nav).
- `home_screen.dart` — full rewrite adding desktop layout; mobile layout preserved.
- `search_screen.dart` — full rewrite with stateful filter/suggestion chips and desktop 3-col layout.
- `player_screen.dart` — full rewrite with desktop layout, lyrics rail, and bottom transport bar.

---

## [0.1.0] — 2026-06-06 — Phase 1: Foundation

### Added
- Flutter project initialized (`musepai`) with Web, Windows, macOS, Linux, Android, iOS targets.
- Dependencies: `provider`, `go_router`, `just_audio`, `http`.
- `theme.dart` — color tokens, gradients, `ThemeData` (dark).
- `ResponsiveWrapper` — breakpoint switcher (mobile / tablet / desktop).
- `PlayerProvider` — `ChangeNotifier` wrapping `just_audio` `AudioPlayer`.
- `NeteaseApi` — `getMusicDetail()` + `getMusicUrl()` calling localhost:3000.
- `Song` model with `fromJson` factory.
- `HomeScreen` — mobile layout with top bar, search field, resume card, mood card, pill grid, scrollable lists, mini player.
- `AccountSecurityScreen` — settings and security options screen.
- `BottomNav` widget.
