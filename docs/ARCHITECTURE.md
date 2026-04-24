# Architektur

Kompakte Referenz der Komponenten, Datenflüsse und Entity-Konventionen. Für die **Warum**-Ebene der Design-Entscheidungen siehe [DECISIONS.md](DECISIONS.md).

## Datei-Struktur

```
packages/
  door_lock_auto.yaml                  # öffentliche Kern-Logik, keine Lock-IDs
  door_lock_auto_local.yaml.example    # Template für Per-Lock-Automations
  door_lock_auto_local.yaml            # gitignored — deine echten IDs
  door_lock_helpers.yaml.example       # Template für Per-Lock-Helper
  door_lock_helpers_local.yaml         # gitignored — deine echten Helper
dashboards/                            # gitignored — lokales Dashboard
docs/
  ARCHITECTURE.md                      # dieses Dokument
  DECISIONS.md                         # Entscheidungslog
```

Alle `_local`-Dateien sind in `.gitignore`. Öffentliche Dateien enthalten nur Platzhalter wie `lock.lock_pro_xxxx`.

## configuration.yaml

```yaml
homeassistant:
  packages:
    door_lock_auto:       !include packages/door_lock_auto.yaml
    door_lock_helpers:    !include packages/door_lock_helpers_local.yaml
    door_lock_auto_local: !include packages/door_lock_auto_local.yaml
```

Alle drei Einträge sind Pflicht.

## Entity-Schema

**Lock ↔ Türkontakt**: 1:1-Paar, Naming-Konvention:

```
lock.<slug>    ↔    binary_sensor.<slug>
```

Beispiel: `lock.lock_pro_xxxx` ↔ `binary_sensor.lock_pro_xxxx`. Der Türkontakt ist physisch in SwitchBot Lock Pro/Ultra integriert; die SwitchBot-Integration legt den `binary_sensor` automatisch an.

**Slug** = Lock-Entity-ID ohne `lock.`-Präfix. Er verknüpft Lock, Türkontakt und alle Helper.

## Per-Lock-Helper

Jedes Lock hat bis zu fünf Helper (Template: `door_lock_helpers.yaml.example`):

| Helper | Pflicht? | Zweck |
|---|---|---|
| `input_select.autolock_<slug>_mode` (off/central/individual) | **Pflicht** | Modus pro Lock, Dashboard-bedienbar |
| `input_datetime.autolock_<slug>_start` (has_time) | Pflicht bei `individual` | Sperrfenster-Start (Trigger) |
| `input_datetime.autolock_<slug>_end` (has_time) | Pflicht bei `individual` | Sperrfenster-Ende (Grenzwert, kein Trigger) |
| `input_number.autolock_<slug>_warning_minutes` | optional | Individuelle Tür-offen-Warnzeit |
| `input_datetime.autolock_<slug>_pause_until` (has_date + has_time) | optional | Pause bis X |

## Globale Helper

| Entity | Zweck |
|---|---|
| `input_boolean.door_lock_auto_enabled` | Master-Schalter |
| `input_select.autolock_master_mode` | Override: "Je nach Lock" / "Alle aus" / "Alle zentral" / "Alle individuell" |
| `input_datetime.door_lock_schedule_start` / `_end` | Zentrales Sperrfenster |
| `input_text.door_lock_entities` | Lock-IDs komma-getrennt (Quelle der Config-Liste) |
| `input_text.door_lock_notify_service` | z.B. `notify.mobile_app_xyz` |
| `input_number.door_lock_retry_count` / `_delay` / `_verify_timeout` | Retry-Parameter |
| `input_number.door_lock_open_warning_minutes` | Default-Warnzeit |
| `input_boolean.door_lock_notify_success` | Erfolgs-Push ja/nein |

## Quelle der Wahrheit: `sensor.door_locks_active_config`

Zentraler Template-Sensor. Attribut `locks` = Dict pro Lock mit allem, was Scripts und Automations brauchen:

```
{
  "lock.lock_pro_xxxx": {
    "mode": "central" | "individual" | "off",
    "effective_start": "22:00",
    "effective_end": "06:00",
    "in_locktime": true | false,
    "warning_minutes": 15,
    "paused_until": "2026-04-25 08:00:00",
    "paused": false,
    "active": true,
    "friendly_name": "..."
  },
  ...
}
```

- `mode` kombiniert Master-Override + per-Lock-`input_select`
- `in_locktime` wird pro Lock berechnet (inklusive Mitternachts-Übergang)
- `active` = `master_on AND mode != 'off' AND NOT paused`

Scripts und Automations lesen **nur** hier, keine duplizierte Template-Logik.

## Weitere Sensoren

| Sensor | Zweck |
|---|---|
| `sensor.autolock_config_issues` | Zählt fehlende/kaputte Helper; `issues`-Attribut listet sie auf |
| `sensor.door_locks_open_doors` | Offene Türen (Namen-Liste) |
| `sensor.door_locks_unlocked_list` | Entriegelte Locks (Namen-Liste) |
| `binary_sensor.door_locks_all_secured` | `on` = alle aktiven Locks gesperrt |

## Scripts

