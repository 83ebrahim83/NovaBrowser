# Security backports

The public record [decision 0016](decisions/0016-pin-chromium-140-and-backport.md)
requires: which CVEs Cobalt has taken, from which upstream commits, in which
release, and what it has not taken and why. Cobalt is built on Chromium 140 and
takes security fixes by backport. This page is how to check that claim.

How fixes are found, adapted and landed is in [`backporting.md`](backporting.md).
Each patch file in `patches/security/` carries its own provenance header.

## 0.4.5 (draft)

Six backports taken from the watcher's September 2026 Chrome 155/156 releases,
and three held. Each was adapted to M140 and verified to apply cleanly against
the checkout at `/opt/cobalt/chromium/m140/src`. The series, with all nine in
it, was run twice, and the second pass reported every backport already applied,
so no two share diff context. Every file the last five touch compiles in
`out/Default`, and a full build carrying all nine succeeded on 2026-10-08. Five
more candidates were evaluated and deferred, and one does not apply.

| Issue | CVE | Severity | Component | Upstream | Backport |
|---|---|---|---|---|---|
| #152 | CVE-2026-91726 | Critical | Out of bounds read in WebGL (Imagination GLES cache) | src `416eced3da89` on refs/branch-heads/7977 (M152) | adapted; Android-only |
| #164 | CVE-2026-91724 | High | Use after free in Input (event queues) | src `f1ba7dd9b41c` on refs/branch-heads/7977 (M152) | adapted; tests omitted |
| #166 | CVE-2026-91728 | High | Integer overflow in V8 (`FeedbackVector::SizeFor`) | v8 `d1ce50477c2e` on refs/branch-heads/15.2 (M152) | adapted; tests omitted |
| #234 | CVE-2026-95339 | Critical | Use after free in ServiceWorker (reentrant job abort) | src `bd4a88f4205c` on refs/branch-heads/7977 (M152) | adapted; tests omitted |
| #238 | CVE-2026-95357 | Critical | Out of bounds write in GPU (ANGLE GL, `glTexImage3D` depth increase, Qualcomm Android) | angle `8983f40bc8ef` on chromium/8059 (M155) | logic verbatim; feature placed apart from other ANGLE backports |
| #180 | CVE-2026-91747 | High | Use after free in Skia (glyph atlas generation on a partial update) | skia `4ca59aa76296` on chrome/m152 | logic verbatim; M140's `GrAtlasManager.cpp` / `TextAtlasManager.cpp` |

- **#238**'s feature sits after `ResetTexImage2DBaseLevel` in `util/autogen`
  rather than in its alphabetical slot, which is the line next to #4's
  `ReattachTextureToFboAfterLayerIncrease`. Those maps are keyed, so order does
  not matter.

### Held: device regression

