# AUDIT.md — Sprint 1 · Bridge + Daten

> **Status 2026-06-08:** Bridge fertig und live verifiziert (O4 erledigt).
> Datenpipeline und M5-Aggregation implementiert, getestet (102/102 Tests) und
> live verifiziert: 84 Mio. EURUSD-Ticks geladen (1243 Tage), Tick→M5-
> Konsistenzcheck bestanden (100.000 Bars, max Abw. 0,2 Pip, kein Flag).
> Look-ahead per Bar-Schlusszeit abgesichert (Opus-Gate).
> Guardrail-Schicht (O6) noch ausstehend. Sprint 1 **nicht abgenommen**.

---

## 1. Was gebaut wurde

### `bridge/` — FastAPI-Server (läuft auf dem Windows-Server)

| Datei | Inhalt |
|---|---|
| `bridge/main.py` | App-Einstiegspunkt; `lifespan`-Handler (MT5 connect / shutdown); `GET /health` |
| `bridge/config.py` | `pydantic-settings`; liest `.env`; `ORDERS_ENABLED` default `False` |
| `bridge/auth.py` | `HTTPBearer`-Dependency `verify_token`; vergleicht Token gegen `ORCA_BRIDGE_TOKEN` |
| `bridge/mt5_client.py` | Dünner MT5-Wrapper; `ensure_connected()` (ein Reconnect-Versuch) |
| `bridge/models.py` | Pydantic-Response-Modelle: `AccountInfo`, `Position`, `TickData`, `HealthStatus` |
| `bridge/routers/account.py` | `GET /account` |
| `bridge/routers/positions.py` | `GET /positions` |
| `bridge/routers/ticks.py` | `GET /ticks/{symbol}` |
| `bridge/routers/order.py` | `POST /order` (Stub, deaktiviert) |
| `bridge/requirements.txt` | Server-Abhängigkeiten (FastAPI, uvicorn, pydantic-settings) |

### `bridge/routers/history.py` — History Endpoints (Bridge-Seite)

| Endpunkt | MT5-Aufruf | Chunking |
|---|---|---|
| `GET /history/ohlc?timeframe=...&from=...&to=...` | `mt5.copy_rates_range` | H4/H1: 30-Tages-Chunks; M15/M5: 7-Tages-Chunks |
| `GET /history/ticks?from=...&to=...` | `mt5.copy_ticks_range` | 1-Tages-Chunks |

Beide Endpunkte: Token-Auth, `require_demo()`, nur EURUSD, Zeiten strikt UTC.

### `orca/data/` — Mac-seitige Datenpipeline

| Datei | Inhalt |
|---|---|
| `orca/data/store.py` | Parquet-Speicher unter `data/` (gitignored); inkrementelles Merge; `save_m5_from_ticks()` / `load_m5_from_ticks()` |
| `orca/data/downloader.py` | Synchroner HTTP-Client für Batch-Downloads über die Bridge |
| `orca/data/pipeline.py` | Inkrementeller Load + Validierung + Save; `run_ohlc(tf)`, `run_ticks()` |
| `orca/data/accessor.py` | Look-ahead-sichere API: `as_of(df, t, bar_duration)` (close-time-Semantik für OHLC, Opus-Gate-Pflicht) + `ForwardIterator` |
| `orca/data/validator.py` | Lücken, Duplikate, Monotonie, bid≤ask; `validate_ohlc()`, `validate_ticks()` |
| `orca/data/aggregator.py` | Deterministisches Tick→M5-Resampling (Bid-OHLC + Mid-OHLC + Spread); `aggregate_ticks_to_m5()`, `validate_m5_vs_native()` |

### `orca/bridge_client.py` — Async HTTP-Client (Mac-Seite)

Async Context Manager `BridgeClient`; initialer Probe auf `__aenter__`; Background-Heartbeat-Task.

### `tests/`

`conftest.py` (MT5-Stub), `test_bridge_endpoints.py` (20 Tests), `test_bridge_client.py` (4 Tests),
`test_bridge_history.py` (9 Tests), `test_data_accessor.py` (8 Tests),
`test_data_validator.py` (7 Tests), `test_data_store.py` (7 Tests).

---

## 2. Endpunkte

