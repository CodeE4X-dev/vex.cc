# TODO — Ronde 2 (audit Claimer / giver / server / stock)

Diinvestigasi terhadap `files/Claimer.lua`, `files/giver.lua`, `server.js`, `files/Stock*.lua`.
Dua bug yang dilaporkan sudah di-root-cause di bawah (akarnya SAMA). Plus 100+ item fix/improve/add/remove/test.

Approve per-poin (mis. "C-1, C-2, CL-3..CL-9 ok") lalu aku implement (TANPA comment di code).

---

## ATURAN WAJIB (berlaku untuk semua implementasi)

- **A-1** JANGAN PERNAH pakai `PlayerRemoving` listener untuk mendeteksi apakah **DIRI SENDIRI** (LocalPlayer) sudah leave/keluar server. Untuk tahu status diri sendiri, pakai **server heartbeat** / **bridge** / state internal — bukan listener leave atas diri sendiri.
- **A-2** `PlayerRemoving` untuk mendeteksi **player LAIN** (victim, claimer, target) leave → BOLEH. Semua pemakaian sekarang (`giver.lua:1051`, `giver.lua:1154`, `Claimer.lua:1640`) memang mendeteksi player lain → tetap dipertahankan, hanya diperhalus (lihat C-6/C-7).
- **A-3** TANPA comment di semua code yang aku tulis/ubah.
- **A-4** Untuk verifikasi data exact saat testing, pakai **Bridge** (lihat section BRIDGE di bawah) — jangan menebak; jalankan Lua live & baca arg/response asli.
- **A-5** Tema GUI: **dark black-grey** (lihat section GUI).

---

## ROOT-CAUSE DUA BUG YANG DILAPORKAN

### Bug A — embed "client-release: trade never started within 120s" padahal trade SUKSES
- Path sukses saat victim masih di server (`Claimer.lua:1583-1602`) set `startTime = 0` + `lastSuccessAt = os.time()` tapi `hitId` tetap dipegang dan **`claimedAt` tidak di-reset**.
- Watchdog `Claimer.lua:443` nyala kalau `claimedAt > 0 and startTime == 0` — TIDAK mengecualikan `lastSuccessAt > 0`. Jadi deferred-success kelihatan identik dengan never-started, dan begitu `os.time() - claimedAt > 120` → `releaseStuckClaim("trade never started")` → kirim `failed`.

### Bug B — "🔄 Claimer X left server; listener stays armed for rejoin" padahal claimer nggak beneran leave
- Efek domino dari Bug A: setelah `failed` palsu, claimer abandon + ServerHop keluar, jadi `Players.PlayerRemoving` giver (`giver.lua:1157`) nge-log "left". Claimer keluar HANYA karena Bug A membunuh trade yang bagus. (Catatan: ini deteksi player LAIN → sesuai A-2, listener-nya tetap, cuma log-nya yang diperbaiki.)

---

## FIX KRITIS (kerjakan duluan)