The adapted patches are in `patches/security/`, and their series entries are
commented out. A device test on 2026-10-08 (Galaxy M31, Mali GPU) of the full
build carrying all nine found that an accelerated 2D canvas which has drawn
text or an anti-aliased path reads back `51,170,122,253` instead of
`51,170,119,255` for an opaque `#3a7` fill with `getImageData`. This hits every
pixel, not only those near the drawing. Plain `fillRect`s and CPU canvases
(`willReadFrequently`) read back correctly, and the 0.4.3 nightly is correct in
every case. Text and paths go through the vertex-buffer upload path and plain
rects do not, which points first at #209 and then at the two read/write-back
fixes. The build host was removed the same day, so this was not bisected. Each
of the three comes back only after a rebuild with it reverted clears the repro
(`canvas-readback-repro.html`, kept beside that build's APK).

| Issue | CVE | Severity | Component | Upstream | Backport |
|---|---|---|---|---|---|
| #209 | CVE-2026-93387 | High | Improper state validation in Skia (silent `updateData` failure) | skia `f593e0fb8833` on chrome/m152 | verbatim except `GrDrawingManager` |
| #246 | CVE-2026-95293 | High | Uninitialized resource in GPU (`GLTextureHolder` upload/readback errors) | src `dee0423444e3` on refs/branch-heads/7977 (M152) | logic verbatim |
| #354 | CVE-2026-102319 | High | Uninitialized resource in GPU (RasterDecoder marks SharedImage cleared after a failed upload) | src `ef6ecad70a04` on refs/branch-heads/7977 (M152) | adapted; refactor hunks and tests omitted |

- **#209** returns `GrRenderTask::ExecutionResult` upstream, a type that came
  in with CVE-2026-91740 (deferred below). M140's `executeRenderTasks()` still
  returns "any task executed", so a failed upload resets the flush state and
  returns false. No task runs, so nothing draws from the stale buffers, which
  is the fix.
- **#354** checks for GL errors in M140's existing `!IsCleared()` branch.
  Upstream also has a `success` flag there, from a later change. Upstream's
  swap of three drain loops for the new `DrainGLErrors()` changes no behaviour
  and is left out, so this patch shares no context with #246.

### Deferred

| Issue | CVE | Severity | Component | Reason |
|---|---|---|---|---|
| #174, #169, #358 | CVE-2026-91740, CVE-2026-91733, CVE-2026-102325 | High | Skia Ganesh flush-failure handling | One upstream series: skia `559b075da530` (render-task execution results) and `e8efd15f214a` (`GrDirectContext::FlushResult`, a public API change), then src `648496c1b066` (Chromium's callers). It stands on four July 2026 Ganesh changes M140 does not have (`731888c3`, `04dbd152`, `30d6603d`, `cee83412`). Taking it means porting six Skia changes plus Chromium's side, so it is a project of its own, not a backport. |
| #375 | CVE-2026-103629 | High | Integer overflow in Skia (32-bit `SkPath` generation ID) | skia `76f502419c33` widens the ID to 64 bits in `SkPathData`, which M140 does not have. The Chromium half (`395f22b533ce`) widens the paint-cache IPC and regenerates command buffer code. Two ports, both needing a full rebuild. |
| #232 | CVE-2026-95322 | Critical | Out of bounds write in GPU (AHardwareBuffer NV12 readback) | The vulnerable code path (`AHardwareBuffer_lockPlanes` NV12 readback in `ahardwarebuffer_image_backing_factory.cc`) does not exist in M140. It was introduced in a later milestone, so this CVE is not applicable to Cobalt's Chromium 140 base. |

### Does not apply to Cobalt

| Issue | CVE | Why |
|---|---|---|
| #341 | CVE-2026-102303 | The fix (src `931dace617e9`, M153) is in `GLES2DecoderImpl`'s lazy clear and `framebuffer_manager.cc`, which are the validating decoder. Cobalt always runs the passthrough decoder (see #1, #65 and #147). |

## 0.4.3 (September 2026)

28 fixes, from the watcher's second batch (#26 to #35, the Chrome 153 release
of 8 September) and then its third (#36 to #150, the same release's Medium
fixes and the High ones it listed later).

Most fixes were not on a release branch at all. They landed on main before
Chrome 153 branched and shipped by being there, so a scan of the release
branches found 45 of the 122. A second scan, 55,000 commits back through
`branch-heads/8010` and through ANGLE's and V8's main, found fixes for all but 4
of the rest, matched by the `Bug:` or `Fixed:` line.

What was taken was chosen for being reachable on Android and cheap to build:
no `.mojom` change, no Blink runtime flag, nothing that ports a feature. Every
patch applies to M140 exactly (no fuzz), and the series was run twice to show
that no two backports share diff context.

### Taken