| Methode | Pfad | Auth | Beschreibung |
|---|---|---|---|
| GET | `/health` | Keine | Heartbeat; liefert `{"status":"ok","mt5_connected":bool,"timestamp":…}` |
| GET | `/account` | Bearer | Kontoinfo: Login, Balance, Equity, Margin, Currency, Leverage, Server, `is_demo` |
| GET | `/positions` | Bearer | Offene Positionen als Liste; optionaler Query-Parameter `?symbol=EURUSD` |
| GET | `/ticks/{symbol}` | Bearer | Aktueller Bid/Ask/Last für `symbol`; nur `EURUSD` erlaubt (400 sonst) |
| POST | `/order` | Bearer | **Stub — gibt 503 zurück** (siehe §4) |

---

## 3. Sicherheit

**Bind-Adresse:** `<bridge-host>:8000` (Tailscale-IP des Windows-Servers).
Die Bridge ist ausschließlich über das private Tailscale-Netz erreichbar, nicht
über das öffentliche Internet.
Deployment-Kommando: `uvicorn bridge.main:app --host <bridge-host> --port 8000`.

**Token-Auth:** FastAPI `HTTPBearer`-Dependency `verify_token` in `bridge/auth.py`.
Eingebunden via `dependencies=[Depends(verify_token)]` auf jedem geschützten Router.
Tokenpräfung mit `secrets.compare_digest` (timing-sicher).

| Fehlerfall | HTTP-Status | Detail |
|---|---|---|
| Kein `Authorization`-Header | 401 | FastAPI HTTPBearer wirft automatisch |
| Falsches Token | 401 | `"Invalid or missing bearer token"` |
| Nicht-Demo-Konto | 403 | `"…not a demo account…"` (SCOPE §2.1) |
| Korrektes Token | — | Request weiterverarbeitet |

`/health` ist bewusst ohne Auth: gibt keine sensitiven Daten zurück, muss auch ohne
Token vom Heartbeat-Client erreichbar sein (und bleibt auch bei 403-Status erreichbar).

**Demo-Hard-Refuse (SCOPE §2.1):** `connect()` ruft nach `mt5.initialize()` sofort
`mt5.account_info()` auf und prüft `trade_mode == ACCOUNT_TRADE_MODE_DEMO` (= 0).
Nicht-Demo-Konto: `mt5.shutdown()`, `_non_demo_refused = True`, kein Reconnect-Versuch.
Alle Datenpfade rufen `require_demo()` auf — löst `NotDemoAccountError` aus, die der
FastAPI-Exception-Handler zu HTTP 403 mappt.

---

## 4. `/order` — Feature-Flag-Stub

`POST /order` ist implementiert aber **deaktiviert**.

- **Flag:** `ORDERS_ENABLED` (Env-Var, gelesen in `bridge/config.py`)
- **Default:** `False` (hard-coded in `Settings`-Klasse)
- **Verhalten bei `ORDERS_ENABLED=false`:** HTTP 503 mit Detail
  `"Order execution is disabled. Set ORDERS_ENABLED=true only after risk-execution guardrails are approved."`
- Auth-Prüfung findet trotzdem statt (falsches Token → 401, kein Token → 401, dann erst Feature-Flag).
- `ORDERS_ENABLED=true` darf **nur nach ausdrücklicher Supervisor-Freigabe** der
  Guardrail-Schicht gesetzt werden (SCOPE §3 / CLAUDE.md §Berechtigungen).

---

## 5. Heartbeat / Reconnect

**Bridge-Seite (`bridge/mt5_client.py`):**
`ensure_connected()` prüft `mt5.terminal_info()`; wenn `None`, wird einmal
`mt5.initialize()` aufgerufen. Kein Retry-Loop — dauerhafter Ausfall bleibt als Fehler sichtbar.

**Client-Seite (`orca/bridge_client.py`):**
- `__aenter__` führt einen initialen `_probe()` auf `/health` durch → `is_connected` sofort bekannt.
- `_heartbeat_loop()` pingt `/health` alle 30 s (konfigurierbar).
- Bei Verbindungsverlust (`httpx.ConnectError` / `httpx.TimeoutException` / non-200):
  `is_connected = False`, Log-Level ERROR:
  `"Bridge connection LOST — bot must halt. No automatic retries (SCOPE §3)."`
- Kein automatischer Retry von ausstehenden Calls — der aufrufende Bot-Loop muss
  `is_connected` vor jeder Aktion prüfen und bei `False` anhalten.
