# AUDIT.md — Sprint 1 · Bridge + Daten

> **Status 2026-06-09:** Sprint 1 **abgenommen** (Opus-Supervisor-Freigabe).
> Bridge live verifiziert (O4); Datenpipeline + M5-Aggregation live verifiziert
> (O5/O5b, Tick→M5-Konsistenz bestanden); risk-execution-Guardrails als hart
> erzwungene, fail-closed Schicht (O6, inkl. Missing-State-Fix); 178/178 Tests;
> Look-ahead per Bar-Schlusszeit abgesichert. `ORDERS_ENABLED=false` — kein
> Live-Handel.
>
> **Laufend (2026-06-10):** Strategy-Phase Step 1 — Primitive + Marktstruktur gebaut
> (`orca/strategy/`); 229/229 Tests; I1/I2/I3/I5 bestanden. Opus-Gate (Look-ahead)
> vor Merge — noch ausstehend.

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

### `orca/risk/` — Guardrail-Schicht (Mac-Seite)

| Datei | Inhalt |
|---|---|
| `orca/risk/models.py` | Datenklassen: `OrderRequest`, `AccountState`, `DayState`, `Decision`, `Action` (Enum) |
| `orca/risk/config.py` | SCOPE.md-Parser → `GuardrailConfig` (frozen dataclass); fail-closed, kein Default |
| `orca/risk/guardrails.py` | Reine Entscheidungsfunktion `evaluate_order()`; 10 Guardrails in Reihenfolge; I/O-frei, deterministisch |
| `orca/risk/state.py` | `DayState`-Persistenz (JSON, `state/` gitignored); `load_or_init_day_state()` / `save_day_state()`; Neustart-sicher |
| `orca/risk/kill_switch.py` | `check_kill_switch(path)` (rein); `do_flat(bridge_client)` (async, gegen Mock getestet; live-inaktiv bis Freigabe) |
| `orca/risk/execution.py` | `submit_order()`; `ORDERS_ENABLED`-Gate (Env-Var, default `false`); kein Live-Aufruf ohne Freigabe |
| `orca/risk/decision_log.py` | Strukturiertes JSON-Lines-Log jeder Entscheidung; lokal, gitignored |

### `tests/`

