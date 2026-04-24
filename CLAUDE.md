# CLAUDE.md — ha-door-lock

## Projekt

HA Package für automatisches Sperren von SwitchBot Lock Pro/Ultra Schlössern mit Retry-Logik, Pro-Lock-Modi (via input_select), individuellen Sperrfenstern (Start+Ende), Pause-Funktion und Push-Benachrichtigungen. Status: **experimentell** (aktiv in Erprobung).

## Wichtige Dokumente

- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) — Komponenten, Datenfluss, Entity-Schema, Neuer-Lock-Checkliste
- [docs/DECISIONS.md](docs/DECISIONS.md) — Design-Entscheidungen mit Begründung
- [README.md](README.md) — Installation und Bedienung für den Endnutzer

Diese Datei (CLAUDE.md) ist der Einstiegspunkt für Claude-Sessions und ergänzt die anderen Docs um session-relevante Hinweise.

## Voraussetzungen

- Home Assistant **2024.10+** (Sections-Layout im Dashboard)

## Aktueller Stand

- `packages/door_lock_auto.yaml` — v3 öffentliche Kern-Logik (Template-Sensoren, Scripts, Automations OHNE konkrete Lock-IDs). Enthält: `door_lock_scheduled_central`, `door_lock_config_sync`, `door_lock_door_open_warning`.
- `packages/door_lock_auto_local.yaml.example` + `_local.yaml` (gitignored) — die drei per-Lock-Trigger-Automations: `door_lock_scheduled_individual`, `door_lock_pause_expired`, `door_lock_unlocked_during_locktime`.
- `packages/door_lock_helpers.yaml.example` + `_local.yaml` (gitignored) — Per-Lock-Helper als YAML-Package.
- `dashboards/door_locks.yaml` — Sections-Layout, Mode-Dropdown + Start/Ende pro Lock.
- `README.md` — fertig.
- GitHub: https://github.com/Badspawn1/ha-door-lock (Branch: `master`).

**Regel**: keine echten Entity-IDs in den öffentlichen Dateien. Platzhalter: `lock.lock_pro_xxxx`, `lock.lock_ultra_yyyy`. Beides `_local.yaml` ist in `.gitignore`.

## Architektur v3

### Quelle der Wahrheit: `sensor.door_locks_active_config`
Ein Template-Sensor dessen `locks`-Attribut pro Lock einen Dict-Eintrag enthält:
```
{mode, effective_start, effective_end, in_locktime, warning_minutes,
 paused_until, paused, active, friendly_name}
```
Scripts und Automations lesen von hier — keine duplizierte Template-Logik. `mode` kombiniert Master-Override + Per-Lock-`input_select`. `in_locktime` wird pro Lock berechnet (inklusive Mitternachts-Übergang).

### Modus pro Lock via input_select
- `input_select.autolock_<slug>_mode` mit Optionen `off / central / individual`
- Dashboard-bedienbar per Dropdown — keine Labels mehr
- `off` = Lock ignoriert
- `central` = folgt zentralem Fenster (`door_lock_schedule_start/_end`)
- `individual` = eigenes Fenster via `autolock_<slug>_start/_end`
- Master-Override (`input_select.autolock_master_mode`): "Je nach Lock" (Default), "Alle aus", "Alle zentral", "Alle individuell"

### Sperrfenster-Semantik (Variante A)
- `_start` = Zeitpunkt, an dem der Lock zuschnappt (Trigger)
- `_end` = Ende des Relock-Schutzfensters (kein Trigger, nur Grenzwert)
- Zwischen start und end: `door_lock_unlocked_during_locktime` triggert Auto-Relock bei manuellem Entriegeln
- Außerhalb des Fensters: manuelles Entriegeln erlaubt, keine Reaktion

### Per-Lock-Helper-Konvention
Slug = Lock-Entity-ID ohne `lock.` Präfix (z.B. `lock_pro_xxxx`).
- `input_select.autolock_<slug>_mode` — Pflicht
- `input_datetime.autolock_<slug>_start` (has_time) — Pflicht wenn Mode individuell
- `input_datetime.autolock_<slug>_end` (has_time) — Pflicht wenn Mode individuell
- `input_number.autolock_<slug>_warning_minutes` — optional Override der Tür-offen-Warnzeit
- `input_datetime.autolock_<slug>_pause_until` (has_date + has_time) — optional für Pause bis X

Helper werden per `packages/door_lock_helpers_local.yaml` (gitignored) angelegt — Template dafür ist `packages/door_lock_helpers.yaml.example`.