- Bei Wiederherstellung: `is_connected = True`, Log INFO `"Bridge connection RESTORED"`.

**Test-Abdeckung:**
- `test_is_connected_true_when_health_ok` — `is_connected=True` nach erfolgreichem Probe.
- `test_is_connected_false_when_bridge_unreachable` — `is_connected=False` wenn Probe fehlschlägt.
- `test_heartbeat_halt_on_connection_loss` — führt `_heartbeat_loop()` mit simuliertem
  `ConnectError` aus; assertiert `is_connected=False` und ERROR-Log `"LOST"` (O1 erledigt).
- `test_heartbeat_restored_sets_connected_true` — assertiert `is_connected=True` und
  INFO-Log `"RESTORED"` nach Wiederverbindung.

---

## 6. Tests

Alle Tests laufen auf macOS; `MetaTrader5` wird via `sys.modules`-Stub in `tests/conftest.py` gemockt.

| Datei | Test | Ergebnis |
|---|---|---|
| `test_bridge_endpoints.py` | `test_health_no_auth_required` | ✅ |
| | `test_health_mt5_disconnected` | ✅ |
| | `test_auth_missing_token_returns_401` | ✅ |
| | `test_auth_wrong_token_returns_401` | ✅ |
| | `test_get_account_demo` | ✅ |
| | `test_get_account_mt5_down` | ✅ |
| | `test_get_positions_empty` | ✅ |
| | `test_get_positions_with_data` | ✅ |
| | `test_get_positions_mt5_down` | ✅ |
| | `test_get_ticks_eurusd` | ✅ |
| | `test_get_ticks_symbol_case_insensitive` | ✅ |
| | `test_get_ticks_unsupported_symbol` | ✅ |
| | `test_get_ticks_mt5_down` | ✅ |
| | `test_connect_refuses_non_demo_account` | ✅ |
| | `test_non_demo_account_refused_all_data_endpoints` | ✅ |
| | `test_non_demo_refused_detail_mentions_demo` | ✅ |
| | `test_health_not_blocked_by_non_demo` | ✅ |
| | `test_order_disabled_by_default` | ✅ |
| | `test_order_disabled_even_with_bad_body` | ✅ |
| | `test_order_no_auth_returns_401` | ✅ |
| `test_bridge_client.py` | `test_is_connected_true_when_health_ok` | ✅ |
| | `test_is_connected_false_when_bridge_unreachable` | ✅ |
| | `test_heartbeat_halt_on_connection_loss` | ✅ |
| | `test_heartbeat_restored_sets_connected_true` | ✅ |
| `test_bridge_history.py` | `test_get_ohlc_h4` | ✅ |
| | `test_get_ohlc_unknown_timeframe` | ✅ |
| | `test_get_ohlc_mt5_none` | ✅ |
| | `test_get_ohlc_non_demo` | ✅ |
| | `test_get_ohlc_unsupported_symbol` | ✅ |
| | `test_get_ohlc_requires_auth` | ✅ |
| | `test_get_ticks_history` | ✅ |
| | `test_get_ticks_history_mt5_none` | ✅ |
| | `test_get_ticks_history_requires_auth` | ✅ |
| `test_data_accessor.py` | `test_as_of_returns_only_past_rows` | ✅ |
| | `test_as_of_no_future_data_leaks` | ✅ |
| | `test_as_of_naive_t_raises` | ✅ |
| | `test_forward_iterator_chronological` | ✅ |
| | `test_forward_iterator_no_rewind` | ✅ |
| | `test_forward_iterator_exhausted` | ✅ |
| | `test_forward_iterator_cursor_before_first_next_raises` | ✅ |
| | `test_forward_iterator_cursor_advances` | ✅ |
| | `test_m5_bar_invisible_before_close` | ✅ |
| | `test_m5_bar_visible_at_close` | ✅ |
| | `test_as_of_bar_duration_string_timeframe` | ✅ |
| | `test_as_of_bar_duration_unknown_tf_raises` | ✅ |
| | `test_h4_bar_not_visible_before_close` | ✅ |
| | `test_forward_iterator_with_bar_duration_cursor_is_close_time` | ✅ |
| `test_data_aggregator.py` | `test_aggregate_produces_m5_bars` | ✅ |
| | `test_aggregate_ohlc_correctness` | ✅ |
| | `test_aggregate_tick_volume` | ✅ |
| | `test_aggregate_spread_preserved` | ✅ |
| | `test_aggregate_mid_prices` | ✅ |
| | `test_aggregate_two_m5_bars` | ✅ |
| | `test_aggregate_missing_columns_raises` | ✅ |
| | `test_aggregate_single_tick_bar` | ✅ |
| | `test_aggregate_deterministic` | ✅ |
| | `test_aggregated_m5_bar_invisible_before_close` | ✅ |
| | `test_aggregated_m5_bar_visible_at_close` | ✅ |
| | `test_validate_m5_identical_bars_no_flags` | ✅ |
| | `test_validate_m5_large_deviation_flags` | ✅ |
| | `test_validate_m5_no_overlap_returns_error` | ✅ |
| | `test_validate_m5_reports_bar_count` | ✅ |
| `test_data_validator.py` | `test_validate_ohlc_clean` | ✅ |
| | `test_validate_ohlc_high_lt_low` | ✅ |
| | `test_validate_ohlc_duplicate_timestamps` | ✅ |
| | `test_validate_ohlc_empty` | ✅ |
| | `test_validate_ticks_bid_gt_ask` | ✅ |
| | `test_validate_ticks_clean` | ✅ |
| | `test_validate_ticks_empty` | ✅ |
| | `test_validate_ticks_same_second_different_msc_no_false_positive` | ✅ |
| `test_data_store.py` | `test_save_and_load_ohlc` | ✅ |
| | `test_incremental_save_deduplicates` | ✅ |
| | `test_latest_ohlc_time` | ✅ |
| | `test_load_ohlc_missing_returns_none` | ✅ |
| | `test_save_and_load_ticks` | ✅ |
| | `test_incremental_ticks_deduplicates` | ✅ |
| | `test_latest_tick_time_returns_none_when_empty` | ✅ |
| | `test_save_and_load_m5_from_ticks` | ✅ |
| | `test_m5_from_ticks_incremental_dedup` | ✅ |
| `test_measure_history_depth.py` | `test_fetch_ohlc_path_and_params` | ✅ |
| | `test_fetch_ticks_path_and_params` | ✅ |
| | `test_fetch_ohlc_http_error_returns_none` | ✅ |
| | `test_fetch_ohlc_network_exception_returns_none` | ✅ |
| | `test_no_double_slash_in_url` | ✅ |
| | `test_find_oldest_date_3y_boundary` | ✅ |
| | `test_find_oldest_date_2y_boundary` | ✅ |
| | `test_find_oldest_date_http_error_returns_none` | ✅ |
| | `test_find_oldest_date_history_exceeds_cap` | ✅ |
| | `test_find_oldest_date_precision` | ✅ |
| | `test_measure_ohlc_no_recent_data` | ✅ |
| | `test_measure_ohlc_detects_3y_history` | ✅ |
| | `test_measure_ohlc_detects_short_history` | ✅ |
| | `test_measure_ticks_no_recent_data` | ✅ |
| | `test_measure_ticks_reports_depth` | ✅ |
| `test_publish_audit_leak.py` | `test_token_leak_reports_key_name` | ✅ |
| | `test_token_leak_reports_line_number` | ✅ |
| | `test_server_leak_reports_key_name` | ✅ |
| | `test_bridge_url_host_leak_reports_key_name` | ✅ |
| | `test_tailscale_ip_reports_pattern` | ✅ |
| | `test_ts_net_hostname_reports_pattern` | ✅ |
| | `test_clean_audit_passes` | ✅ |
| | `test_placeholder_token_not_checked` | ✅ |
| **Gesamt** | **102 / 102** | **✅** |