`conftest.py` (MT5-Stub), `test_bridge_endpoints.py` (20 Tests), `test_bridge_client.py` (4 Tests),
`test_bridge_history.py` (9 Tests), `test_data_accessor.py` (14 Tests — inkl. Look-ahead-Tests),
`test_data_aggregator.py` (15 Tests), `test_data_validator.py` (7 Tests), `test_data_store.py` (9 Tests),
`test_measure_history_depth.py` (15 Tests), `test_publish_audit_leak.py` (8 Tests),
`test_risk_config.py` (4 Tests), `test_risk_guardrails.py` (29 Tests),
`test_risk_state.py` (12 Tests), `test_risk_kill_switch.py` (7 Tests),
`test_risk_execution.py` (5 Tests — inkl. ORDERS_ENABLED-Gate),
`test_scope_watchdog.py` (19 Tests — Build-Zeit-Tooling, alle Modell-Calls gemockt).

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
| `test_risk_config.py` | `test_load_config_from_real_scope` | ✅ |
| | `test_config_missing_file_raises` | ✅ |
| | `test_config_corrupt_value_raises` | ✅ |
| | `test_config_is_frozen` | ✅ |
| `test_risk_guardrails.py` | `test_kill_switch_active_halts` | ✅ |
| | `test_kill_switch_inactive_continues` | ✅ |
| | `test_bridge_disconnected_halts` | ✅ |
| | `test_not_demo_halts` | ✅ |
| | `test_wrong_symbol_rejects` | ✅ |
| | `test_missing_sl_rejects` | ✅ |
| | `test_missing_tp_rejects` | ✅ |
| | `test_rr_exactly_min_accepts` | ✅ |
| | `test_rr_below_min_rejects` | ✅ |
| | `test_risk_per_trade_below_limit_accepts` | ✅ |
| | `test_risk_per_trade_above_limit_rejects` | ✅ |
| | `test_equity_zero_rejects` | ✅ |
| | `test_max_positions_zero_accepts` | ✅ |
| | `test_max_positions_reached_rejects` | ✅ |
| | `test_trades_below_daily_max_accepts` | ✅ |
| | `test_trades_at_daily_max_rejects` | ✅ |
| | `test_daily_loss_below_stop_accepts` | ✅ |
| | `test_daily_loss_at_stop_halts` | ✅ |
| | `test_circuit_breaker_active_halts` | ✅ |
| | `test_day_state_none_halts` | ✅ |
| | `test_fail_closed_all_good_accepts` | ✅ |
| | `test_determinism` | ✅ |
| `test_risk_state.py` | `test_fresh_init_no_file` (→ Bootstrap-Pfad) | ✅ |
| | `test_init_overwrites_existing_file` | ✅ |
| *(O6-Fix)* | `test_missing_file_raises_state_missing_error` | ✅ |
| | `test_missing_file_does_not_create_any_file` | ✅ |
| | `test_restart_same_day_loads` | ✅ |
| | `test_new_day_resets` | ✅ |
| | `test_rollover_requires_existing_file` | ✅ |
| | `test_corrupt_file_raises` | ✅ |
| | `test_empty_file_raises` | ✅ |
| | `test_missing_key_raises` | ✅ |
| | `test_trades_count_persists` | ✅ |
| | `test_halt_persists_across_restart` | ✅ |
| `test_risk_kill_switch.py` | `test_no_kill_file_returns_false` | ✅ |
| | `test_kill_file_exists_returns_true` | ✅ |
| | `test_kill_switch_check_order_priority` | ✅ |
| | `test_kill_switch_beats_high_risk_reject` | ✅ |
| | `test_do_flat_closes_all_positions` | ✅ |
| | `test_do_flat_no_positions_no_close` | ✅ |
| | `test_do_flat_get_positions_error_does_not_raise` | ✅ |
| `test_risk_guardrails.py` | `test_circuit_breaker_survives_restart` | ✅ |
| *(Nachreichung)* | `test_trade_counter_survives_restart` | ✅ |
| | `test_corrupt_state_evaluate_halts` | ✅ |
| *(O6-Fix)* | `test_missing_state_evaluate_halts` | ✅ |
| | `test_bridge_down_beats_all_reject_checks` | ✅ |
| | `test_rejected_order_does_not_mutate_day_state` | ✅ |
| | `test_risk_exactly_at_limit_accepts` | ✅ |
| `test_risk_execution.py` | `test_orders_enabled_false_is_default` | ✅ |
| | `test_reject_not_submitted` | ✅ |
| | `test_halt_not_submitted` | ✅ |
| | `test_accept_orders_disabled_not_submitted` | ✅ |
| | `test_accept_orders_enabled_calls_bridge` | ✅ |
| `test_scope_watchdog.py` | `test_new_test_file_is_not_drift` | ✅ |
| *(Scope-Watchdog)* | `test_guardrails_bugfix_is_not_drift` | ✅ |
| | `test_second_symbol_is_drift` | ✅ |
| | `test_llm_in_runtime_is_drift` | ✅ |
| | `test_orders_enabled_true_is_drift` | ✅ |
| | `test_missing_key_skips_gracefully` | ✅ |
| | `test_missing_key_does_not_crash` | ✅ |
| | `test_garbage_response_is_unclear` | ✅ |
| | `test_partial_json_missing_drift_key_is_unclear` | ✅ |
| | `test_model_exception_is_unclear` | ✅ |
| | `test_api_key_not_in_json_report` | ✅ |
| | `test_api_key_not_in_error_message` | ✅ |
| | `test_audit_only_diff_skips_model` | ✅ |
| | `test_removes_audit_hunk` | ✅ |
| | `test_empty_after_filter_if_only_audit` | ✅ |
| | `test_non_audit_diff_unchanged` | ✅ |
| | `test_parses_clean_json` | ✅ |
| | `test_parses_json_in_markdown_fence` | ✅ |
| | `test_missing_drift_key_is_error` | ✅ |
| **Gesamt** | **178 / 178** | **✅** |

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
| ~~O6~~ | ~~Guardrail-Schicht: alle §3-Guardrails fehlen noch~~ | **erledigt** — `orca/risk/` implementiert; 10 Guardrails (Prüfreihenfolge a–j, erste Verletzung gewinnt); `evaluate_order()` deterministisch + I/O-frei; `DayState` Neustart-persistent; Kill-Switch + `do_flat()` gegen Mock getestet; `submit_order()` mit `ORDERS_ENABLED`-Gate (default `false`); Missing-State fail-closed (O6-Fix); 42+10+5 Tests (159/159 gesamt). Bekannte Einschränkungen: s. §8b |
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