| Issue | CVE | Severity | Component | Upstream | Backport |
|---|---|---|---|---|---|
| #27 | CVE-2026-87440 | High | Out of bounds read in Media (`IntervalMap`) | src `f3c09a109718` (main, M153) | verbatim |
| #28 | CVE-2026-87444 | High | Memory corruption in Codecs (libyuv NEON) | libyuv `8d7ed8af356a`, rolled in by src `398717fe2d57` (M153) | the arm64 clobber fix only |
| #34 | CVE-2026-87492 | High | Incorrect authorization in DevTools | src `e8900489be45` (M152) | adapted |
| #49 | CVE-2026-87564 | High | Type confusion in V8 (`Array.prototype.flat`) | v8 `6a7e263a0970` (15.3) | verbatim |
| #59 | CVE-2026-87612 | High | Type confusion in V8 (snapshot) | v8 `1a42286e205b` (15.3) | verbatim |
| #55 | CVE-2026-87587 | High | Use after free in V8 (Maglev for-in) | v8 `5febc6b5195c` (15.3) | verbatim, moved |
| #65 | CVE-2026-87650 | High | Out of bounds read in WebGL (ANGLE GL) | angle `0e80263dfe75` (main) | verbatim; ANGLE half only |
| #64 | CVE-2026-87647 | High | Uninitialized resource in GPU (SharedImage) | src `7def51f0b56c` (M153) | verbatim |
| #62 | CVE-2026-87639 | High | Use after free in WebPackaging | src `e70631df78a1` (main) | verbatim |
| #36 | CVE-2026-87499 | High | Incorrect authorization in Network | src `41268bddbfa9` (M153) | adapted (includes) |
| #85 | CVE-2026-87452 | Medium | Incorrect authorization in GPU (passthrough) | src `fc0834fae837` (main) | verbatim |
| #86 | CVE-2026-87453 | Medium | Confused deputy in BackgroundFetch | src `8046f4cc56ed` (main) | verbatim |
| #89 | CVE-2026-87456 | Medium | Uninitialized resource in Media (WebCodecs) | src `a2581c3f7c0e` (main) | adapted |
| #100 | CVE-2026-87476 | Medium | Incorrect authorization in Loader | src `0c9003632d87` (main) | verbatim |
| #102 | CVE-2026-87479 | Medium | Insufficient policy enforcement (DownloadURL IPC) | src `f66e7ae1e551` (main) | adapted |
| #84 | CVE-2026-87451 | Medium | Information leak in Downloads | src `3558bf2bb5f1` (main) | adapted |
| #111 | CVE-2026-87493 | Medium | Missing authorization in FileSystem (FSA move) | src `2f959d402954` (main) | verbatim |
| #113 | CVE-2026-87495 | Medium | Information leak in Scroll | src `708c49844359` (main) | adapted |
| #115 | CVE-2026-87497 | Medium | Uninitialized resource in Codecs (JPEG) | src `154727b1485f` (main) | adapted |
| #117 | CVE-2026-87503 | Medium | Inappropriate implementation in Downloads | src `79649f9623c5` (main) | verbatim |
| #119 | CVE-2026-87505 | Medium | Incorrect authorization in FileSystem | src `42d042d4eacd` (main) | adapted |
| #120 | CVE-2026-87506 | Medium | Privilege elevation in WebUI | src `d074777b1332` (main) | verbatim |
| #122 | CVE-2026-87508 | Medium | Incorrect authorization in Loader (workers) | src `03b026a35f12` (main) | verbatim |
| #126 | CVE-2026-87516 | Medium | Observable discrepancy in Navigation | src `62029e959eeb` (main) | verbatim |
| #135 | CVE-2026-87533 | Medium | Use after free in DevTools | src `d5186c8dddab` (main) | verbatim |
| #144 | CVE-2026-87549 | Medium | Incomplete cleanup in Downloads | src `64a4680b5157` (main) | verbatim |
| #145 | CVE-2026-87550 | Medium | Improper escaping in CSS (Typed OM) | src `9ef64552f77b` (main) | verbatim |
| #149 | CVE-2026-87557 | Medium | Missing authorization (worker final URL) | src `0db5072577f3` (main) | adapted |