---

## 7. Look-ahead-Bias

### 7a. Allgemein (Sprint 1)
Die Bridge ist eine reine I/O-Schicht — keine Strategie-Logik, kein Look-ahead-Risiko dort.

### 7b. Bar-Schlusszeit-Semantik (Sprint 3, Opus-Gate-Pflicht)
Ein OHLC-Bar mit Öffnungszeit T ist erst ab T + Timeframe-Dauer (= Schlusszeit) verfügbar.

| Timeframe | Öffnungszeit | Schlusszeit | `bar_duration` |
|---|---|---|---|
| H4 | 08:00 | 12:00 | `timedelta(hours=4)` / `"H4"` |
| H1 | 14:00 | 15:00 | `timedelta(hours=1)` / `"H1"` |
| M15 | 14:00 | 14:15 | `timedelta(minutes=15)` / `"M15"` |
| M5 | 14:00 | 14:05 | `timedelta(minutes=5)` / `"M5"` |

**API:** `as_of(df, t, bar_duration="M5")` — Vergleich `close_time <= t`.
Ohne `bar_duration` (Tick-/Event-Daten): `open_time <= t` (unverändertes Verhalten).
Opus-Gate-Pflicht: **Alle** Strategie- und Backtest-Aufrufe müssen `bar_duration` setzen.