## 8b. O6 — Guardrail-Schicht (`orca/risk/`)

### 10 Guardrails — Schwellen aus SCOPE.md §3

Alle Schwellen werden beim Start einmalig aus SCOPE.md geparst (`load_guardrail_config()`).
Kein hartkodierter Zahlenwert in `guardrails.py`. Erste Verletzung gewinnt (fail-closed).

| Check | Guardrail | Schwelle | Aktion | Grenzfall-Test |
|---|---|---|---|---|
| a | Kill-Switch | Datei `./KILL` vorhanden | HALT | `test_kill_switch_active_halts` |
| b | Bridge-Heartbeat | `is_connected=False` | HALT | `test_bridge_disconnected_halts` |
| c | Demo-Flag | `is_demo=False` | HALT | `test_not_demo_halts` |
| d | Symbol-Whitelist | Symbol ∉ `["EURUSD"]` | REJECT | `test_wrong_symbol_rejects` |
| e | Pflicht-SL/TP | SL oder TP = `None` | REJECT | `test_missing_sl_rejects`, `test_missing_tp_rejects` |
| f | Mindest-RR | RR < 2,0 | REJECT | `test_rr_exactly_min_accepts` (2,0 → ACCEPT), `test_rr_below_min_rejects` (1,9 → REJECT) |
| g | Max. Risiko/Trade | `risk_frac` > 0,5 % | REJECT | `test_risk_per_trade_below_limit_accepts` (0,49 % → ACCEPT), `test_risk_per_trade_above_limit_rejects` (0,51 % → REJECT) |
| h | Max. gleichzeitige Positionen | offene Positionen ≥ 1 | REJECT | `test_max_positions_reached_rejects` |
| i | Max. Trades / Tag | Trades heute ≥ 3 | REJECT | `test_trades_at_daily_max_rejects` |
| j | Tages-Verlust-Stopp | kumulierter realisierter Verlust ≤ −1,5 % der SOD-Equity | HALT | `test_daily_loss_at_stop_halts` (−1,5 % → HALT), `test_daily_loss_below_stop_accepts` (−1,4 % → ACCEPT) |

Circuit Breaker aktiv (vorheriger HALT): `test_circuit_breaker_active_halts`.
Kein `DayState` (fail-closed): `test_day_state_none_halts`.

### Drei Definitionen (festgenagelt durch Docstrings + Tests)

**Risiko/Trade:** Equity-Anteil, der bei SL-Treffer verloren ginge, berechnet VOR Orderstellung.
Formel: `risk_usd = |entry − SL| × 100.000 × Lots`; `risk_frac = risk_usd / account.equity`.
Referenz: `account.equity`. Equity = 0 oder SL = entry → fail-closed REJECT.

**Trade heute:** Ein ACCEPTierter Order, der zu einer Positionseröffnung führte.
Abgelehnte Orders zählen NICHT. Quelle: `DayState.trades_opened_today` (persistenter Tageszähler).

**Tagesverlust:** Kumulierter realisierter (geschlossener) P&L seit 00:00 UTC;
Referenz = `DayState.start_of_day_equity`. Floating/unrealisierter P&L geht NICHT in den
Trigger ein — als bekannte Einschränkung dokumentiert (s. unten).

### DayState-Persistenz (`state/day_state.json`, gitignored)

Zwei getrennte Pfade (nach O6-Fix):

| Pfad | Funktion | Wann aufgerufen |
|---|---|---|
| Lad-Pfad | `load_day_state()` | Run-Loop |
| Bootstrap-Pfad | `init_day_state()` | Init-CLI / Erst-Einrichtung |

`load_day_state()` — Verhalten:

| Szenario | Verhalten |
|---|---|
| File fehlt | `StateMissingError` → Caller muss HALT setzen (fail-closed) |
| File korrupt/unlesbar | `StateCorruptError` → Caller muss HALT setzen (fail-closed) |
| File für heute | State laden, Zähler und Circuit Breaker bleiben erhalten |
| File für anderen Tag (Rollover) | Frischer State für heute; erlaubt weil Vortagesdatei vorhanden |

