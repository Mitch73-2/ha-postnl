# Working in this repository

Home Assistant custom integration for PostNL parcel tracking **plus MyMail
letters and per-letter image entities**. Distributed via HACS; not part of HA
core. **Silver** quality tier, minimum HA `2024.7.0`. A **fork** of
`arjenbos/ha-postnl` (see below). Three APIs behind one bearer token.

## Shared conventions — fetch when relevant

Suite-wide rules live in
[`.github/CONVENTIONS.md`](https://github.com/ha-parcel-integrations/.github/blob/main/CONVENTIONS.md)
and are **not** repeated here. Don't fetch it every session — fetch it **before**
you act in one of these areas:

| Before you … | Fetch `CONVENTIONS.md` § |
|---|---|
| touch entities, sensors, **image entity**, config/options flow, coordinator, diagnostics, translations | *Home Assistant developer docs* (its table points on to the canonical HA page — the MyMail photos use the [image entity](https://developers.home-assistant.io/docs/core/entity/image) page). Don't rely on memory |
| add/rename a parcel field, a `ParcelStatus`, or a bus event; change first-refresh or unmapped-status logging | *Parcel contract* (this repo implements it; below is only where PostNL deviates) |
| consider "fixing" a lint/pattern the skill flags (poll interval, `requests`/sync, inline client) | *Deliberate skill divergences* — likely intentional, don't re-flag |
| commit, bump, tag, release, or write release notes; add a feature without a test | *Workflow / Commits / Versioning / Testing* |

**Suite-wide tripwire, kept inline on purpose:** the first refresh runs in
`__init__.py` *before* `async_forward_entry_setups` — `async_setup_entry` sets
`entry.runtime_data` (the coordinator reads `runtime_data.auth`) then awaits
`coordinator.async_config_entry_first_refresh()`. From a forwarded platform HA
can't catch `ConfigEntryNotReady` and half-sets-up the entry. This also
guarantees `coordinator.letters` is populated before `image.py` registers its
initial entities (a listener callback after setup). Runtime-only; tests don't
catch a regression here. Do not move it back into a platform.

## Load-bearing PostNL decisions — do not refactor away

Each line is a guardrail: the rule, then why it must stay. The code holds the
detail; this list stops you re-breaking a past fix.

**Auth & token refresh (do not weaken)**
- **PKCE login with re-login fallback** (`auth.py` / `AsyncConfigEntryAuth`):
  try a refresh-token exchange first; on failure re-run the full
  username/password login; reauth is the last resort. Order matters — don't
  reorder. This deliberately avoids HA's `OAuth2Session` (which would
  re-introduce the browser-extension onboarding the fork dropped).
- `check_and_refresh_token` **preserves the old refresh token** when PostNL's
  response omits a new one, and holds an **`asyncio.Lock`** (with a re-check
  inside) so two callers never spend the same rotating token twice.
- **Auth-error split.** Only a definitive credential rejection
  (`PostNLInvalidAuth`) escalates to `ConfigEntryAuthFailed` / reauth. Any other
  `PostNLAuthError` (recaptcha, rate-limit, changed widget, network blip) →
  generic `HomeAssistantError` → retryable `UpdateFailed` / `ConfigEntryNotReady`.
  This stopped the "logged out ~once a day" bug — do not collapse these.
- **Reauth guards the account**: `reauth_confirm` uses `async_set_unique_id` +
  `_abort_if_unique_id_mismatch` so a *different* account's credentials abort
  instead of rebinding.

**The three APIs & their quirks**
- **GraphQL** (`graphql.py`, shipment list), **Track & Trace** (`jouw_api.py`,
  per-shipment status + MyMail letters + image bytes), **Login** (`login_api.py`,
  userinfo). All share one bearer token via `auth.py`.
- **Every `jouw_api` call has a `(10, 60)` timeout** — `requests` has no
  session-level default; a hanging PostNL server would block an executor thread
  (and the whole refresh) forever.
- **API clients are reused across polls** — `PostNLGraphql`/`PostNLJouwAPI` are
  rebuilt only when the access token changes (`_api_token`); each owns a
  `requests.Session` connection pool that would otherwise leak every poll.
- `aiohttp.ClientError` is not caught in the coordinator (wrapped
  automatically); `requests` errors *are* caught (executor jobs re-raise them).
- **`jouw.postnl.nl` is the universal backend — never route to `.be`.** The
  GraphQL inbox is account-scoped, not domain-scoped (`.be` returns a
  byte-identical list); MyMail on `.be` returns HTTP 400 (NL-only feature). A
  NL/BE dropdown would be a no-op for parcels and break letters — do not add one.
  Belgian accounts are already covered. The real Belgium gap is **bpost**.

**MyMail letters & images**
- **Server-driven-UI payload**, not a clean letter list: `extract_letters`
  walks `screen.sections[].items[]` for `type == "Letter"`. Dates come as
  `"16 juni"` (no year); `parse_letter_date` infers the year from the ~2-week
  retention window.
- **MyMail needs app-identification headers** (`PostNLJouwAPI.mymail_headers`:
  `api-version`, `app-platform`, `device-token`, …) mirroring the mobile app —
  values occasionally need bumping when PostNL ships a new app version.
- **Letter image URLs require auth** — the `PostNLLetterImage` entity fetches
  bytes server-side via `PostNLJouwAPI.image()` and serves them through HA's
  authenticated image proxy. Do not switch to a redirect scheme.
- `postnl_letter_announced` fires per new letter; `_known_letter_ids` mirrors
  `_known_state`, reset only after a successful letters fetch.

**Status mapping & per-parcel resilience**
- **PostNL status is a Dutch human string, not an enum** — `map_parcel_status`
  uses **ordered substring patterns** (more specific first, e.g. "wordt vandaag
  bezorgd" before "bezorgd"); the raw string lives on `raw_status`, never
  `status`. Unmapped → `ParcelStatus.UNKNOWN`.
- **`receiver` / `weight` / `dimensions`**: `receiver` from
  `colli.recipient.names.personName` (active) or GraphQL `receiverTitle`
  (delivered short-circuit). `weight`/`dimensions` from `colli.details.dimensions`
  (native g+mm) via `_convert_native_dimensions` → canonical kg+cm with the long
  edge as `length` and a `"L x W x H cm"` `text`; native dict kept under `raw`.
  Delivered parcels skip T&T → both `None`.
- **One broken parcel no longer fails the refresh.** The active-path T&T call
  degrades per parcel: reuse the last good transform (`_parcel_cache`, only from
  real colli data, pruned each poll), else GraphQL-only fields; `UpdateFailed`
  is the last resort when there's nothing to show.
- Unknown-status warnings fire once per distinct value from **both**
  `map_parcel_status` and `map_observation_status`, with an `issues/new` link
  (`_NEW_ISSUE_URL`); one-shot sets `_LOGGED_UNKNOWN_STATUSES` /
  `_LOGGED_UNKNOWN_OBSERVATION_CODES`.

**History (opt-in, default OFF — `CONF_INCLUDE_HISTORY`)**
- Top-level `history`: ordered `{timestamp, status, raw_status}`, capped at
  `HISTORY_MAX_EVENTS` (20). Built by `build_history` from T&T
  `analyticsInfo.allObservations` (`_extract_observations` prefers it over the
  truncated `observations`). Top-level so it survives `strip_raw()`; `None` when
  off.
- **Delivered parcels get history too** — the delivered short-circuit makes the
  extra T&T call via `_delivered_history`. **Non-fatal** (a `RequestException`
  falls back to `None`), cached per barcode (`_delivered_history_cache`, one call
  per parcel ever); failures are NOT cached so the next poll retries.
- Per-event status maps from the stable `observationCode` via
  `_OBSERVATION_CODE_MAP` + `map_observation_status` (NOT the Dutch text).
- **Milestone vs meta + carry-forward (do not undo).** Only milestone codes
  carry a movement status; meta codes (`_OBSERVATION_META_CODES`: ETA recalcs,
  enrichment, …) inherit the previous milestone's stage so the timeline never
  bounces backward on a cosmetic event. Baseline before the first milestone is
  `registered`. The one legitimate step-back is a real delay/failure
  (`G01`/`G05`/`T04` → `in_transit`). Unmapped codes stay `null` and do NOT carry
  forward. A fixed status for ETA codes is wrong by construction.

**Events, triggers & surfaces**
- Incoming events (`postnl_parcel_registered` / `_status_changed` / `_delivered`
  / `_delivery_time_changed`) run over the **full receiver list** (active +
  delivered) so the terminal hop is visible: change **to** DELIVERED fires only
  `_delivered`; already-delivered fires nothing; `registered` only for
  not-yet-delivered new barcodes. `delivery_time_changed` only on a non-null
  `planned_*` that differs — `value → null` is silent. State in `_known_state` /
  `_known_delivery_times`.
- Outgoing (`postnl_outgoing_parcel_status_changed` / `_outgoing_parcel_delivered`)
  run over the **full `data['sender']`** list — own shipments *and* returns both
  land in `senderShipments`, so returns are covered for free.
  `delivered` wins the terminal hop; **no** outgoing `registered` /
  `delivery_time_changed`. State in `_known_outgoing_state`.
- `device_id` on every payload (`_fire_change_events` / `_fire_letter_events`,
  cached in `_cached_device_id`). `device_trigger.py` exposes six no-code
  triggers (four parcel + `letter_announced` + the outgoing pair) under
  `device_automation.trigger_type`.
- **Sensor cleanup is sensor-scoped**: filter `entity_entry.domain == "sensor"`
  before treating an `{account_id}_*` unique_id as a barcode, else it deletes the
  refresh button **and the letter image entities**. `_last_update` (and other
  non-parcel `{account_id}_*` sensors) **must** stay in `non_parcel_unique_ids`.
- **Refresh `button`** (`{account_id}_refresh`), **diagnostic `last_update`
  sensor** (reads `coordinator.last_success_time`, stamped before
  `_async_update_data` returns), **deliveries `calendar`**
  (`{account_id}_deliveries`, read-only over non-delivered receiver parcels, no
  extra API calls, enabled by default; letters are NOT on it). Per-parcel sensors
  are removed by the summary sensor via `entity_registry.async_remove` (the old
  self-remove raced and left ghosts).
- **Entities**: `has_entity_name = True` + `translation_key` everywhere (no
  `_attr_name`); icons in `icons.json`, translated unit-of-measurement (no
  `_attr_icon` / `_attr_native_unit_of_measurement`); device name
  `"PostNL (<email>)"`; `_attr_attribution`; `_unrecorded_attributes` keeps
  parcel/letter lists (and `history`) out of the recorder.
- **Options flow** has no `entry.add_update_listener` — `async_schedule_reload`
  on submit. `CONF_REFRESH_INTERVAL` = 15/30/60/120/240 min, default 30.

## Planned / skipped

- **Planned (next major)**: exception translations (`UpdateFailed` f-strings →
  `translation_key` + placeholders); per-letter events (e.g.
  `postnl_letter_received`) instead of the watch-the-count workaround.
- **Skipped on purpose**: slimming `extra_state_attributes` (recorder handled);
  `async-dependency` / `inject-websession` (Platinum) — the APIs use `requests`
  via executor jobs, aiohttp would be a big refactor for marginal gain.

## Fork / upstream relationship

Fork of [`arjenbos/ha-postnl`](https://github.com/arjenbos/ha-postnl), maintained
by [@peternijssen](https://github.com/peternijssen). HACS releases ship from this
fork; fixes that apply upstream are filed as separate PRs against `arjenbos/main`.
`manifest.json` still lists `@arjenbos` as codeowner. Cross-repo coordination is
in `CHANGES.md`. Branding uses the upstream assets in `home-assistant/brands`
(PostNL has a stable core icon) — unlike the other carriers' local `brand/`.

## Running tests

```
python -m pytest tests/ --cov=custom_components.postnl
```

Coverage must stay **above 95%** (silver `test-coverage` rule). Run before
committing.
