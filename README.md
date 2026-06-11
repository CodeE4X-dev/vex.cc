# TODO — Round 2 (Claimer / giver / server / stock audit)

Investigated against `files/Claimer.lua`, `files/giver.lua`, `server.js`, `files/Stock*.lua`.
Two reported bugs are root-caused below (they are the SAME underlying bug). Plus 100+ fix/improve/add/remove/test items.

Approve point-by-point (e.g. "C-1, C-2, B-3..B-9 ok") and I implement (no code comments).

---

## ROOT-CAUSE OF THE TWO REPORTED BUGS

### Bug A — embed shows "client-release: trade never started within 120s" on a trade that SUCCEEDED
- Success path with victim still present (`Claimer.lua:1583-1602`) sets `startTime = 0` + `lastSuccessAt = os.time()` but keeps `hitId` and **does not reset `claimedAt`**.
- The watchdog at `Claimer.lua:443` fires when `claimedAt > 0 and startTime == 0` — it does **not** exclude `lastSuccessAt > 0`. So a deferred-success looks identical to never-started, and once `os.time() - claimedAt > 120` it calls `releaseStuckClaim("trade never started")` → posts `failed`.

### Bug B — "🔄 Claimer X left server; listener stays armed for rejoin" when claimer didn't really leave
- Chain reaction from Bug A: after the false `failed`, claimer abandons + ServerHops, so giver's raw `Players.PlayerRemoving` (`giver.lua:1157`) logs "left". The claimer left only because Bug A killed a good trade.

---

## CRITICAL FIXES (do first)

- **C-1** `Claimer.lua:443` — add `and (currentTradeData.lastSuccessAt or 0) == 0` to the never-started guard so a deferred-success is never treated as never-started. (Primary fix for Bug A.)
- **C-2** `Claimer.lua:1583` deferred-success path — also reset `currentTradeData.claimedAt = 0` (or refresh it to `os.time()`) so the 120s window can't fire against an already-successful chain.
- **C-3** `Claimer.lua:402 releaseStuckClaim` — early-return if `(currentTradeData.lastSuccessAt or 0) > 0`; never post `failed` after any success. Log + finalise as success instead.
- **C-4** `releaseStuckClaim` — before posting `failed`, re-read hit status from server (`GET /api/hits/queue` or a `GET /api/hits/:id`) and skip if already `success`. Prevents racing a server-side success.
- **C-5** Watchdog loop (`Claimer.lua:418`) — when `lastSuccessAt > 0`, switch from "never started" logic to "deferred chain" logic (wait for victim leave / followup window), never to `releaseStuckClaim`.
- **C-6** `giver.lua:1157` — split the log: if the claimer left AFTER a confirmed trade with us, log "✅ Claimer X done & left" (success), not "left; armed for rejoin". Track a per-claimer `tradedOnce` flag.
- **C-7** `giver.lua` PlayerRemoving — debounce: wait 2–3s and re-check `Players:FindFirstChild(name)` before declaring "left" (covers same-frame respawn/relog flaps).
- **C-8** Add a server endpoint `GET /api/hits/:id` returning a single hit (used by C-4); currently only `/api/hits/queue` exists.
- **C-9** `Claimer.lua` — when posting any final status, include the locally-known `itemsTransferred`/`rapTransferred` so a late success isn't recorded as 0/0.
- **C-10** `server.js:5015 /status` — when a `failed` arrives but the hit already has `itemsTransferred > 0` recorded this session, log a warning and prefer the richer record (defensive against Bug A residue).

---

## CLAIMER.LUA — TRADE STATE MACHINE