- **C-1** `Claimer.lua:443` — tambah `and (currentTradeData.lastSuccessAt or 0) == 0` ke guard never-started biar deferred-success nggak pernah dianggap never-started. (Fix utama Bug A.)
- **C-2** `Claimer.lua:1583` path deferred-success — reset juga `currentTradeData.claimedAt = 0` (atau refresh ke `os.time()`) supaya window 120s nggak bisa nyala ke chain yang sudah sukses.
- **C-3** `Claimer.lua:402 releaseStuckClaim` — early-return kalau `(currentTradeData.lastSuccessAt or 0) > 0`; jangan pernah kirim `failed` setelah ada sukses. Log + finalise jadi success.
- **C-4** `releaseStuckClaim` — sebelum kirim `failed`, baca status hit dari server (`GET /api/hits/:id`, lihat C-8) dan skip kalau sudah `success`. Cegah racing dengan success sisi server.
- **C-5** Watchdog loop (`Claimer.lua:418`) — kalau `lastSuccessAt > 0`, pindah dari logika "never started" ke logika "deferred chain" (tunggu victim leave / followup window), jangan ke `releaseStuckClaim`.
- **C-6** `giver.lua:1157` — pisahkan log: kalau claimer leave SETELAH ada trade confirmed sama kita, log "✅ Claimer X selesai & keluar" (success), bukan "left; armed for rejoin". Pakai flag per-claimer `tradedOnce`. (PlayerRemoving tetap — ini player lain, A-2.)
- **C-7** `giver.lua` PlayerRemoving claimer — debounce: tunggu 2–3s lalu re-cek `Players:FindFirstChild(name)` sebelum nyatain "left" (nutup flap respawn/relog). (Tetap deteksi player lain, A-2.)
- **C-8** Tambah endpoint server `GET /api/hits/:id` (single hit) buat C-4; sekarang cuma ada `/api/hits/queue`.
- **C-9** `Claimer.lua` — saat kirim status final apapun, sertakan `itemsTransferred`/`rapTransferred` yang diketahui lokal biar success telat nggak kecatat 0/0.
- **C-10** `server.js:5015 /status` — kalau `failed` masuk tapi hit sudah punya `itemsTransferred > 0`, log warning & pilih record yang lebih kaya (defensif terhadap sisa Bug A).
- **C-11** (A-1) Audit: pastikan TIDAK ADA deteksi "diri sendiri leave" lewat PlayerRemoving di manapun. Status diri sendiri murni dari heartbeat/server. (Sekarang sudah clean — jadikan aturan permanen + komentar di TODO, bukan di code.)

---

## CLAIMER.LUA — STATE MACHINE TRADE

- **CL-1** Reset `claimedAt = 0` di semua jalur terminal (audit path deferred + followup).
- **CL-2** Ganti 3 blok `currentTradeData = { ... }` yang hampir sama (`~518`, `~564`) dengan satu helper `resetTradeData(keepSuccess)` biar nggak drift.
- **CL-3** Satu sumber kebenaran "trade aktif": satu `isTradeActive()` daripada campur `targetPlayer ~= ""`, `hitId ~= nil`, `startTime ~= 0`, `confirmed`.
- **CL-4** Cancel `task.delay(FOLLOWUP_WINDOW_SEC + 30)` (`1596`) saat hit finalise lebih awal, biar nggak nyala basi ke `hitId` yang sudah didaur ulang.
- **CL-5** Guard closure followup-delay dengan snapshot `savedHit` DAN `lastSuccessAt` biar nggak nge-act ke claim baru yang reuse state.
- **CL-6** Confirm watchdog (`1626`) bisa nge-fire `recordTradeResult("success")` walau nggak ada item pindah — wajib `itemsTransferred > 0` ATAU `confirmed` sebelum klaim success, kalau nggak tandai `failed`/`unknown`.
- **CL-7** `tradeItemsCount`/`tradeRAPValue` module-level dipakai ulang antar-ronde; snapshot per-confirm biar nggak dobel/stale.
- **CL-8** Akumulasi RAP (`1702-1703`) nganggap tiap `Confirmed` batch baru — dedupe pakai trade-session id biar `Confirmed` yang ke-fire ulang nggak dobel total.
- **CL-9** Heuristik "should record" (`496-511`) bisa nandai success asli jadi nggak kecatat ("random-or-cancelled") kalau `expectedTarget` keburu di-clear — catat items/RAP terlepas dari keputusan success-rate.
- **CL-10** `postHitStatus` nggak ada retry; bungkus 2–3 attempt + backoff biar status final yang drop nggak hilang (yang bikin server stale→timeout).
- **CL-11** `postHitStatus` harus anggap HTTP 409 `already-finalised`/`cannot-downgrade-success` sebagai terminal-OK (berhenti retry), bukan failure.
- **CL-12** Idempotency: sertakan `clientFinalId` (hitId + result) biar post final dobel jadi no-op di server.
- **CL-13** Handler `TeleportInitFailed` (`460`) manggil `releaseStuckClaim` tanpa syarat — guard dengan `lastSuccessAt == 0` juga.
- **CL-14** Wrong-server release (`447`, `TELEPORT_CONFIRM_WINDOW = 8s`) terlalu agresif; naikkan ~15s atau verifikasi teleport benar-benar antri sebelum release.
- **CL-15** `loadHitState()` dibaca 2x per tick (`441-442`); baca sekali, cache.
- **CL-16** Watchdog `task.wait(10)` → deteksi teleport-fail telat 10s; pecah jadi cek cepat (2s) untuk teleport-confirm dan lambat (10s) untuk heartbeat.
- **CL-17** `transfer-items` 90 item × `PER_ADD_DELAY` blocking ~45s+30s; jalankan via `task.spawn` biar coroutine heartbeat nggak kelaparan (cegah claimer ditandai stale).
- **CL-18** Sama untuk `transfer-tokens` confirm-loop 30s — pastikan heartbeat tetap jalan.
- **CL-19** Auto-accept `ReceivedTradeRequest` harus re-validasi giver terhadap expected victim hit yang aktif, bukan terima giver siapapun saat hit dipegang.
- **CL-20** Plafon keras umur 1 hit (mis. 8 menit) → finalise `timeout` lokal & bebaskan claimer, jangan cuma andalkan stale recovery server.
- **CL-21** Log HTTP code + body `postHitStatus` ke GUI/console untuk tiap post final (visibilitas testing).
- **CL-22** Saat `recordTradeResult` mutusin `failed`, sertakan heartbeat code + age terakhir biar message embed diagnostik, bukan cuma "trade never started".
- **CL-23** Reset `confirmedTriggered`/`confirmed` saat claim baru biar `confirmed=true` basi nggak nge-short-circuit auto-ready victim berikutnya.
- **CL-24** `isTradeUiActive()` dipoll di 3 loop; konsolidasi jadi satu state berbasis signal.