**Testabdeckung:** `test_m5_bar_invisible_before_close`, `test_m5_bar_visible_at_close`,
`test_aggregated_m5_bar_invisible_before_close`, `test_aggregated_m5_bar_visible_at_close`
+ H4/H1-Varianten → schlagen fehl, sobald Look-ahead-Schutz entfernt wird.

### 7c. M5-aus-Ticks (Sprint 3)
Tick-aggregierte M5-Bars werden über `as_of(..., bar_duration="M5")` genauso geschützt
wie native Bars. Konsistenzcheck gegen native M5 live durchgeführt und bestanden (O5b erledigt — siehe §8).

---

## 8. Offene Punkte / Bekannte Risiken

| Nr. | Beschreibung | Priorität |
|---|---|---|
| ~~O1~~ | ~~"Bot must halt"-Verhalten nur strukturell geprüft~~ | **erledigt** — `test_heartbeat_halt_on_connection_loss` assertiert `is_connected=False` und ERROR-Log `"LOST"` |
| ~~O2~~ | ~~Token-Vergleich mit `==` statt `secrets.compare_digest`~~ | **erledigt** — `secrets.compare_digest` in `bridge/auth.py` |
| O3 | `ensure_connected()` macht nur einen Reconnect-Versuch; bei dauerhaftem MT5-Ausfall kein weiterer Retry (by design) | Info |
| ~~O4~~ | ~~Bridge-Live-Test (Mac ↔ Windows über Tailscale) noch nicht durchgeführt~~ | **erledigt** — `/health` antwortet HTTP 200 über das Tailscale-Netz; `/account` bestätigt `is_demo: true` (`trade_mode == ACCOUNT_TRADE_MODE_DEMO`); Demo-Hard-Refuse aktiv |
| ~~O5~~ | ~~Datenpipeline: Live-Datenlauf + vollständiger Download ausstehend~~ | **erledigt** — 84.050.623 EURUSD-Ticks geladen, 2023-01-09 → 2026-06-05 (1243 Tage); Tick-Validierung sauber (kein bid>ask, monoton, kein doppelter `time_msc`); 252.264 aggregierte M5-Bars gespeichert |
| ~~O5b~~ | ~~Konsistenzcheck aggregiert vs. nativ M5 ausstehend~~ | **erledigt** — Überlappungsfenster 2025-01-29 → 2026-06-05, **100.000 Bars verglichen** (vollständiger Inner-Join, kein Vergleichs-Cap — alle verfügbaren nativen Bars einbezogen; der native M5-Datensatz selbst ist auf ~100k Bars durch Terminal-/Abruf-Limit begrenzt, s. §8a); open/high/low: max_diff=0,000000, mean=0,000000, kein Flag; close: max_diff=0,000020 (~0,2 Pip), mean=0,000000, kein Flag; 5-Pip-Schwelle weit unterschritten, kein systematischer Drift; **Einschätzung: aggregierte M5 == native M5 (Bid) im Rahmen der Toleranz**. 3-Jahres-Abdeckung nicht betroffen — sie kommt aus Ticks (kein Bar-Cap, ab 2023-01-09) |
| O6 | Guardrail-Schicht (`risk-execution`-Subagent): alle §3-Guardrails fehlen noch | Hoch |
| O7 | Kein Rate-Limiting auf der Bridge (über Tailscale akzeptabel, dokumentiert) | Niedrig |
| ~~O8~~ | `validate_ticks()` meldete ~48 Mio. False-Positive-Duplikate (Sekundenauflösung von `time`; mehrere Ticks/Sekunde ist normales EURUSD-Verhalten) | **erledigt** — Validator nutzt jetzt `time_msc` (ms-Präzision) als Dedup- und Monotonie-Key, konsistent mit `save_ticks()`. Regressionstest: `test_validate_ticks_same_second_different_msc_no_false_positive` |

