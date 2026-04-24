# SwitchBot Türschloss - Auto-Lock

![Status](https://img.shields.io/badge/status-experimental-orange)
![Home Assistant](https://img.shields.io/badge/Home%20Assistant-2024.10%2B-blue)

> **⚠ Experimentell** — dieses Paket befindet sich aktiv in der Erprobung. Logik, Entity-Schema und Konfiguration können sich ohne Vorwarnung ändern. Vor produktivem Einsatz selbst testen. Feedback und Issues willkommen.

Home Assistant Package zum automatischen Sperren von [SwitchBot Lock](https://www.switch-bot.com/products/switchbot-lock-pro) Türschlössern (Lock Pro, Lock Ultra) mit Retry-Logik, Auto-Relock, Pro-Lock-Modi, Pause-Funktion und Benachrichtigungen.

## Dokumentation

- **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)** — Komponenten, Datenfluss, Entity-Schema
- **[docs/DECISIONS.md](docs/DECISIONS.md)** — Design-Entscheidungen mit Begründung

## Features

- **Sperrfenster-basiertes Auto-Lock** — Pro Lock ein Start/Ende-Fenster (z.B. 22:00–06:00). Zum Start wird gesperrt, während des Fensters wird jede Entriegelung per Auto-Relock zurückgesetzt.
- **Pro-Lock-Modi per Dashboard** — Jedes Schloss einzeln steuerbar via `input_select`: `off`, `central` oder `individual` (Dropdown auf dem Dashboard, keine Labels nötig).
- **Individuelle Sperrfenster pro Lock** — Eigenes Start/Ende pro Schloss (z.B. Kinderzimmer 20:00–07:00, Haustür 22:00–06:00).
- **Individuelle Tür-offen-Warnzeit pro Lock** — Eigene Warnzeit-Schwelle pro Schloss (optional, sonst Default).
- **Pause-Funktion pro Lock** — Schloss temporär deaktivieren bis Datum/Zeit X (Dashboard-Shortcuts: 2h, bis morgen, bis…).
- **Tür-offen-Check vor dem Sperren** — Offene Türen werden nicht blind gesperrt; Wartezeit + kritische Warnung, Abbruch wenn Tür offen bleibt.
- **Master-Override** — Global übersteuern: "Je nach Lock", "Alle aus", "Alle zentral", "Alle individuell".
- **Config-Sync-Check** — Automatische Push-Meldung bei fehlenden Helpern oder unavailable Locks.
- **Retry-Logik + Statusverifikation** — Konfigurierbare Versuche; prüft ob tatsächlich gesperrt wurde.
- **Push-Benachrichtigungen** — Kritische Warnungen bei Fehler (durchbrechen "Nicht stören"), Info-Push bei Entriegelung und Pause-Ende.
- **Manueller Trigger** — Per Event (`door_lock_force`) jederzeit auslösbar.

## Entity-Schema

Das Package leitet den Türkontaktsensor automatisch vom Lock-Entity ab (funktioniert für Lock Pro und Lock Ultra):

```
lock.lock_pro_xxxx    ->  binary_sensor.lock_pro_xxxx
lock.lock_ultra_xxxx  ->  binary_sensor.lock_ultra_xxxx
```

Es wird kein separates Mapping benötigt.

**Slug-Konvention:** Der `slug` eines Locks ist die Entity-ID ohne den `lock.`-Präfix. Beispiel:

```
lock.lock_pro_xxxx    ->  slug = lock_pro_xxxx
lock.lock_ultra_yyyy  ->  slug = lock_ultra_yyyy
```

Der Slug wird für die Benennung der Per-Lock-Helper verwendet (siehe unten).

## Voraussetzungen

- **Home Assistant 2024.10+** (Sections-Layout für das Dashboard)
- SwitchBot Lock Pro oder Lock Ultra mit integriertem Türkontakt
- SwitchBot Integration (via Bluetooth oder Cloud)
- HA Companion App (für Push-Benachrichtigungen)

## Installation

### 1. Dateien kopieren

Kopiere `packages/door_lock_auto.yaml` nach `/config/packages/` auf deinem Home Assistant Server.

```bash
# Per SCP:
scp packages/door_lock_auto.yaml root@homeassistant.local:/config/packages/

# Oder manuell per File Editor Add-on
```

Per-Lock-Helper liegen in `packages/door_lock_helpers_local.yaml` (gitignored, siehe **Per-Lock-Konfiguration (Helper)**).

### 2. configuration.yaml erweitern

```yaml
homeassistant:
  packages:
    door_lock_auto:       !include packages/door_lock_auto.yaml
    door_lock_helpers:    !include packages/door_lock_helpers_local.yaml
    door_lock_auto_local: !include packages/door_lock_auto_local.yaml
```

Alle drei Einträge sind Pflicht:

- `door_lock_auto.yaml` — öffentliche Kern-Logik (Template-Sensoren, Scripts, generische Automations). Keine Lock-IDs.
- `door_lock_helpers_local.yaml` — deine Per-Lock-Helper (gitignored). Template: `door_lock_helpers.yaml.example`.
- `door_lock_auto_local.yaml` — deine Per-Lock-Automations mit state- und Zeit-Triggern (gitignored). Template: `door_lock_auto_local.yaml.example`.

Die beiden `_local`-Dateien enthalten deine konkreten Entity-IDs und kommen nicht auf Github.

### 3. Home Assistant neu starten

**Settings > System > Restart** — ein YAML-Neuladen reicht nicht, da `input_*` Entities nur beim Systemstart geladen werden.

### 4. Schlösser konfigurieren

Nach dem Neustart in **Developer Tools > States** (oder direkt im Dashboard):

1. `input_text.door_lock_entities` — Lock-Entity-IDs komma-getrennt eintragen:
   ```
   lock.lock_pro_xxxx,lock.lock_pro_yyyy,lock.lock_pro_zzzz
   ```
2. `input_text.door_lock_notify_service` — Notify-Service setzen:
   ```
   notify.mobile_app_your_phone
   ```
3. `input_boolean.door_lock_auto_enabled` auf `on` setzen
4. Zentrales Sperrfenster: `input_datetime.door_lock_schedule_start` + `_schedule_end` (Default 22:00–06:00)
5. `input_select.autolock_master_mode` auf gewünschten Modus (Standard: "Je nach Lock")
6. Pro Lock den Modus wählen (siehe **Modi**) und — bei `individual` — Start/Ende setzen
7. Für individuelle Locks: YAML-Trigger ergänzen (siehe **YAML-Nachpflege**)

### 5. Testen

Manuell über **Developer Tools > Events**:

```
Event Type: door_lock_force
```

Dies sperrt alle aktiven konfigurierten Schlösser sofort (unabhängig vom Zeitplan, aber abhängig von Master-Modus und Per-Lock-Modus).

## Modi

Der Modus wird pro Lock über einen `input_select`-Helper gesteuert. Der Helper heißt `input_select.autolock_<slug>_mode` und kann direkt per Dashboard-Dropdown umgestellt werden.

| Wert | Verhalten |
|------|-----------|
| `off` | Lock wird komplett ignoriert |
| `central` | Lock folgt dem zentralen Sperrfenster (`door_lock_schedule_start` / `_end`) |
| `individual` | Lock nutzt eigenes Sperrfenster (`autolock_<slug>_start` / `_end`) |

Der Master-Modus (`input_select.autolock_master_mode`) kann Per-Lock-Modi global übersteuern — z.B. "Alle aus" als Kill-Switch. Siehe **Master-Override**.

## Per-Lock-Konfiguration (Helper)

Pro Lock gibt es bis zu fünf Helper. Alle folgen der Slug-Konvention (z.B. Slug `lock_pro_xxxx` für `lock.lock_pro_xxxx`).

| Helper-Entity | Typ | Pflicht? | Zweck |
|---------------|-----|----------|-------|
| `input_select.autolock_<slug>_mode` | options: off/central/individual | **Pflicht** | Modus pro Lock |
| `input_datetime.autolock_<slug>_start` | `has_time: true` | **Pflicht** bei Mode `individual` | Sperrfenster-Start |
| `input_datetime.autolock_<slug>_end` | `has_time: true` | **Pflicht** bei Mode `individual` | Sperrfenster-Ende |
| `input_number.autolock_<slug>_warning_minutes` | min: 5, max: 60 | Optional | Individuelle Tür-offen-Warnzeit |
| `input_datetime.autolock_<slug>_pause_until` | `has_date: true`, `has_time: true` | Optional | Pause-Funktion bis Zeitpunkt X |

### Helper anlegen

Zwei Wege — empfohlen ist **Variante A** (alle Helper in einem Rutsch via YAML).

**Variante A — Per YAML-Package (empfohlen):**

1. `packages/door_lock_helpers.yaml.example` als Vorlage nutzen
2. Kopie lokal als `packages/door_lock_helpers_local.yaml` anlegen (diese Datei ist in `.gitignore` — echte Entity-IDs bleiben privat)
3. Für jeden Lock den Helper-Block duplizieren und `<SLUG>` durch den tatsächlichen Slug ersetzen
4. In `configuration.yaml` unter `packages:` referenzieren (siehe **Installation Schritt 2**)
5. HA-Neustart → alle Helper sind sofort in **Settings > Helpers** sichtbar und im Dashboard bedienbar

**Variante B — Manuell via UI:**

1. **Settings > Devices & Services > Helpers**
2. **Create Helper** klicken
3. Typ auswählen (Dropdown / Date/Time / Number)
4. Namen exakt nach Konvention vergeben (`autolock_<slug>_<feld>`)

Der Config-Sync-Check (`sensor.autolock_config_issues`) meldet automatisch per Push, wenn ein Helper fehlt, oder ein Lock mit Modus `individual` kein `_start`/`_end` hat.

## Per-Lock-Automations

Home Assistant kann Automation-Trigger nicht dynamisch aus einer Liste von Helpern erzeugen. Deshalb leben die drei Automations, die konkrete Lock-Entity-IDs brauchen, in einer separaten **lokalen** Datei `packages/door_lock_auto_local.yaml` (gitignored). Das Template dafür ist `packages/door_lock_auto_local.yaml.example`.

**Vorgehen:**

1. `packages/door_lock_auto_local.yaml.example` als Vorlage nutzen
2. Kopie lokal als `packages/door_lock_auto_local.yaml` anlegen
3. Pro Lock die Trigger-Zeilen duplizieren und `<SLUG>` durch den tatsächlichen Slug ersetzen
4. In `configuration.yaml` einbinden (siehe **Installation Schritt 2**)
5. Reload: **Developer Tools > YAML > Reload Automations**

### Pflicht: `door_lock_unlocked_during_locktime`

Pro Lock eine State-Trigger-Zeile. Ohne diese Zeile wird der Auto-Relock beim manuellen Entriegeln für diesen Lock **nie** ausgelöst.

```yaml
triggers:
  - trigger: state
    entity_id: lock.lock_pro_xxxx
    from: "locked"
    to: "unlocked"
  - trigger: state
    entity_id: lock.lock_pro_yyyy
    from: "locked"
    to: "unlocked"
  # ... für jeden Lock eine Zeile
```

Die Automation läuft `mode: parallel, max: 10` — jeder Lock-Run ist isoliert. Ein im Tür-offen-Wait hängender Lock blockiert die anderen nicht.

Nach dem Entriegeln wartet die Automation **60 Sekunden** (Puffer für den BT-Türkontakt-Sensor, der die "Tür offen"-Meldung verzögert liefert), dann erst prüft sie, ob die Tür wieder zu ist, und sperrt erneut.

### Nur bei `individual`-Locks: `door_lock_scheduled_individual`

Zusätzlich eine `at:`-Zeile mit dem `_start`-Helper:

```yaml
triggers:
  - trigger: event
    event_type: autolock_individual_manual_trigger
    id: manual
  - trigger: time
    at: input_datetime.autolock_lock_pro_xxxx_start
    id: scheduled
  - trigger: time
    at: input_datetime.autolock_lock_pro_yyyy_start
    id: scheduled
```

Der Trigger feuert zum **Start** des Sperrfensters. Das Ende ist kein Trigger, sondern nur Grenzwert für den Auto-Relock-Schutz.

### Optional: `door_lock_pause_expired`

Nur für Locks, die `pause_until` nutzen:

```yaml
triggers:
  - trigger: time
    at: input_datetime.autolock_lock_pro_xxxx_pause_until
  - trigger: time
    at: input_datetime.autolock_lock_ultra_yyyy_pause_until
```

Nach der Anpassung Package neu laden (**Developer Tools > YAML > Reload Automations** reicht hier).

## Pause-Funktion

Die Pause-Funktion deaktiviert ein einzelnes Schloss temporär bis zu einem bestimmten Zeitpunkt. Danach wird der Auto-Lock automatisch wieder aktiv — inklusive Info-Push.

### Use-Case

- **Gästezimmer** — Gäste kommen spät nach Hause, Schloss soll für 2 Nächte nicht automatisch sperren
- **Renovierung/Handwerker** — Tür bleibt länger offen/frei
- **Urlaub** — Einzelne Locks deaktivieren ohne Master-Schalter zu touchen

### Setup

1. Helper `input_datetime.autolock_<slug>_pause_until` anlegen (mit `has_date: true` und `has_time: true`)
2. In `automation.door_lock_pause_expired` eine `at:`-Zeile ergänzen (siehe **YAML-Nachpflege**)

### Nutzung

Das Dashboard bietet pro Lock vier Pause-Buttons — der passende Shortcut ist meist schneller als den Helper direkt zu setzen.

| Button | Script | Wirkung |
|--------|--------|---------|
| **Pause 2h** | `script.door_lock_pause` mit `hours: 2` | Setzt `pause_until` auf `now() + 2h` |
| **Bis morgen** | `script.door_lock_pause_until_morning` mit `morning_hour: 8` | Setzt `pause_until` auf morgen 08:00 |
| **Bis...** | `action: more-info` auf `input_datetime.autolock_<slug>_pause_until` | Öffnet Datetime-Picker für freie Auswahl |
| **Aufheben** | `script.door_lock_unpause` | Setzt `pause_until` in die Vergangenheit |

**Manuell (abseits des Dashboards):**

Der Helper `input_datetime.autolock_<slug>_pause_until` ist die Source of Truth. Du kannst ihn direkt setzen:

- **Settings > Helpers** > Helper öffnen > Datum+Zeit auswählen
- **Developer Tools > Services** > `input_datetime.set_datetime` mit `target.entity_id` und `data.datetime: "2026-04-20 10:00:00"`
- Eigene Automations: Script `script.door_lock_pause` oder `script.door_lock_pause_until_morning` aufrufen

### Auto-Entpause

Sobald der Pause-Zeitpunkt erreicht ist, triggert `automation.door_lock_pause_expired` und sendet einen Info-Push "Türschloss Pause beendet". Die Config (`sensor.door_locks_active_config`) erkennt die Aufhebung automatisch.

## Master-Override

Über `input_select.autolock_master_mode` kann das Verhalten aller Locks global übersteuert werden. Nützlich z.B. im Urlaub oder für schnelles Umschalten ohne jeden Per-Lock-Modus anzufassen.

| Option | Verhalten |
|--------|-----------|
| **Je nach Lock** (Standard) | Jedes Lock folgt seinem Per-Lock-Modus (`input_select.autolock_<slug>_mode`) |
| **Alle aus** | Alle Locks werden ignoriert (kein Auto-Lock, egal welcher Per-Lock-Modus) |
| **Alle zentral** | Alle Locks folgen dem zentralen Sperrfenster |
| **Alle individuell** | Alle Locks folgen ihrem individuellen Sperrfenster (sofern Helper vorhanden) |

Zusätzlich wirkt `input_boolean.door_lock_auto_enabled` als harter An/Aus-Schalter über dem Master-Modus: Ist er `off`, greift kein Auto-Lock (Force-Event via `door_lock_force` übersteuert dies jedoch).

## Dashboard

> **Hinweis:** Ein fertiges Dashboard ist im Repo aktuell nicht enthalten (das Produktiv-Dashboard des Maintainers ist gitignored, weil es konkrete Lock-IDs referenziert). Die folgende Beschreibung skizziert die empfohlene Struktur — du baust das Dashboard lokal nach deinen Entities.

Empfohlen: natives **Sections-Layout** (HA 2024.10+):

- Eigene Sektion pro Lock mit farbigem Hintergrund (`background: true`) → klare visuelle Abgrenzung
- `heading`-Cards mit Status-Badges für Lock- und Tür-Zustand
- `tile`-Cards mit `features: [lock-commands]` → native Sperren/Entriegeln-Buttons
- **Modus-Dropdown** + **Start/Ende des Sperrfensters** direkt pro Lock editierbar
- Vier Pause-Shortcuts pro Lock (Pause 2h, Bis morgen, Bis…, Aufheben)
- Overview- und Aktivitäts-Sektion + Allgemeine Einstellungen

### Einbinden

1. **Settings > Dashboards > Dashboard hinzufügen** (Name, Icon)
2. Dashboard öffnen, rechts oben **Edit Dashboard**
3. Drei-Punkte-Menü > **Raw Configuration Editor**
4. Eigenes YAML schreiben — pro Lock eine Section mit Tile, Mode-Dropdown, Start/Ende, Pause-Buttons.

Per-Lock-Helper müssen vorher angelegt sein (siehe **Per-Lock-Konfiguration (Helper)**) — sonst zeigt das Dashboard "Entität nicht gefunden".

## Konfigurierbare Parameter

### Globale Einstellungen

| Entity | Beschreibung | Standard |
|--------|-------------|----------|
| `input_boolean.door_lock_auto_enabled` | Master-Schalter | `off` |
| `input_boolean.door_lock_notify_success` | Auch bei Erfolg benachrichtigen | `on` |
| `input_select.autolock_master_mode` | Master-Override (siehe oben) | "Je nach Lock" |
| `input_datetime.door_lock_schedule_start` | Zentrales Sperrfenster Start | 22:00 |
| `input_datetime.door_lock_schedule_end` | Zentrales Sperrfenster Ende | 06:00 |
| `input_number.door_lock_retry_count` | Max. Versuche pro Schloss | 3 |
| `input_number.door_lock_retry_delay` | Pause zwischen Versuchen | 30 Sek. |
| `input_number.door_lock_verify_timeout` | Timeout für Statusbestätigung | 15 Sek. |
| `input_number.door_lock_open_warning_minutes` | Default Tür-offen-Warnung | 15 Min. |
| `input_text.door_lock_entities` | Lock-Entity-IDs (komma-getrennt) | leer |
| `input_text.door_lock_notify_service` | Notify-Service für Push | leer |

### Per-Lock-Helper (Konvention)

Für jedes Lock mit Slug `<slug>` (z.B. `lock_pro_xxxx`):

| Entity | Typ | Pflicht? | Zweck |
|--------|-----|----------|-------|
| `input_select.autolock_<slug>_mode` | off/central/individual | **Pflicht** | Modus |
| `input_datetime.autolock_<slug>_start` | has_time | Pflicht bei `individual` | Sperrfenster-Start |
| `input_datetime.autolock_<slug>_end` | has_time | Pflicht bei `individual` | Sperrfenster-Ende |
| `input_number.autolock_<slug>_warning_minutes` | 5-60 | Optional | Individuelle Tür-offen-Warnzeit |
| `input_datetime.autolock_<slug>_pause_until` | has_date + has_time | Optional | Pause bis Zeitpunkt X |

## Status-Sensoren

| Entity | Beschreibung |
|--------|-------------|
| `binary_sensor.door_locks_all_secured` | `on` wenn alle aktiven Schlösser gesperrt |
| `sensor.door_locks_unlocked_list` | Namen der nicht gesperrten aktiven Schlösser |
| `sensor.door_locks_open_doors` | Namen der offenen Türen |
| `sensor.door_locks_active_config` | Anzahl aktiver Locks + Attribut `locks` mit Detailkonfig pro Lock (`mode`, `effective_start`, `effective_end`, `in_locktime`, `warning_minutes`, `paused`, `active`) |
| `sensor.autolock_config_issues` | Anzahl Konfigurationsprobleme + Attribut `issues` mit Klartext-Liste |

## Ablauf

```
                         input_boolean.door_lock_auto_enabled
                                        |
                                      [off] -> kein Auto-Lock
                                      [on]
                                        |
                                        v
                        input_select.autolock_master_mode
                      /            |             |            \
                   [Alle aus]  [Alle zentral] [Alle indiv.]  [Je nach Lock]
                      |            |             |             (Per-Lock-Modus)
                      |            |             |                  |
                      v            v             v                  v
                   ignorieren   zentr. Fenster individuelle     input_select.
                                 (22-06)      Fenster / Lock    autolock_<slug>_mode
                                        |
                                        v
                            pause_until > now? -> ja: Lock überspringen
                                        |
                                      nein
                                        |
                                        v
                              Zu Fenster-Start: Schloss sperren
                                     (parallel, mit Retry, Door-Open-Check)
                                        |
                                +---- Erfolg ----> Erfolgs-Push (optional)
                                +---- Fehler ----> Kritische Push-Warnung
                                        |
                                        v
                     Während Sperrfenster (start - end):
                              |
                              +-- Schloss wird entriegelt:
                              |      -> Info-Push "Schloss entriegelt"
                              |      -> 60s Puffer (BT-Türkontakt)
                              |      -> Warten bis Tür zufällt
                              |      -> Auto-Relock mit Retry
                              |      -> Bei Fehler: kritische Push
                              |
                              +-- Tür > N Min. offen (N = per-Lock oder Default):
                              |      -> Kritische Push-Warnung
                              |      -> Warten bis Tür geschlossen
                              |      -> Bestätigungs-Push
                              |
                              +-- Pause abgelaufen:
                                     -> Info-Push "Pause beendet"
```

## Benachrichtigungen

| Szenario | Push-Titel | Kritisch? |
|----------|-----------|-----------|
| Alle gesperrt (opt-in) | "Türschlösser gesichert" | Nein |
| Sperren fehlgeschlagen (pro Lock) | "Türschloss NICHT GESPERRT" | Ja |
| Tür offen beim Sperren | "Türschloss WARNUNG" | Ja |
| Schloss entriegelt (im Sperrfenster) | "Türschloss entriegelt" | Nein |
| Auto-Relock fehlgeschlagen (pro Lock) | "Türschloss NICHT GESPERRT" | Ja |
| Tür steht zu lange offen | "Tür steht offen!" | Ja |
| Tür wieder geschlossen | "Tür geschlossen" | Nein |
| Pause abgelaufen | "Türschloss Pause beendet" | Nein |

Failure-Pushes kommen **pro Lock** vom `door_lock_single_with_retry`-Script — nicht mehr aggregiert vom `door_lock_group`. So siehst du sofort welches Schloss betroffen ist.
| Config-Problem erkannt | "Türschloss Config-Problem" | Nein |

Kritische Benachrichtigungen durchbrechen den "Nicht stören"-Modus:
- **iOS**: via `push.sound.critical` — erfordert in der HA Companion App unter *Benachrichtigungen > Kritische Benachrichtigungen* die Aktivierung je Kategorie
- **Android**: via Notification-Channel `alarm_stream_max` — nutzt den Alarm-Audiostream und übersteuert DND/Stumm

## Fehlerverhalten

| Szenario | Verhalten |
|----------|-----------|
| Schloss reagiert nicht | Retry nach Timeout, max. konfigurierbare Versuche |
| Alle Retries erschöpft | Kritische Push-Warnung mit Schloss-Name |
| Tür beim Sperren offen | Wartet bis zur Warnzeit, dann Abbruch + kritische Warnung |
| Keine Schlösser konfiguriert | Script stoppt sofort |
| Entity existiert nicht | Config-Sync-Check meldet "existiert nicht oder ist unavailable" |
| Modus `individual` ohne `_start`/`_end`-Helper | Config-Sync-Check meldet Helper fehlt |
| Modus-Helper `input_select.autolock_<slug>_mode` fehlt | Lock silent als `off` behandelt + Config-Sync-Meldung |
| Pause-Helper nicht angelegt (Pause-Script aufgerufen) | Push-Fehlermeldung, Script stoppt |
| Automation läuft bereits | `mode: single` bzw. `mode: parallel` je nach Automation |
| Tür fällt nicht zu | Warnung nach per-Lock konfigurierter Zeit (oder Default) |

## Lizenz

MIT