## CLAIMER.LUA — CLAIM / IDLE PICKUP

- **CL-25** Deferred-success pegang `hitId` sampai `FOLLOWUP_WINDOW_SEC+30` (=330s) → claimer report `busy` & nggak ambil hit baru. Pendekkan followup ~60–90s, ATAU izinkan claim hit baru saat followup pending (decouple "busy" dari "followup"). **Ini win utama "proses instan saat idle".**
- **CL-26** Status heartbeat (`~1762`) report `busy` tiap `hitId` keisi — report `followup` terpisah biar dashboard tahu claimer sebenarnya agak bebas.
- **CL-27** Thundering herd `fireClaim`: tiap waiter long-poll kebangun & semua POST `/api/hits/claim` barengan; cuma satu menang (lock 15s), sisanya buang request. Tambah jitter atau assignment per-claimer sisi server (SV-9).
- **CL-28** Error path long-poll `task.wait(2)` lalu `task.wait(0.2)` — gabung jadi satu backoff.
- **CL-29** `requestHitClaim` nggak pakai preferredVictim; bisa kirim victim terakhir buat bias ke retrade.
- **CL-30** Backoff eksponensial + cap di reconnect `connectWebSocket` (`1241`) — sekarang `task.wait(1)` rekursif ketat bisa spin kalau gateway nolak.
- **CL-31** Rekursi `connectWebSocket` numpuk call stack tiap reconnect — ubah jadi loop `while true`.
- **CL-32** Heartbeat loop Discord di dalam `OnMessage` (`1223`) nggak pernah exit saat close — bocor 1 coroutine per reconnect. Ikat ke token/generasi yang di-invalidasi `OnClose`.
- **CL-33** Path `is_channel_supported` + `Action(message.content)`: validasi/whitelist command; message Discord rusak jangan bikin handler crash (bungkus pcall).
- **CL-34** Timer fallback (20s) + long-poll dua-duanya manggil `fireClaim`; pastikan `claimWakeup` nggak dobel-proses dalam tick yang sama.

## CLAIMER.LUA — CLEANUP / REMOVE

- **CL-35** Strip semua comment di `Claimer.lua` (A-3).
- **CL-36** Hapus referensi mati `TradeInProgress` yang disebut di comment `1672` (konfirmasi nggak ada sisa).
- **CL-37** Hapus `print("[SuccessRate] ...")` atau rutekan lewat `notifyGUI`/logger konsisten.
- **CL-38** De-dupe dua blok `currentTradeData = {...}` (lihat CL-2).
- **CL-39** Satukan `http_request or (syn and syn.request) or request` ke `getRequestFunc()` di semua tempat.

---

## GIVER.LUA