---

## 8a. Datentiefe — gemessen 2026-06-07

`scripts/measure_history_depth.py` ist fertig (Algorithmus: binäre Suche).

**Methode:** Rückwärts-Suche in 180-Tages-Schritten bis zur ersten leeren Antwort,
dann binäre Suche (Präzision ±14 Tage). Jede Probe fragt ein 30-Tages-Fenster ab
(OHLC) bzw. 7-Tages-Fenster (Ticks) — breit genug, um Wochenend- und Feiertagslücken
nicht mit dem Datenende zu verwechseln. Kein vollständiger Download der Historie.

Messung durchgeführt am 2026-06-07 über das Tailscale-Netz gegen die Live-Bridge
(RoboForex Demo, EURUSD):

| Timeframe | Älteste Kerze | Tiefe (Tage) | ≥ 3 Jahre? |
|---|---|---|---|
| H4 | ≤ 2016-06-07 (Suchtiefe-Cap > 10 J) | > 3652 | **ja** |
| H1 | ≤ 2016-06-07 (Suchtiefe-Cap > 10 J) | > 3652 | **ja** |
| M15 | 2022-05-26 | ~1472 | **ja** |
| M5 (nativ) | 2025-01-29 | ~493 | **nein** (nativ) — ✅ via Tick→M5 |
| Ticks | 2023-01-09 | ~1244 | **ja** |

**M5-Abdeckung via Tick→M5-Aggregation (O5/O5b erledigt):**
Nativ-M5 reicht nur ~493 Tage zurück. **Ursache: Terminal-/Abruf-Limit (~100k Bars).**
Probe 2026-06-08 — Anfrage M5 für 1246 Tage (2023-01-09 → heute, theoretisch ~256k Bars)
lieferte 100.117 Bars ab 2025-01-29 — die älteste Bar bleibt unabhängig vom Anfragefenster
konstant, ein eindeutiges Cap-Muster. Gleiches Muster bei M15: 100k / 96 Bars/Tag × 7/5
≈ 1459 Tage → älteste Kerze 2022-05-26 (~1471 Tage). `terminal_info().maxbars` ist über die
Bridge nicht auslesbar (kein Endpunkt ohne Bridge-Umbau); ein höheres TERMINAL_MAXBARS
würde das native Validierungsfenster verlängern, ist für die 3-Jahres-Abdeckung aber
nicht erforderlich.

**3-Jahres-Abdeckung nicht betroffen** — sie kommt aus Broker-Ticks (kein Bar-Cap):
- 84.050.623 Ticks geladen, 2023-01-09 → 2026-06-05
- 252.264 aggregierte M5-Bars, selber Zeitraum
- Konsistenzcheck gegen native M5 im Fenster 2025-01-29 → 2026-06-05 bestanden:
  100.000 verglichene Bars (vollständiger Inner-Join, kein Vergleichs-Cap),
  max Abweichung close 0,000020 (~0,2 Pip), kein systematischer Drift, kein geflaggtes Feld.

Reicht die Tiefe für 3-Jahres-Backtest nicht aus (< 1095 Tage): nur berichten,
externe Quelle NICHT ohne Supervisor-Freigabe einführen (SCOPE §5).

---

## 9. SCOPE.md §9 — Definition of Done Sprint 1

| Punkt | Status |
|---|---|
| MT5-Bridge läuft, authentifiziert, über Tailscale erreichbar | **Verifiziert** — `/health` 200, `is_demo: true` über Tailscale (O4 erledigt) |
| Endpunkte `/account`, `/positions`, `/ticks`, `/order` (hinter §3-Guardrails) | Implementiert und getestet; `/order` ist Stub bis Guardrails freigegeben |
| Heartbeat/Reconnect Mac-Client ↔ Bridge | Implementiert; Verbindung live verifiziert (O4 erledigt) |
| Datenpipeline H4/H1/M15/M5 + Ticks | **Live verifiziert** — 84 Mio. Ticks geladen, M5-Konsistenzcheck bestanden (§8a, O5/O5b erledigt) |
| `risk-execution`-Guardrails als testbare Schicht | **Offen (O6)** |
| AUDIT.md erzeugt und vom Supervisor freigegeben | Dieses Dokument — **Freigabe ausstehend** |