"main" means the fix landed before Chrome 153 branched. Tests are left out of
this batch: every one of these test files has drifted since M140, and carrying
them would have meant porting test fixtures rather than fixes.

- **#65** is fixed twice upstream: in ANGLE's GL backend and in the validating
  GPU decoder (`texture_manager.cc`). Only the ANGLE half is taken, for the
  same reason as #1 and #6 in 0.4.1: Cobalt always runs the passthrough
  decoder.
- **#117** is Android-specific upstream: a download resumed in minimal-browser
  mode could be renamed to its final path before its content check ran.
- **#102** adds a bad-message reason. It keeps upstream's value, 374, rather
  than M140's next free one, so a crash report means the same thing in both.
- **#84** moves M140's initiator lookup above `ComputeDownloadPolicy()`, which
  now needs it. Upstream has it in that order too.

### Does not apply to Cobalt

| Issues | Why |
|---|---|
| #31, #90 (updater), #42 (Windows caret), #43, #47 (Chromoting, Windows), #87 (CloudAP, Windows), #112 (Windows), #132 (GCPW) | Windows only. |
| #38, #60, #67 (ANGLE D3D11), #96 (Dawn Metal), #58 (macOS gamepad) | A GPU backend or platform Android does not have. |
| #40, #104, #127 | iOS. |
| #32, #48, #53 (one desktop `Browser` fullscreen fix), #33 (print preview), #35 (read anything), #39 (Ozone drag and drop), #92 (FedCM desktop dialog), #114, #121, #129 (desktop bubbles) | Desktop UI; not compiled for Android. |
| #147, and #65's second half | The validating GPU decoder, which Cobalt never runs. |
| #46, #108 | Trusted Web Activities. Cobalt has no TWA or Custom Tabs entry point. |
| #68 | Chrome's Android context menu, which Cobalt's shell does not use. |
| #125 | Only reachable with a file-access delegate installed, which only ChromeOS data-loss prevention does. |
| #66 | Amends HTML-in-Canvas filter code M140 does not have. |
| #103 | Only matters with WebView's `allow_universal_access_from_file_urls` or `loadDataWithBaseURL`; Cobalt uses neither. |

### Deferred, with the reason

| Issues | Why not now |
|---|---|
| #26, #107, #138, #146 | A `.mojom` change. Most of Chromium includes these, so each turns an incremental build into ~26,000 steps. They land with the next change that needs a full rebuild anyway. #26's patch is already written. |
| #19, #44, #110 (V8) | The upstream change does not fit V8 14.0's code; each is a hand port of compiler code with nothing upstream to check it against. #19 may be a rebase trigger (see 0.4.1). |
| #37, #56, #57 (ANGLE translator) | Large conflicts with M140's translator. |
| #29 | Fixed upstream only by a refactor (`c0edd277235be`); the minimal fix was abandoned. |
| #94 | Turns on a restriction M140 does not implement. Taking it means porting the feature. |
| #93 | Would share diff context with #102, which the series cannot tell apart from a failed apply. It lands folded into #102's file, or after it. |
| #133 | Amends response-censoring code M140 does not have yet. |
| #30 and 3 others | No public fix found. |
| The rest of #36 to #150 | Fix found and not yet read. Next release. |

## 0.4.2 (September 2026)

Seven of the eight 0.4.1 deferrals: the three PowerVR fixes that 0.4.1 put
first in line, and the File System Access, GPU client, Omnibox and DevTools
fixes. All seven are from the same two Chrome releases, found the same way, and
none needed a rebase. Only #19 is left, the one that might.