- **GV-1** (C-6/C-7) Bedakan "claimer selesai+keluar" vs "claimer keluar mid-trade" di log PlayerRemoving.
- **GV-2** Track `tradedItemsCount` per-claimer biar log bisa sebut berapa yang diambil sebelum keluar.
- **GV-3** `startedClaimers[name] = nil` saat leave menghapus progress; persist record "selesai dengan X" biar rejoin nggak restart claimer yang sudah kelar.
- **GV-4** Loop victim heartbeat (`925-944`) cuma break di 409/404 — break juga saat trade giver benar-benar kelar biar loop bebas.
- **GV-5** Heartbeat `task.wait(10)` vs timeout leave server 40s cuma ~3 beat slack; turunkan ke 7–8s buat margin saat lag.
- **GV-6** Coroutine heartbeat bisa kelaparan oleh loop trade panjang giver — pastikan jalan di `task.spawn` sendiri, lepas dari wait trade.
- **GV-7** `startTradeOnce(p, 99)` delay awal 10s (`PlayerAdded`) bisa lewat asumsi teleport-confirm claimer; selaraskan dengan `TELEPORT_CONFIRM_WINDOW` Claimer.
- **GV-8** Re-arm-on-rejoin bisa loop selamanya kalau claimer crash berulang; cap percobaan re-arm per claimer.
- **GV-9** Validasi nama claimer terhadap claimer hit aktif (`/api/hits/:id`) biar nama whitelist random yang leave nggak nge-emit log "left" menyesatkan.
- **GV-10** Log webhook saat victim heartbeat pertama balik 409 (server sudah finalise) biar timeline giver+claimer sinkron saat testing.
- **GV-11** Log hit `id` di tiap baris log giver biar 1 hit bisa ditrace lintas giver↔claimer↔server.
- **GV-12** Strip semua comment di `giver.lua` (A-3).
- **GV-13** Kirim `itemsRemaining` sisi giver bareng heartbeat biar server/claimer bisa mutusin retrade vs finalise pintar (ganti window tetap).
- **GV-14** Saat success final, proaktif POST "victim-finished" ke server biar path deferred claimer langsung kelar (matiin tunggu 330s — terkait CL-25/SV-2). **Win idle.**
- **GV-15** Guard loop re-trade `isInventoryStillGood` biar nggak trade item locked yang sama berulang (skip TradeLock/Listing).

---

## SERVER.JS — HIT QUEUE

- **SV-1** Tambah `GET /api/hits/:id` (single hit) — dibutuhkan C-4/GV-9.
- **SV-2** Tambah `POST /api/hits/:id/victim-finished` (atau flag) biar giver sinyal "inventory abis/selesai" → server finalise + bangunin claimer instan (terkait GV-14/CL-25). **Win idle.**
- **SV-3** `/status` (`5015`) — saat downgrade ditolak (409), kembalikan record yang ada (sertakan `itemsTransferred`) biar client bisa self-heal.
- **SV-4** Catat event log terstruktur per transisi (id, from, to, claimer, ts, message) ke ring buffer + `GET /api/hits/events` buat timeline testing.
- **SV-5** Threshold stale-recovery (5min) & victim-leave (40s) sudah env-overridable — tampilkan di `/api/hits/stats` biar kelihatan.
- **SV-6** Saat hit recover dari stale (`4303`), increment metric + log claimer mana yang nge-drop (bantu spot pola Bug A: success-then-stale).
- **SV-7** `notifyHitsWaiters` clear SEMUA waiter tiap bump — tambah id hit yang berubah di response wait biar claimer bisa skip wake yang nggak relevan.
- **SV-8** WebSocket asli opsional (`ws` pkg) `/ws/hits` push `{type:'hit', id}` ke claimer idle; long-poll tetap fallback. (Kamu minta — tapi long-poll sudah ~0.2s; WS terutama mangkas overhead request + enable assignment terarah.)
- **SV-9** Assignment sisi server: daripada N claimer rebutan `/claim`, server pilih claimer idle (by heartbeat) & push hit ke dia → matiin thundering herd (CL-27).
- **SV-10** Track presence claimer dari `/api/claimers/heartbeat` & expose list `idle` buat SV-9.
- **SV-11** Guard `/claim` biar victim yang sama nggak dikasih ke 2 claimer di server beda (dedupe victim+jobId).
- **SV-12** `HITS_CLAIM_LOCK_MS = 15s` lock global nge-serialize SEMUA claim — bikin optimistic (compare-and-set di kandidat) biar claim victim berbeda nggak saling blok.
- **SV-13** Tambah `claimedByJobId` biar heartbeat/status dari instance server salah ditolak (cegah clobber lintas-server).
- **SV-14** Validasi `itemsTransferred`/`rapTransferred` non-negatif; clamp.
- **SV-15** Gagal `recordHitToStats` cuma `console.error` — tambah retry/queue biar stats nggak hilang diam-diam.
- **SV-16** Embed builder: sertakan hit `id` (pendek) + claimer di title/footer buat traceability.
- **SV-17** Persist `hitsVersion` biar restart server nggak reset versi long-poll ke 0 & salah sinyal "change" ke semua claimer.
- **SV-18** `/api/hits/wait` max 30s; selaraskan dengan claimer 25s + 0.2s — jaga timeout claimer < timeout server.