### Scripts
- `door_lock_single_with_retry(lock_entity)` — Einzelschloss mit Retry + Door-Open-Check vor Sperren
- `door_lock_group(mode, [locks])` — orchestriert: mode ∈ {central, individual, all, list}
- `door_lock_pause(lock_entity, hours)` — setzt pause_until auf now()+hours
- `door_lock_pause_until_morning(lock_entity, morning_hour=8)` — setzt pause_until auf morgen X:00
- `door_lock_unpause(lock_entity)` — setzt pause_until in die Vergangenheit

### Automations
- `door_lock_scheduled_central` — zentrale Start-Zeit + `door_lock_force`-Event (mode=all)
- `door_lock_scheduled_individual` — Event-Trigger `autolock_individual_manual_trigger` + pro Lock eine `at:`-Zeile mit `_start`-Helper
- `door_lock_pause_expired` — pro Lock mit pause_until eine `at:`-Zeile → Info-Push bei Ablauf
- `door_lock_config_sync` — reagiert auf Anstieg von `sensor.autolock_config_issues` → Push
- `door_lock_unlocked_during_locktime` — Auto-Relock bei manuellem Entriegeln im eigenen Fenster. Pro Lock eigener State-Trigger (locked→unlocked); Condition prüft `active` + `in_locktime`. Sequenz: Info-Push → **60s Puffer** (BT-Türkontakt braucht Zeit bis "Tür offen" gemeldet ist) → `wait_template` auf Tür zu → 3s → `door_lock_single_with_retry`.
- `door_lock_door_open_warning` — filtert pro Lock auf `in_locktime` + `active`

### Failure-Notifications
`door_lock_single_with_retry` pusht nach erschöpften Retries selbst eine kritische Meldung pro Lock. `door_lock_group` aggregiert keine Failures mehr (nur noch Erfolgs-Meldung bei vollständigem Erfolg, opt-in via `input_boolean.door_lock_notify_success`).

### YAML-Nachpflege bei neuen Locks
Für jeden Lock mindestens zur Helper-Anlage:
1. `automation.door_lock_unlocked_during_locktime` → State-Trigger-Zeile (`entity_id: lock.<slug>`, `from: "locked"`, `to: "unlocked"`) — sonst wird Auto-Relock für diesen Lock nie ausgelöst.

Nur bei individuellen Locks zusätzlich:
2. `automation.door_lock_scheduled_individual` → eine Zeile unter `at:` mit `input_datetime.autolock_<slug>_start`

Nur falls Pause genutzt wird:
3. `automation.door_lock_pause_expired` → eine Zeile mit `input_datetime.autolock_<slug>_pause_until`

Grund: HA-Automation-Trigger sind statisch zur Load-Time. Kein Weg drum herum außer Polling (wurde abgelehnt für Zeit-Genauigkeit).

### Abgrenzung scheduled vs. unlocked_during_locktime
Keine Überschneidung: `scheduled_central` / `scheduled_individual` feuern auf **Zeit-Triggern** (Sperrfenster-Start) und rufen `door_lock_group` → `single_with_retry`. `unlocked_during_locktime` feuert auf **State-Triggern** (locked→unlocked eines bereits gesperrten Locks). Der Zustandsübergang beim planmäßigen Sperren ist unlocked→locked, triggert also nicht. Nur manuelles Entriegeln innerhalb des Fensters löst den Relock-Flow aus. Jeder Lock läuft im eigenen parallelen Automation-Run (`mode: parallel, max: 10`) — ein im Tür-offen-Wait hängender Lock blockiert die anderen nicht mehr (Vorgänger-Bug: globaler `door_locks_all_secured`-Transition-Trigger blieb "stuck off").

## Wichtige Implementierungsdetails

- Template-Sensoren: `name:`-Feld bestimmt die entity_id (slugifiziert) — hier konsequent englisch gesetzt damit Scripts stabil sind. `unique_id` ist nur für den Registry-Eintrag und ändert die entity_id NICHT nachträglich.
- Modus-Quelle: `input_select.autolock_<slug>_mode` (früher Labels). Dashboard-bedienbar, keine Settings-Navigation nötig.
- Dict-Merge in Jinja via `| combine({...})` Filter.
- Kritische Notifications cross-platform: iOS via `push.sound.critical: 1`, Android via `channel: alarm_stream_max` + `priority: high` + `ttl: 0` — beide Plattformen ignorieren jeweils fremde Keys.
- YAML-Reload reicht für Template-Sensoren aus (nicht erst bei Änderungen an Input-Helpers — dafür Neustart).

## Git-Hinweis

Repo wurde einmal neu erstellt um persönliche Entity-IDs aus der History zu entfernen. Keine echten Geräte-IDs oder Notify-Services in den Dateien — immer Platzhalter wie `lock.lock_pro_xxxx` verwenden.