| Issue | CVE | Severity | Component | Upstream | Backport |
|---|---|---|---|---|---|
| #4 | CVE-2026-87464 | Critical | Use after free in WebGL (ANGLE GL, PowerVR Android) | angle `82f9e7ef4530` (M153), with its prerequisite `a17d5224d83f` (M151) | adapted, prerequisite included |
| #23 | CVE-2026-85050 | High | Out of bounds write in WebGL (ANGLE GL, PowerVR) | angle `7df613367a1d` (M152) | verbatim logic |
| #3 | CVE-2026-87438 | Critical | Out of bounds write in WebGL (ANGLE GL, PowerVR) | angle `fca5efdfeff5` (M153) | adapted |
| #14 | CVE-2026-84354 | High | Incorrect authorization in FileSystem | src `a4775d86854c` + `da77132fcf66` (main, Chrome 152) | adapted, both changes |
| #13 | CVE-2026-84351 | High | Out of bounds write in GPU (GLES2 client) | src `4bb7d52b63ad` (M152) | regenerated from the generator |
| #15 | CVE-2026-84357 | High | Omnibox (document provider) | src `a9d6126968cd` (M152) | verbatim code |
| #17 | CVE-2026-85042 | High | Use after free in DevTools | src `4dfbc081b5e3` (M152) | adapted |

- **#4** extends a workaround, `reattachTextureToFboAfterLayerIncrease`, that
  M140's ANGLE does not have. So the M151 change that introduced it (itself a
  fix, bug 541748549) is carried in the same patch. It touches ANGLE's core
  `Framebuffer` and `Texture` observers, with the new code behind a PowerVR
  Android feature.
- **#3** has a companion change in `gpu/command_buffer/service`, which patches
  the validating decoder; Cobalt never runs that decoder (see #1), so only the
  ANGLE half is taken. Upstream reverted it on M152 for lack of bake time and
  kept it on M153. In Cobalt it is limited to PowerVR, the same as upstream.
- **#14** is the rename check (`FileSystemHandle.move()` now needs write access
  to the parent directory) and the change that enabled it by default, carried
  together with the defaults Chrome shipped. File System Access on local files
  is stable on Android in M140, so any page reaches it.
- **#13** is 40 KB, most of it generated. It was not hand-edited. The
  generator changes were applied, and M140's own generator was run; before
  the change it reproduces M140's checked-in files byte for byte, so its
  output differs only by the fix. The fix clamps every `glGet*` result to
  `GLGetNumValuesReturned(pname)`, so a pname missing from that table would
  come back empty and break WebGL. So every pname M140's clamped getters
  accept was checked against the table, by enum value: M140 leaves out exactly
  the pnames upstream leaves out, which the client answers from its caches
  and which never reach the clamp.
- **#15** is in Chrome's omnibox document provider, which is compiled into
  Cobalt but not driven by Cobalt's address bar. It is taken for completeness.
- **#17** is reachable only with remote debugging on, or through an extension
  using `chrome.debugger`.
- Each ANGLE patch declares its feature at its own place in ANGLE's feature
  lists. The first draft appended each one after the last, and the series'
  "already applied" check, a reverse apply, then failed for every patch whose
  context a later one had changed, 0.4.1's CVE-2026-87488 included. No two
  backports may share diff context; see `tools/patches/series.txt`.

The ANGLE fixes only run on ANGLE's GL backend on PowerVR GPUs. They were
built and smoke-tested on a Mali phone (Galaxy M31), where ANGLE runs on Vulkan,
so that test shows they break nothing there. It does not exercise them. They
have not been run on PowerVR hardware.

The #13 clamp was checked on the same phone with WebGL Report's WebGL 2 page,
which reads every limit through `glGet*`. All of them came back, including the
two-value ones (line width range, viewport dimensions). The WebGL Aquarium ran
at 34 fps.

## 0.4.1 (September 2026)

Source: the 25 issues filed by the security watcher for **Chrome
152.0.7977.75** (1 September) and **153.0.8010.36** (8 September). Every fix
was located on the M152 or M153 release branch by its bug id, read, and checked
against what Cobalt compiles and runs before anything was taken.

### Taken