## SERVER.JS — DISCORD EMBED

- **SV-19** "❌ FAILED" dengan message `client-release: trade never started` di-suppress/downgrade kalau ada item transferred (defensif Bug A; terkait C-10).
- **SV-20** Tampilkan `itemsTransferred`/`rapTransferred` di embed success.
- **SV-21** Tampilkan staleRecoveries (`4393`) konsisten di semua state.
- **SV-22** Warna/title `leave` vs `failed` vs `timeout` harus beda jelas (verifikasi color mapping).
- **SV-23** Truncate item lines di `\n` (sudah fix ronde 1 — verifikasi berlaku ke SEMUA embed builder).
- **SV-24** Tambah field "durasi" (claimedAt→finishedAt) buat spot trade lambat.
- **SV-25** Rate-limit edit Discord per hit (debounce processing→success cepat) biar nggak kena 429.

## SERVER.JS — ROBUSTNESS

- **SV-26** Kontensi tulis `withHitsQueue` — pastikan serialize tulis (file DB) biar status+heartbeat barengan nggak ilang update.
- **SV-27** Validasi semua param `:id` (panjang/charset) sebelum scan queue.
- **SV-28** `statusMessage` sudah cap 280 — strip juga control char.
- **SV-29** `recoverStaleHits` jalan di dalam `/claim`; jalankan juga via timer biar stale recover walau nggak ada claimer polling.
- **SV-30** Tambah `/api/hits/admin/force-status` (auth) buat recovery manual dari dashboard.
- **SV-31** Log saat `failed` langsung ngikutin `processing` dari claimer yang sama dalam < beberapa detik (signature Bug A) buat monitoring.

---

## STOCK.LUA / STOCK2 / STOCK3

- **ST-1** Terapkan jaminan pcall-wrapper `executeRedistribute` ke `sendTokensToTarget` (masih manual `TradeInProgress=false` tiap branch → risiko leak saat error tak terduga). Bungkus `table.pack(pcall(...))`.
- **ST-2** Verifikasi `Stock.lua` dapat comment-strip + pcall wrapper sama kayak Stock2/Stock3 (ronde 1 ngerjain Stock.lua; konfirmasi paritas).
- **ST-3** `TradeInProgress` bool polos tanpa timeout — tambah max-age biar trade wedged nggak ngunci stock selamanya.
- **ST-4** Whitelist refresh tiap 90s; cache & diff biar nggak re-log tiap entry tiap siklus (spam log).
- **ST-5** `getTokenCount` `WaitReplion("Inventory")` tiap heartbeat — cache handle replion sekali.
- **ST-6** `collectInventoryStats` iterasi semua item tiap heartbeat (60s) — reuse scan yang sama buat keputusan redistribute.
- **ST-10** `teleportWithRetry` rekursi numpuk stack tiap retry — ubah jadi loop.
- **ST-11** Strip comment Stock.lua (& konfirmasi Stock2/3 full stripped).
- **ST-12** `sendStockHeartbeat` & command-poll share request fn — satukan error handling/backoff.
- **ST-13** Slot-conflict kick (`handleStockRegisterResponse`) — stop juga semua loop spawned, bukan cuma set `AllOperationsStopped`.

