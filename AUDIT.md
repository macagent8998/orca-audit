# AUDIT.md — Sprint 1 · Bridge + Daten

> **Status 2026-06-02:** Bridge-Code fertig und getestet. Datenpipeline und
> Guardrail-Schicht noch ausstehend. Dieses Dokument wird nach Live-Verifikation
> und Fertigstellung der offenen Punkte dem Supervisor zur Freigabe vorgelegt.

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

### `orca/bridge_client.py` — Async HTTP-Client (Mac-Seite)

Async Context Manager `BridgeClient`; initialer Probe auf `__aenter__`; Background-Heartbeat-Task.

### `tests/`

`conftest.py` (MT5-Stub), `test_bridge_endpoints.py` (16 Tests), `test_bridge_client.py` (3 Tests).

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
| **Gesamt** | **24 / 24** | **✅** |

---

## 7. Look-ahead-Bias

Nicht zutreffend für Sprint 1. Die Bridge ist eine reine I/O-Schicht (MT5-Daten durchleiten),
keine Strategie-Logik. Look-ahead-Prüfung ist Pflicht-Gate vor dem Strategie-Merge (SCOPE §8).

---

## 8. Offene Punkte / Bekannte Risiken

| Nr. | Beschreibung | Priorität |
|---|---|---|
| ~~O1~~ | ~~"Bot must halt"-Verhalten nur strukturell geprüft~~ | **erledigt** — `test_heartbeat_halt_on_connection_loss` assertiert `is_connected=False` und ERROR-Log `"LOST"` |
| ~~O2~~ | ~~Token-Vergleich mit `==` statt `secrets.compare_digest`~~ | **erledigt** — `secrets.compare_digest` in `bridge/auth.py` |
| O3 | `ensure_connected()` macht nur einen Reconnect-Versuch; bei dauerhaftem MT5-Ausfall kein weiterer Retry (by design) | Info |
| O4 | Bridge-Live-Test (Mac ↔ Windows über Tailscale) noch nicht durchgeführt — Windows-Server-Setup ausstehend | Hoch |
| O5 | Datenpipeline (`data`-Subagent): H4/H1/M15/M5 OHLC + Ticks fehlt komplett | Hoch |
| O6 | Guardrail-Schicht (`risk-execution`-Subagent): alle §3-Guardrails fehlen noch | Hoch |
| O7 | Kein Rate-Limiting auf der Bridge (über Tailscale akzeptabel, dokumentiert) | Niedrig |

---

## 9. SCOPE.md §9 — Definition of Done Sprint 1

| Punkt | Status |
|---|---|
| MT5-Bridge läuft, authentifiziert, über Tailscale erreichbar | Code fertig; **Live-Deployment ausstehend (O4)** |
| Endpunkte `/account`, `/positions`, `/ticks`, `/order` (hinter §3-Guardrails) | Implementiert und getestet; `/order` ist Stub bis Guardrails freigegeben |
| Heartbeat/Reconnect Mac-Client ↔ Bridge | Implementiert; **live noch nicht verifiziert (O4)** |
| Datenpipeline H4/H1/M15/M5 + Ticks | **Offen (O5)** |
| `risk-execution`-Guardrails als testbare Schicht | **Offen (O6)** |
| AUDIT.md erzeugt und vom Supervisor freigegeben | Dieses Dokument — **Freigabe ausstehend** |