`init_day_state()` — immer frischer Null-State; darf NICHT vom Run-Loop aufgerufen werden.

Felder: `date_utc`, `start_of_day_equity`, `trades_opened_today`, `realized_pnl_today`, `halted_until`.
`halted_until = heute` → Circuit Breaker aktiv für den Rest des Tages; nächster Tag → Reset.

### O6-Fix: Missing-State fail-closed (2026-06-09)

**Befund:** `load_or_init_day_state()` behandelte ein fehlendes State-File als Reset
(frischer Null-State), nicht als Fehler. Ein mid-day gelöschtes File hätte
`trades_opened_today=0` und `halted_until=None` ergeben — Circuit Breaker und
Tageszähler wären lautlos zurückgesetzt worden.

**Fix:** Lade- und Bootstrap-Pfad getrennt. `load_day_state()` wirft `StateMissingError`
wenn kein File existiert — identisches fail-closed-Verhalten wie bei `StateCorruptError`.
Frische Nullwerte nur über explizites `init_day_state()` (Bootstrap-CLI), das der
Run-Loop nie selbst aufruft. Rollover (vorhandene Vortagesdatei, date < today) bleibt
im Lade-Pfad erlaubt. 5 neue Tests (4 in `test_risk_state.py`, 1 in `test_risk_guardrails.py`).

### Bekannte Einschränkungen

- **Floating-Loss nicht im Trigger:** Nur realisierter P&L löst den Tages-Verlust-Stopp aus.
  Ein großer offener Float-Verlust ist erst nach Positionsschluss im Zähler sichtbar.
  Accepted risk (Demo-Kontext; keine Positionen ohne SL erlaubt, Risiko dadurch begrenzt).
- **`do_flat()` live-inaktiv:** Gebaut und gegen gemockte Bridge getestet.
  Wird erst aktiv, wenn `ORDERS_ENABLED=true` gesetzt ist (Supervisor-Freigabe erforderlich).
- **`ORDERS_ENABLED=false`:** Kein echter Order-Call. `/order`-Stub bleibt 503.
  Aktivierung nur nach ausdrücklicher Freigabe (SCOPE §3 / CLAUDE.md §Berechtigungen).
- **Run-Loop-Integration (Strategie-/Execution-Phase):** MUSS `StateMissingError` und
  `StateCorruptError` fangen und HALTEN — nie frisch initialisieren. Wird am
  Strategie-/Execution-Gate per Test verifiziert (Bypass-Schutz für den Missing-State-Fix).

---

## 8c. Scope-Watchdog (`orca/tools/scope_watchdog.py`, SCOPE §8)

### Was es ist

Leichtgewichtiger Drift-Melder: liest ein git-Diff + `SCOPE.md` und fragt ein
**günstiges Modell** (`claude-haiku-4-5-20251001`, konfigurierbar), ob das Diff
Funktionalität/Abhängigkeiten/Symbole/Konten einführt, die über `SCOPE.md`
hinausgehen. Ausgabe: strukturiertes JSON + Exit-Code + menschenlesbare Zeile.

### Was es ausdrücklich NICHT ist

- **Kein Gate, kein Veto.** Blockiert nichts, committet nichts, schreibt nichts.
- **Kein Ersatz für das Opus-Gate.** Opus entscheidet über Freigaben; der Watchdog meldet nur.
- **Nie im Trading-Loop.** Reines Bauzeit-/CI-Tooling; nicht von `orca.risk`, `orca.strategy`
  oder `orca.data` importiert.

### Wie er läuft

```bash
python scripts/scope_watchdog.py                    # letzter Commit
python scripts/scope_watchdog.py --range HEAD~3..HEAD
python scripts/scope_watchdog.py --diff file.diff
```

| Exit-Code | Bedeutung |
|---|---|
| 0 | OK — kein Drift erkannt |
| 1 | Scope-Drift erkannt — Mensch + Opus müssen prüfen |
| 2 | Unklar — kein Key / Modell nicht erreichbar / Antwort unparsebar |

Fail-safe: nie still grün. Exit 2 bei jedem Zweifel. AUDIT.md-Hunks werden
vor dem API-Call herausgefiltert (keine False-Positives durch Doku-Commits).
API-Key aus `ANTHROPIC_API_KEY`-Env-Var; nie geloggt, nie im Report.
Optionaler lokaler Report: `reports/scope_watchdog_last.json` (gitignored).