---

## GUI — TEMA DARK BLACK-GREY (semua panel)

Palet acuan: bg `#0E0E10`/`#141417`, panel `#1A1A1E`, stroke `#2A2A30`, teks `#E6E6EA`, sub `#9A9AA2`, accent netral grey `#3A3A42`, status hijau/merah/amber redup. Hindari gradient biru terang sekarang.

### Claimer GUI
- **G-1** Re-theme panel Claimer ke dark black-grey (bg near-black, panel abu gelap, stroke tipis, teks off-white).
- **G-2** Status line eksplisit: IDLE / CLAIMED / TELEPORTING / TRADING / CONFIRMING / FOLLOWUP / FINALISING (warna per-state, dark-friendly).
- **G-3** Hasil final terakhir + alasan + items/RAP transferred.
- **G-4** HTTP code `postHitStatus` terakhir (dot hijau/merah) buat health sekilas.
- **G-5** Counter queue pending/processing (sudah dipoll `869`) di panel.
- **G-6** Health koneksi long-poll/WS + waktu wake terakhir.
- **G-7** Panel log expandable (gaya Stock) dengan warna severity di tema dark.
- **G-8** Tombol "Force release current hit" (recovery manual).
- **G-9** Tombol "Re-fire claim now" (manual `fireClaim`).
- **G-10** Countdown ke fallback claim berikutnya + sisa followup-window.
- **G-11** Draggable + minimize + fade-in halus (samain pola Stock GUI yang sudah ada).

### Stock GUI
- **G-12** Re-theme Stock GUI ke dark black-grey (sekarang gradient biru `(85,110,160)` → ganti grey gelap).
- **G-13** Tampilkan stock peer online + token/RAP mereka (`/api/stocks/list`).
- **G-14** Tampilkan command yang lagi dieksekusi + status ack terakhir.
- **G-15** Booth used/free live dari `countMyListings` di status line.
- **G-16** Konsistenkan severity color log ke palet dark.

### giver GUI (kalau ada panel)
- **G-17** Kalau giver punya panel/log overlay, samakan ke tema dark black-grey + tampilkan hit id aktif & status heartbeat victim.

---

## BRIDGE — AMBIL DATA EXACT SAAT TESTING (bukan nebak)

Bridge: `BASE_URL=http://194.13.80.145:8080`, header `x-api-key: <KEY>`. Alur: `GET /api/clients` → `POST /api/exec {clientId, code}` → poll `GET /api/jobs/:id` → baca `logs/ret/error`. Bisa juga `POST /api/screenshot` buat review GUI visual.

- **BR-1** Dump `currentTradeData` live via exec saat sebuah hit jalan: `return HttpService:JSONEncode(currentTradeData)` (expose ke `_G` dulu) — verifikasi `claimedAt/startTime/lastSuccessAt/hitId` di momen watchdog mau nyala (buktiin Bug A).
- **BR-2** Watch `TradeStatus`: pasang listener sementara via exec yang push `p1` + cabang yang diambil ke tabel lalu `return` — lihat arg ASLI, bukan asumsi.
- **BR-3** Tap replion `Set` events (Ready/Confirmed/TradeItems) via exec selama 1 trade, kumpulin ke tabel, `return` — verifikasi `itemsTransferred`/`rapTransferred` akurat (CL-7/CL-8).
- **BR-4** Probe `/api/hits/*` dari sisi game: exec yang `request` ke endpoint hits & `return` body — bandingkan dengan log server (cocokin id).
- **BR-5** Screenshot panel Claimer & Stock SETELAH re-theme dark — `POST /api/screenshot`, buka `BASE_URL+screenshotUrl`, review kontras/overlap/teks (G-1..G-17).
- **BR-6** Ukur latency real: exec yang stamp `os.clock()` di claimedAt/firstTradeReq/firstConfirm/finalised, `return` selisihnya → dipakai retune K-1..K-7.
- **BR-7** Verifikasi aturan A-1: exec yang grep PlayerGui/script aktif buat mastiin nggak ada listener self-leave (sanity check setelah implement).
- **BR-8** Repro Bug B aman: exec yang simulasikan claimer leave (atau cek log giver) + screenshot — konfirmasi log baru "selesai & keluar" muncul, bukan "left armed for rejoin".
- **BR-9** `/api/decompile` (via bridge) kalau perlu konfirmasi signature remote game terbaru sebelum ubah call (mis. TradeStatus/ConfirmTrade arg).
- **BR-10** Pakai `POST /api/exec {cleanup:true}` saat nyuntik script test biar loop/koneksi test sebelumnya nggak ganggu (jangan rejoin kecuali perlu).