| Issue | CVE | Severity | Component | Upstream | Backport |
|---|---|---|---|---|---|
| #5 | CVE-2026-87488 | Critical | Use after free in WebGL (ANGLE GL, Mali) | angle `99f12d73e026` (M153) | adapted |
| #7 | CVE-2026-87628 | Critical | Use after free in Cast (Open Screen) | src `6cfbc1f240eb` (M152) | verbatim |
| #8 | CVE-2026-84324 | High | Use after free in Proxy | src `a87859ad7a79` (M152) | adapted |
| #10 | CVE-2026-84326 | High | Uninitialized resource in V8 (Turboshaft) | v8 `32f541982747` (15.2) | adapted, test included |
| #11 | CVE-2026-84333 | High | Use after free in Dawn (Vulkan, PowerVR) | dawn `4178cb7e771b` (M152) | adapted |
| #18 | CVE-2026-85043 | High | Incomplete cleanup in Network (auth) | src `71067ec79dbb` (M152) | adapted |
| #21 | CVE-2026-85048 | High | Use after free in Compositing (viz) | src `1e944c7f193a` (M152) | adapted |
| #22 | CVE-2026-85049 | High | Use after free in Skia | skia `0873ec164a06` (M152) | verbatim |
| #24 | CVE-2026-85051 | High | Type confusion in Compositing (cc) | src `29a203e805e4` (M152) | adapted |
| #25 | CVE-2026-85052 | High | Out of bounds read in CrashReporting | src `4d2a8274815f` (M152) | verbatim |

"Adapted" means the surrounding code differs in M140 and the change was
re-expressed against it. The logic is unchanged in every case, and each patch
file says exactly what differed. Tests are carried where M140's test files
allow.

### Does not apply to Cobalt

Checked against the build, not assumed.

| Issue | CVE | Why |
|---|---|---|
| #1 | CVE-2026-84352 (Critical, WebGL) | The fix is in the **validating** GLES2 decoder. Cobalt sets `is_desktop_android`, which sets `enable_validating_command_decoder = false` (`ui/gl/features.gni`): it always uses the passthrough decoder (ANGLE), so this code never runs. |
| #6 | CVE-2026-87527 (Critical, WebGL) | Same: validating decoder only (`texture_manager.cc`). |
| #2 | CVE-2026-84353 (Critical, Shared Tab Groups) | Tab-group sync UI; not built for Cobalt, which has no sync. No fix commit is visible on the M152/M153 branches either. |
| #9 | CVE-2026-84325 (DataTransfer) | ChromeOS only (ARC VM paths). |
| #12 | CVE-2026-84349 (Browser) | Desktop `BrowserView`; not built on Android. |
| #16 | CVE-2026-84359 (Skia) | Skia Graphite, which this build does not compile (0 objects). |
| #20 | CVE-2026-85046 (V8 type confusion) | The bug is in *inlined* `Array.prototype.sort`. V8 14.0 does not inline sort in either compiler, so the vulnerable code does not exist in M140. |

### Deferred, with the reason

| Issue | CVE | Why not now |
|---|---|---|
| #19 | CVE-2026-85045 (High, V8 Maglev) | **Possible rebase trigger 1.** V8 14.0 represents HeapNumber virtual objects differently: a raw number materialised as one shared literal, which is very likely the same bug. Upstream's fix is written against a representation 14.0 does not have, so carrying it means writing new deoptimisation code with nothing upstream to check it against. 0016: a subtly wrong security fix is worse than a missing one, because it reports as fixed. Needs V8-specialist review before it is either written or declared a rebase trigger. |

PowerVR matters more than it looks: budget MediaTek phones (Helio G35/G37 and
similar) ship PowerVR GPUs, which is exactly Cobalt's 4 GB target class. #3, #4
and #23 were taken in 0.4.2, with #13, #14, #15 and #17; see above. Only #19
is still deferred: the one that may force a rebase.