| Script | Zweck |
|---|---|
| `door_lock_single_with_retry(lock_entity)` | Einzelschloss sperren mit Retry-Loop, Door-Open-Check vorne, kritischer Push nach erschöpften Retries |
| `door_lock_group(mode, [locks])` | Orchestrator, `mode ∈ {central, individual, all, list}` — startet pro Ziel-Lock einen parallelen `single_with_retry`-Run |
| `door_lock_pause(lock_entity, hours)` | setzt `pause_until` auf `now() + hours` |
| `door_lock_pause_until_morning(lock_entity, morning_hour=8)` | setzt `pause_until` auf morgen X:00 |
| `door_lock_unpause(lock_entity)` | setzt `pause_until` in die Vergangenheit |

## Automations (öffentlicher Teil: `door_lock_auto.yaml`)

| Automation | Trigger | Aktion |
|---|---|---|
| `door_lock_scheduled_central` | Zeit: `door_lock_schedule_start` oder Event `door_lock_force` | `door_lock_group(mode=central)` bzw. `mode=all` |
| `door_lock_config_sync` | State: `sensor.autolock_config_issues` steigt | Push mit Issue-Liste |
| `door_lock_door_open_warning` | State: `sensor.door_locks_open_doors` ändert | pro Lock im eigenen Fenster Warnung nach `warning_minutes` |

## Automations (lokal: `door_lock_auto_local.yaml`)

Diese drei brauchen konkrete Lock-IDs, leben daher in der lokalen Datei:

| Automation | Trigger | Aktion |
|---|---|---|
| `door_lock_scheduled_individual` | Pro `individual`-Lock: Zeit = `_start`-Helper | `door_lock_group(mode=list, [lock])` |
| `door_lock_pause_expired` | Pro Lock mit `pause_until`: Zeit = `_pause_until` | Info-Push |
| `door_lock_unlocked_during_locktime` | Pro Lock: State `lock.<slug>` locked→unlocked | Info-Push → **60s Puffer** → warte auf Tür-zu → Relock via `single_with_retry` |

## Sperrfenster-Semantik (Variante A)

- **`_start`** = Zeitpunkt, an dem der Lock zuschnappt (Trigger für `scheduled_*`).
- **`_end`** = Ende des Relock-Schutzfensters. **Kein Trigger**, nur Grenzwert.
- Zwischen `_start` und `_end`: jede Entriegelung löst via `unlocked_during_locktime` den Relock-Flow aus.
- Außerhalb: manuelles Entriegeln bleibt unberührt.

Midnight-Crossing: `effective_start > effective_end` → `in_locktime = (now >= start OR now < end)`.

## Datenfluss: Entriegelt während Sperrzeit

```
Lock geht locked → unlocked
  │
  ▼
Per-Lock state-Trigger feuert (nur dieses Lock)
  │
  ▼
Condition: active=true AND in_locktime=true?  ── nein → Stop
  │ ja
  ▼
Info-Push "Schloss entriegelt"
  │
  ▼
60 s Puffer   ◄── BT-Türkontakt braucht Zeit, bis "Tür offen" gemeldet wird
  │
  ▼
wait_template: Tür zu? (timeout 60 min)
  │
  ▼
Tür zu UND Lock noch nicht gesperrt?
  │ ja
  ▼
3 s Delay → script.door_lock_single_with_retry
  │
  ▼
Retry-Loop → bei Erfolg: still; bei Failure: kritischer Push pro Lock
```

## Datenfluss: Planmäßiges Sperren

```
Zeit = _start-Helper  ──►  scheduled_central / scheduled_individual
  │
  ▼
door_lock_group (mode=central|individual|all|list)
  │
  ├─► Filter via sensor.door_locks_active_config (nur active Locks)
  │
  ▼
Pro Ziel-Lock: script.turn_on door_lock_single_with_retry (parallel)
  │
  ▼
single_with_retry:
  ├── Tür offen? → Warnung + wait (warning_minutes) → bei Timeout: kritische Push, Abbruch
  ├── Retry-Loop: lock.lock → verify → bei Fail: delay → nächster Versuch
  └── Nach letztem Retry immer noch nicht gesperrt: kritische Push
```

## Kritische Push-Notifications

Cross-platform via gemeinsamem Payload — jede Plattform ignoriert fremde Keys:

```yaml
data:
  push:                         # iOS
    sound:
      name: default
      critical: 1
      volume: 1.0
  channel: alarm_stream_max     # Android
  priority: high
  ttl: 0
```

## Failure-Notification-Modell

- `single_with_retry` pusht **pro Lock selbst** nach erschöpften Retries.
- `door_lock_group` aggregiert **keine** Failures mehr — nur noch eine opt-in Erfolgs-Meldung (via `input_boolean.door_lock_notify_success`), wenn alle Ziel-Locks gesperrt wurden.
- Vorteil: sofort sichtbar, welches Schloss betroffen ist.

## Neuer Lock — Checkliste

1. In `packages/door_lock_helpers_local.yaml` Helper-Block duplizieren, `<SLUG>` ersetzen.
2. In `packages/door_lock_auto_local.yaml` drei Trigger-Einträge:
   - **Pflicht**: `unlocked_during_locktime` state-Trigger (`lock.<slug>` locked→unlocked).
   - Nur bei `individual`: `scheduled_individual` time-Trigger (`_start`-Helper).
   - Nur bei Pause-Nutzung: `pause_expired` time-Trigger (`_pause_until`-Helper).
3. `input_text.door_lock_entities` um die neue Lock-ID erweitern.
4. Dashboard-Sektion für den neuen Lock anlegen.
5. HA-Neustart (oder nur Reload Automations, wenn nur Trigger-Zeilen geändert wurden).