---

## INSTRUMENTASI (log args/response, dipasangkan dengan BRIDGE)

- **T-1** Flag `DEBUG` global di Claimer.lua; kalau ON, log tiap remote `:InvokeServer` nama + args.
- **T-2** Log tiap request `/api/hits/*` URL + body + response code + body (sisi claimer) ke ring buffer + GUI.
- **T-3** Log full args `ReceivedTradeRequest` (From, id) tiap accept.
- **T-4** Log arg `TradeStatus` (`p1`) + cabang yang diambil (deferred/finalise/failed).
- **T-5** Log tiap replion `Set` key + siapa (Ready/Confirmed/TradeItems) + value saat DEBUG.
- **T-6** Dump snapshot `currentTradeData` tiap transisi terminal (sebelum reset) buat post-mortem.
- **T-7** server.js: env `LOG_HITS=1` console.log tiap transisi status + diff hit full.
- **T-8** server.js: ring buffer `/api/hits/events` (SV-4) tampil di dashboard buat timeline live.
- **T-9** Correlation id (hit id) di SETIAP baris log giver/claimer/server biar 1 hit bisa di-grep end-to-end.
- **T-10** giver.lua DEBUG: log response code victim-heartbeat tiap tick.
- **T-11** Mode "dry-run" claimer: lakukan semua tapi skip confirm final — validasi state machine tanpa mindahin item.
- **T-12** Timing log (claimedAt, firstTradeReq, firstConfirm, finalised) buat ukur latency real vs 120s/40s/330s → retune.
- **T-13** Log saat watchdog SEHARUSNYA release tapi di-suppress guard `lastSuccessAt` baru (buktiin fix Bug A nangkep).
- **T-14** server.js: hitung + expose berapa `failed` bawa message `client-release: trade never started` per hari (metric regresi Bug A).
- **T-15** `/api/hits/selftest`: jalanin fake hit pending→processing→success buat validasi pipeline tanpa victim asli.

---

## KONSTANTA YANG DIRETUNE (setelah data T-12/BR-6)

- **K-1** `TRADE_NEVER_STARTED_TIMEOUT = 120` — cek vs delay giver real (NEVER_WAITED_FOR_LOAD 27s + teleport ~15s + request pertama). Mungkin 150s.
- **K-2** `TELEPORT_CONFIRM_WINDOW = 8` — kemungkinan kekecilan (CL-14); 12–15s.
- **K-3** `FOLLOWUP_WINDOW_SEC = 300` (+30) — kepanjangan buat throughput idle (CL-25); 60–90s atau decouple.
- **K-4** `POST_TRADE_GRACE_SEC = 10` — oke; verifikasi vs delay pra-trade giver 10s.
- **K-5** `HITS_VICTIM_LEAVE_TIMEOUT_MS = 40s` vs heartbeat giver 10s — jaga ≥ 3× heartbeat; kalau heartbeat→8s (GV-5), 40s tetap aman.
- **K-6** `HITS_STALE_PROCESSING_MS = 5min` — harus > umur maksimum 1 hit (CL-20) biar hit lambat-tapi-valid nggak di-recover mid-trade.
- **K-7** `CONFIRM_RESOLVE_TIMEOUT=12 / CONFIRM_HARD_TIMEOUT=20` — verifikasi vs ~7s latency TradeStatus; oke, ikat ke CL-6.

---

## PRIORITAS

1. **C-1 … C-11** (fix kedua bug + tegakkan aturan A-1)
2. **CL-25 / GV-14 / SV-2** (idle pickup — win "proses instan saat idle")
3. **G-1 … G-17** (re-theme GUI dark black-grey)
4. **BR-1 … BR-10 + T-1 … T-15** (instrumentasi + verifikasi pakai bridge dengan data asli)
5. Sisanya per-section.