- **CL-1** Reset `claimedAt = 0` in every terminal path (`finishHit`, `abandonClaimLocal`, `releaseStuckClaim` already do; audit deferred + followup paths).
- **CL-2** Replace the 3 near-duplicate `currentTradeData = { ... }` re-inits (`~518`, `~564`) with a single `resetTradeData(keepSuccess)` helper to stop drift between them.
- **CL-3** Single source of truth for "trade active": one `isTradeActive()` instead of mixing `targetPlayer ~= ""`, `hitId ~= nil`, `startTime ~= 0`, `confirmed`.
- **CL-4** Cancel the `FOLLOWUP_WINDOW_SEC + 30` `task.delay` (`1596`) when the hit finalises early, else it fires stale against a recycled `hitId`.
- **CL-5** Guard the delayed-followup closure with the captured `savedHit` AND `lastSuccessAt` snapshot to avoid acting on a new claim that reused state.
- **CL-6** Confirm watchdog (`1626`) can fire `recordTradeResult("success")` even if no items moved — require `itemsTransferred > 0` OR `confirmed` before claiming success, else mark `failed`/`unknown`.
- **CL-7** `tradeItemsCount`/`tradeRAPValue` are module-level and reused across rounds; snapshot them per-confirm to avoid double counting / stale RAP.
- **CL-8** RAP accumulation (`1702-1703`) assumes every `Confirmed` is a new batch — dedupe by trade-session id so a re-fired `Confirmed` doesn't double the totals.
- **CL-9** `recordTradeResult` "should record" heuristic (`496-511`) can mark a real success as not-recorded ("random-or-cancelled") if `expectedTarget` was cleared early — record items/RAP regardless of the success-rate decision.
- **CL-10** `postHitStatus` has no retry; wrap in 2–3 attempts w/ backoff so a dropped final status isn't lost (causes server-side stale→timeout later).
- **CL-11** `postHitStatus` should treat HTTP 409 `already-finalised`/`cannot-downgrade-success` as terminal-OK (stop retrying), not as failure.
- **CL-12** Add idempotency: include a `clientFinalId` (hitId + result) so duplicate final posts are no-ops server-side.
- **CL-13** `TeleportInitFailed` handler (`460`) calls `releaseStuckClaim` unconditionally — guard with `lastSuccessAt == 0` too (don't fail a finished chain on a late teleport error).
- **CL-14** Wrong-server release (`447`, `TELEPORT_CONFIRM_WINDOW = 8s`) is aggressive; raise to ~15s or verify teleport actually queued before releasing.
- **CL-15** Persisted `loadHitState()` is read twice per tick (`441-442`); read once and cache.
- **CL-16** Watchdog `task.wait(10)` means up to 10s late detection of teleport failure — split into a fast (2s) teleport-confirm check and slow (10s) heartbeat.
- **CL-17** On `claimerHandleCmd` transfer-items, 90 items × `PER_ADD_DELAY` can block ~45s+30s; run as `task.spawn` so the heartbeat coroutine isn't starved (prevents claimer being marked stale).
- **CL-18** Same for transfer-tokens 30s confirm loop — ensure heartbeat keeps firing during it.
- **CL-19** `ReceivedTradeRequest` auto-accept should re-validate giver against the active hit's expected victim, not accept any giver while a hit is held.
- **CL-20** Add a hard ceiling on a single hit's lifetime (e.g. 8 min) → finalise as `timeout` locally and free the claimer rather than relying only on server stale recovery.
- **CL-21** Log the actual `postHitStatus` HTTP code + body to GUI/console for every final post (testing visibility).
- **CL-22** When `recordTradeResult` decides `failed`, include the last server heartbeat code + age so the embed message is diagnostic, not just "trade never started".
- **CL-23** Track `confirmedTriggered`/`confirmed` reset on new claim to avoid a stale `confirmed=true` short-circuiting the next victim's auto-ready.
- **CL-24** `isTradeUiActive()` is polled in 3 loops; consolidate to one signal-driven state.

## CLAIMER.LUA — CLAIM / IDLE PICKUP

- **CL-25** Deferred-success holds `hitId` up to `FOLLOWUP_WINDOW_SEC+30` (=330s) → claimer reports `busy` and won't grab new hits. Shorten followup to ~60–90s, OR allow claiming a new hit while a followup is pending (decouple "busy" from "followup").
- **CL-26** Heartbeat status (`Claimer.lua:~1762`) reports `busy` whenever `hitId` set — report `followup` separately so dashboard shows the claimer is actually free-ish.
- **CL-27** `fireClaim` thundering herd: every long-poll waiter wakes and all POST `/api/hits/claim` at once; only one wins (15s claim lock), rest waste a request. Add small jitter or server-side per-claimer assignment.
- **CL-28** Long-poll loop `task.wait(0.2)` after each response is fine, but on error path it does `task.wait(2)` then `task.wait(0.2)` — collapse to a single backoff.
- **CL-29** `requestHitClaim` has no preferredVictim use; could pass last-known victim to bias toward retrades.
- **CL-30** Add exponential backoff + cap on `connectWebSocket` reconnect (`1241`) — currently `task.wait(1)` tight recursive loop can spin if gateway rejects.
- **CL-31** `connectWebSocket` recursion grows the call stack on every reconnect — convert to a `while true` loop.
- **CL-32** Discord heartbeat loop inside `OnMessage` (`1223`) never exits on close — leaks a coroutine per reconnect. Tie it to a token/generation that the OnClose invalidates.
- **CL-33** `is_channel_supported` + `Action(message.content)` path: validate/whitelist commands; a malformed Discord message shouldn't crash the handler (wrap in pcall).
- **CL-34** Fallback claim timer (20s) + long-poll both call `fireClaim`; ensure `claimWakeup` can't double-process within the same tick.

## CLAIMER.LUA — CLEANUP / REMOVE

- **CL-35** Strip all code comments in `Claimer.lua` (user rule: no comments).
- **CL-36** Remove the dead `TradeInProgress` references mentioned in the comment at `1672` (confirm none remain).
- **CL-37** Remove `print("[SuccessRate] ...")` debug prints or route them through `notifyGUI`/logger consistently.
- **CL-38** De-duplicate the two identical `currentTradeData = {...}` blocks (see CL-2).
- **CL-39** Consolidate repeated `http_request or (syn and syn.request) or request` into the existing `getRequestFunc()` everywhere.

## CLAIMER.LUA — GUI

- **CL-40** Show current hit state (IDLE / CLAIMED / TRADING / CONFIRMING / FOLLOWUP / FINALISING) as an explicit status line.
- **CL-41** Show last final result + reason + items/RAP transferred.
- **CL-42** Show last `postHitStatus` HTTP code (green/red dot) for at-a-glance health.
- **CL-43** Show queue pending/processing counts (already polled at `869`) in the panel.
- **CL-44** Show websocket/long-poll connection health + last wake time.
- **CL-45** Expandable log panel (matching Stock GUI style) with severity colors.
- **CL-46** Button: "Force release current hit" for manual recovery.
- **CL-47** Button: "Re-fire claim now" (manual `fireClaim`).
- **CL-48** Countdown to next fallback claim + followup-window remaining.

---

## GIVER.LUA

- **GV-1** (C-6/C-7) Distinguish "claimer done+left" vs "claimer left mid-trade" in the PlayerRemoving log.
- **GV-2** Track per-claimer `tradedItemsCount` so the log can say how much was taken before they left.
- **GV-3** `startedClaimers[name] = nil` on leave wipes progress; persist a "completed with X" record so a rejoin doesn't restart a finished claimer.
- **GV-4** Victim heartbeat loop (`925-944`) only breaks on 409/404 — also break/stop when the giver's own trade is fully done to free the loop.
- **GV-5** Heartbeat `task.wait(10)` vs server 40s leave-timeout gives only ~3 missed beats of slack; lower to 7–8s for safety margin during lag.
- **GV-6** Heartbeat coroutine can be starved by the giver's long trade loops — verify it runs in its own `task.spawn` unaffected by trade waits.
- **GV-7** `startTradeOnce(p, 99)` initial delay 10s (`PlayerAdded`) can exceed claimer's teleport-confirm assumptions; align timings with Claimer's `TELEPORT_CONFIRM_WINDOW`.
- **GV-8** Re-arm-on-rejoin can loop forever if a claimer crashes repeatedly; cap re-arm attempts per claimer.
- **GV-9** Validate the claimer name against the active hit's claimer (`/api/hits/queue`) so a random whitelisted name leaving doesn't emit a misleading "left" log.
- **GV-10** Add a webhook log when the victim heartbeat first returns 409 (server already finalised) so giver+claimer timelines line up in testing.
- **GV-11** Log the hit `id` in every giver log line so a hit can be traced across giver↔claimer↔server.
- **GV-12** Strip all comments in `giver.lua`.
- **GV-13** Send giver-side `itemsRemaining` count with heartbeat so server/claimer can decide retrade vs finalise smartly (instead of fixed followup window).
- **GV-14** On final success, proactively POST a "victim-finished" to the server so claimer's deferred path resolves immediately (kills the 330s wait — ties to CL-25).
- **GV-15** Guard `isInventoryStillGood` re-trade loop against trading the same locked items repeatedly (skip TradeLock/Listing items).

---

## SERVER.JS — HIT QUEUE

- **SV-1** Add `GET /api/hits/:id` (single hit) — needed by Claimer C-4 to verify status before posting failed.
- **SV-2** Add `POST /api/hits/:id/victim-finished` (or reuse status `leave`/flag) so giver can signal "inventory empty / done" → server finalises + wakes claimer instantly (ties GV-14 / CL-25).
- **SV-3** `/status` (`5015`) — when downgrading is rejected (409), return the existing record so the client can self-heal (already returns reason; include `itemsTransferred`).
- **SV-4** Record a structured event log per status transition (id, from, to, claimer, ts, message) to a ring buffer + `GET /api/hits/events` for testing/timeline.
- **SV-5** Stale-recovery (`4296`, 5min) and victim-leave (40s) thresholds should be per-hit overridable via env already — surface them in `/api/hits/stats` for visibility.
- **SV-6** When a hit is recovered from stale (`4303`), increment a metric + log which claimer dropped it (helps spot Bug A pattern: success-then-stale).
- **SV-7** `notifyHitsWaiters` clears ALL waiters on any version bump — fine, but add the changed hit id(s) in the wait response so claimers can skip irrelevant wakes.
- **SV-8** Optional true WebSocket (`ws` pkg) endpoint `/ws/hits` pushing `{type:'hit', id}` to idle claimers; keep long-poll as fallback. (User asked — but note long-poll already ~0.2s; WS mainly cuts request overhead + enables targeted assignment.)
- **SV-9** Server-side assignment: instead of N claimers racing `/claim`, let server pick an idle claimer (by heartbeat) and push the hit to it → eliminates thundering herd (CL-27).
- **SV-10** Track claimer presence from `/api/claimers/heartbeat` and expose `idle` list so SV-9 can target.
- **SV-11** Guard `/claim` against handing the same victim to two claimers in different servers (dedupe by victim+jobId).
- **SV-12** `HITS_CLAIM_LOCK_MS = 15s` global lock serialises ALL claims across all victims — make it per-nothing/optimistic (compare-and-set on the candidate) so unrelated claims don't block each other.
- **SV-13** Add `claimedByJobId` so a heartbeat/status from the wrong server instance is rejected (prevents cross-server status clobber).
- **SV-14** Validate `itemsTransferred`/`rapTransferred` are non-negative numbers; clamp.
- **SV-15** `recordHitToStats` failure is only `console.error` — add a retry/queue so stats aren't silently lost.
- **SV-16** Embed builder: include hit `id` (short) and claimer in the title/footer for traceability.
- **SV-17** Persist `hitsVersion` so a server restart doesn't reset long-poll versions to 0 and falsely signal "change" to every claimer.
- **SV-18** `/api/hits/wait` max 30s; align with claimer's 25s + 0.2s loop — document and keep claimer timeout < server timeout to avoid double-wait.

## SERVER.JS — DISCORD EMBED

- **SV-19** "❌ FAILED" with message `client-release: trade never started` should be suppressed/downgraded if items were transferred (defensive vs Bug A; ties C-10).
- **SV-20** Show `itemsTransferred`/`rapTransferred` on success embeds.
- **SV-21** Show staleRecoveries count (already in code `4393`) consistently on all states.
- **SV-22** Color/title for `leave` vs `failed` vs `timeout` should be visually distinct (already labeled — verify embed color mapping).
- **SV-23** Truncate item lines at `\n` (already fixed in round 1 — verify applies to all embed builders, not just one).
- **SV-24** Add a "duration" field (claimedAt→finishedAt) to spot slow trades.
- **SV-25** Rate-limit Discord edits per hit (debounce rapid processing→success) to avoid 429s.

## SERVER.JS — ROBUSTNESS

- **SV-26** `withHitsQueue` write contention — ensure it serialises writes (file DB) so concurrent status+heartbeat don't lose updates.
- **SV-27** Validate all `:id` params (length/charset) before queue scan.
- **SV-28** Cap `statusMessage` already 280 — also strip control chars.
- **SV-29** `recoverStaleHits` runs inside `/claim`; also run it on a timer so stale hits recover even with no claimers polling.
- **SV-30** Add `/api/hits/admin/force-status` (auth) for manual dashboard recovery.
- **SV-31** Log when a `failed` immediately follows a `processing` from the same claimer within < a few seconds (Bug A signature) for monitoring.

---

## STOCK.LUA / STOCK2 / STOCK3

- **ST-1** Apply the same `executeRedistribute` pcall-wrapper guarantee to `sendTokensToTarget` (it still has manual `TradeInProgress=false` on every branch → leak risk on unexpected error). Wrap in `table.pack(pcall(...))`.
- **ST-2** `Stock.lua` — verify it received the same comment-strip + pcall wrapper as Stock2/Stock3 (round 1 did Stock.lua; confirm parity).
- **ST-3** `TradeInProgress` is a plain bool with no timeout — add a max-age so a wedged trade can't lock the stock forever.
- **ST-4** Whitelist refresh every 90s; cache and diff to avoid re-logging every entry each cycle (log spam).
- **ST-5** `getTokenCount` does `WaitReplion("Inventory")` every heartbeat — cache the replion handle once.
- **ST-6** `collectInventoryStats` iterates all items every heartbeat (60s) — fine, but reuse the same scan for redistribute decisions instead of re-scanning.
- **ST-7** Stock GUI: show online stock peers + their token/RAP (from `/api/stocks/list`) for at-a-glance consolidation state.
- **ST-8** Stock GUI: show current command being executed + last ack status.
- **ST-9** Stock GUI: live booth used/free count from `countMyListings` in the status line.
- **ST-10** `teleportWithRetry` recursion grows stack on each retry — convert to loop.
- **ST-11** Strip comments in Stock.lua (and confirm Stock2/3 fully stripped).
- **ST-12** `sendStockHeartbeat` and command-poll share request fn — unify error handling/backoff.
- **ST-13** Slot-conflict kick (`handleStockRegisterResponse`) — also stop all spawned loops, not just set `AllOperationsStopped`.

---

## TESTING & INSTRUMENTATION (log args / responses)

- **T-1** Add a global `DEBUG` flag in Claimer.lua; when on, log every remote `:InvokeServer` name + args (ReadyUp/ConfirmTrade/AddItemToTrade/etc).
- **T-2** Log every `/api/hits/*` request URL + body + response code + body (claimer side) to a ring buffer + GUI.
- **T-3** Log the full `ReceivedTradeRequest` args table (From, id) on each accept.
- **T-4** Log the full `TradeStatus` arg (`p1`) and the resulting branch taken (deferred / finalise / failed).
- **T-5** Log every replion `Set` event key + who (Ready/Confirmed/TradeItems) with values when DEBUG.
- **T-6** Dump `currentTradeData` snapshot on every terminal transition (before reset) for post-mortem.
- **T-7** server.js: `LOG_HITS=1` env to console.log every status transition with full hit diff.
- **T-8** server.js: `/api/hits/events` ring buffer (SV-4) surfaced on the dashboard for live timeline during testing.
- **T-9** Add correlation id (hit id) to EVERY log line across giver/claimer/server so one hit's lifecycle can be grepped end-to-end.
- **T-10** giver.lua DEBUG: log victim-heartbeat response codes each tick.
- **T-11** Add a synthetic "dry-run" mode on claimer: do everything but skip the final trade confirm, to validate state machine without moving items.
- **T-12** Add timing logs (claimedAt, firstTradeReq, firstConfirm, finalised) to measure real latencies vs the 120s/40s/330s constants — use real numbers to retune them.
- **T-13** Log when watchdog WOULD have released but was suppressed by the new `lastSuccessAt` guard (proves Bug A fix is catching it).
- **T-14** server.js: count + expose how many `failed` carried message `client-release: trade never started` per day (regression metric for Bug A).
- **T-15** Add a `/api/hits/selftest` that walks a fake hit pending→processing→success to validate the pipeline without a real victim.

---

## CONSTANTS TO RETUNE (after T-12 data)

- **K-1** `TRADE_NEVER_STARTED_TIMEOUT = 120` — verify vs real giver delay (NEVER_WAITED_FOR_LOAD 27s + teleport ~15s + first request). Possibly 150s.
- **K-2** `TELEPORT_CONFIRM_WINDOW = 8` — likely too low (CL-14); 12–15s.
- **K-3** `FOLLOWUP_WINDOW_SEC = 300` (+30) — too long for idle throughput (CL-25); 60–90s or decouple.
- **K-4** `POST_TRADE_GRACE_SEC = 10` — fine; verify against giver's 10s pre-trade delay.
- **K-5** `HITS_VICTIM_LEAVE_TIMEOUT_MS = 40s` vs giver heartbeat 10s — keep ≥ 3× heartbeat; if heartbeat→8s (GV-5), 40s stays safe.
- **K-6** `HITS_STALE_PROCESSING_MS = 5min` — should be > max single-hit lifetime (CL-20) so a slow-but-valid hit isn't recovered mid-trade.
- **K-7** `CONFIRM_RESOLVE_TIMEOUT=12 / CONFIRM_HARD_TIMEOUT=20` — verify vs observed ~7s TradeStatus latency; OK but tie to CL-6.

---

## PRIORITY

1. C-1 … C-10 (fixes both reported bugs)
2. CL-25 / GV-14 / SV-2 (idle pickup — the real "process instantly while idle" win)
3. T-1 … T-15 (instrumentation so we can verify everything with real args/responses)
4. Everything else by section.