### Modell und Kosten

Default: `claude-haiku-4-5-20251001` — günstigstes verfügbares Modell (SCOPE §7).
Konfigurierbar via `--model`. Max. 512 Output-Tokens pro Aufruf.
Kein echtes Modell in den Tests — ausschließlich gemockte Clients.

### Tests (`tests/test_scope_watchdog.py`, 19 Tests)

| Klasse | Was getestet wird |
|---|---|
| `TestInScopeDiff` | Neuer Test / Bugfix → drift=false, Exit 0 |
| `TestOutOfScopeDiff` | GBPUSD / LLM im Runtime / `ORDERS_ENABLED=true` → drift=true, Exit 1 |
| `TestNoApiKey` | Kein Key → skip, Exit 2, kein Crash |
| `TestUnparseableResponse` | Garbage / fehlendes Feld / Exception → Exit 2 |
| `TestSecretHandling` | API-Key nie im JSON-Report oder Fehlermeldung |
| `TestAuditMdFilter` | AUDIT.md-only-Diff → kein Modell-Call, Exit 0 |
| `TestFilterDiff` | Unit-Tests `_filter_diff()` |
| `TestParseResponse` | Unit-Tests `_parse_response()` inkl. Markdown-Fences |

---

## 8d. Strategy-Phase Step 1 — Primitive + Marktstruktur (2026-06-10)

### Modul-Aufteilung (`orca/strategy/`)

| Datei | Inhalt |
|---|---|
| `primitives.py` | Konstanten (`PIP`, `SWING_K`, `EQ_TOL`, `OB_LOOKBACK`, `FVG_MIN`); `slice_ohlc_as_of()` (bar_duration keyword-only required); `validate_ohlc()` |
| `structure.py` | `SwingPoint`, `StructureEvent`, `MarketStructure`; `find_swings()` (K=2 Fraktal, +K-Lag erzwungen); `compute_market_structure()` (CHoCH/BOS Zustandsmaschine) |
| `liquidity.py` | `LiquidityLevel`, `EqualCluster`, `Sweep`; `find_liquidity_levels()`, `find_equal_clusters()` (EQ_TOL=1 pip), `detect_sweeps()` (Wick-basiert + Close-Return) |
| `zones.py` | `OrderBlock`, `FairValueGap`, `DealingRange`; `find_order_blocks()` (letzte Gegenkerze in OB_LOOKBACK), `find_fvg()` (3-Bar-Gap, ≥ FVG_MIN), `apply_mitigation()` (FVG 50 % Fill, OB First Entry), `find_dealing_range()` |
| `__init__.py` | Re-Export aller öffentlichen Symbole |

**Begründung der Aufteilung:** `primitives` hat keine internen Abhängigkeiten → immer zuerst importierbar. `structure` baut auf `primitives`. `liquidity` braucht nur `SwingPoint` aus `structure`. `zones` braucht `MarketStructure` + `SwingPoint`. Keine zirkulären Importe.

### Look-ahead-Absicherung

**+K-Bestätigungs-Lag (I2):** `find_swings()` iteriert nur über `range(K, n-K)`. Ein Swing bei Index i erfordert die rechten Nachbarn `i+1..i+K` — die sind nur vorhanden, wenn `i+K ≤ n-1`, also `n ≥ i+K+1`. Das erzwingt der Loop unabhängig vom Caller-Slicing.

**`slice_ohlc_as_of()` (I1):** `bar_duration` ist keyword-only ohne Default. Weglassen → `TypeError` via Python-Sprachgarantie. Die Funktion filtert auf `close_time = open_time + bar_duration ≤ t`, nicht auf `open_time ≤ t`.

**Deterministisch (I3):** Alle Datenklassen `frozen=True`; alle Funktionen pure (kein I/O, kein Zustand). Gleiche Eingabe → gleiche Ausgabe per Definition.

**Master-Look-ahead (I5):** `slice_ohlc_as_of(df, t, ...)` liefert für identisches `df[:t+1]` immer den gleichen Slice — Mutationen in `df[t+1:]` haben keinen Effekt. Alle Primitive sind reine Funktionen über den gelieferten Slice. Getestet mit dramatischen Preisverfälschungen nach t und mit zusätzlich angehängten Bars.

### Tests (`tests/test_strategy_step1.py`, 51 Tests)

| Klasse | Was getestet wird |
|---|---|
| `TestBarDurationRequired` (I1) | TypeError ohne bar_duration; positionaler Aufruf ebenfalls abgefangen; String-Timeframe; unbekannter TF |
| `TestSwingLag` (I2) | Self-Check a: Slice ≤ i+K-1 → kein Swing; Self-Check b: Slice = i+K → Swing da; SL-Lag analog |
| `TestSwingBasic` | Zu kurze Serie; korrekte Anzahl SH/SL; Float-Preis; letzten K Bars nie zurückgegeben |
| `TestMarketStructure` | Neutral (kein Break); bullish CHoCH; bearish CHoCH; events-Tuple vollständig; fehlende Spalte → ValueError |
| `TestLiquidity` | SH→buy_side, SL→sell_side; Equal-Cluster innerhalb / außerhalb Toleranz; Sell-side-Sweep detektiert; kein Sweep ohne Close-Return; Level kann sich nicht selbst sweepen |
| `TestFVG` | Bullish/Bearish erkannt; unter FVG_MIN verworfen; kein FVG bei Überlappung |
| `TestMitigation` | FVG mitigated bei 50 %-Fill; nicht mitigated wenn Preis fern bleibt; OB bei First Entry mitigated; already-mitigated bleibt |
| `TestDealingRange` | Basis-Range; Premium; Discount; kein Result bei fehlenden Swings; neueste Swings verwendet |
| `TestOrderBlocks` | Bullish OB gefunden; origin_bar < break_bar; kein Event → keine OBs |
| `TestSliceOhlcAsOf` | Nicht-geschlossene Bars ausgeschlossen; Reset-Index; String-Timeframe |
| `TestDeterminism` (I3) | Swings, MarketStructure, FVG, DealingRange — gleiche Eingabe → gleiche Ausgabe |
| `TestLookAhead` (I5) | Mutation nach t; Append nach t; drei Timeframes (M5/M15/H4) |

### Bekannte Einschränkungen / Vorbehalte

- `compute_market_structure()` verfolgt nur den **letzten** bestätigten Swing pro Typ als aktives Referenzniveau. Wenn zwei Swings dieselbe Bar-Confirmation haben, gewinnt der mit dem höheren Index.
- `find_dealing_range()` verwendet die **aktuellsten** Swings nach Index, nicht zwingend die strukturell relevantesten. Für Step 2 (Signale) kann eine kontextbasierte Filterung notwendig sein.
- Noch kein Integration-Test gegen echte EURUSD-Historiendaten; solcher Test ist für den Opus-Gate-Review (Look-ahead-Prüfung vor Merge) vorgesehen.

**Nächster Step:** Strategy Step 2 — Signalregelkaskade H4→H1→M15→M5 (erst nach Opus-Gate-Review dieses Steps).

---

## 9. SCOPE.md §9 — Definition of Done Sprint 1

| Punkt | Status |
|---|---|
| MT5-Bridge läuft, authentifiziert, über Tailscale erreichbar | **Verifiziert** — `/health` 200, `is_demo: true` über Tailscale (O4 erledigt) |
| Endpunkte `/account`, `/positions`, `/ticks`, `/order` (hinter §3-Guardrails) | Implementiert und getestet; `/order` ist Stub bis Guardrails freigegeben |
| Heartbeat/Reconnect Mac-Client ↔ Bridge | Implementiert; Verbindung live verifiziert (O4 erledigt) |
| Datenpipeline H4/H1/M15/M5 + Ticks | **Live verifiziert** — 84 Mio. Ticks geladen, M5-Konsistenzcheck bestanden (§8a, O5/O5b erledigt) |
| `risk-execution`-Guardrails als testbare Schicht | **Implementiert** — `orca/risk/`; 10 Guardrails; `evaluate_order()` deterministisch; `ORDERS_ENABLED=false` (kein Live-Call); Missing-State fail-closed (O6-Fix); 159 Tests (O6 + O6-Fix erledigt, s. §8b) |
| AUDIT.md erzeugt und vom Supervisor freigegeben | **Freigegeben** — Opus-Supervisor 2026-06-09 (Meilenstein Bridge + Daten + Guardrails) |
