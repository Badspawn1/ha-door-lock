# SwitchBot Türschloss - Auto-Lock

Home Assistant Package zum automatischen Sperren von [SwitchBot Lock](https://www.switch-bot.com/products/switchbot-lock-pro) Türschlössern (Lock Pro, Lock Ultra) mit Retry-Logik, Auto-Relock und Benachrichtigungen.

## Features

- **Zeitgesteuerte Sperrung** - Alle Schlösser zur konfigurierbaren Uhrzeit sperren (Standard: 22:00)
- **Retry-Logik** - Automatische Wiederholungsversuche bei Fehlschlag (konfigurierbar)
- **Statusverifikation** - Prüft ob das Schloss tatsächlich gesperrt wurde
- **Auto-Relock** - Sperrt automatisch wieder, sobald die Tür zufällt (während der Sperrzeit)
- **Tür-offen-Warnung** - Kritische Push-Warnung wenn eine Tür länger als 15 Min. offen steht
- **Push-Benachrichtigungen** - Kritische Warnungen bei Fehler (durchbrechen "Nicht stören"), Info-Push bei Entriegelung
- **Manueller Trigger** - Per Event jederzeit auslösbar

## Entity-Schema

Das Package leitet den Türkontaktsensor automatisch vom Lock-Entity ab (funktioniert für Lock Pro und Lock Ultra):

```
lock.lock_pro_xxxx    ->  binary_sensor.lock_pro_xxxx
lock.lock_ultra_xxxx  ->  binary_sensor.lock_ultra_xxxx
```

Es wird kein separates Mapping benötigt.

## Voraussetzungen

- Home Assistant 2024.1+
- SwitchBot Lock Pro oder Lock Ultra mit integriertem Türkontakt
- SwitchBot Integration (via Bluetooth oder Cloud)
- HA Companion App (für Push-Benachrichtigungen)

## Installation

### 1. Datei kopieren

Kopiere `packages/door_lock_auto.yaml` nach `/config/packages/` auf deinem Home Assistant Server.

```bash
# Per SCP:
scp packages/door_lock_auto.yaml root@homeassistant.local:/config/packages/

# Oder manuell per File Editor Add-on
```

### 2. configuration.yaml erweitern

```yaml
homeassistant:
  packages:
    door_lock_auto: !include packages/door_lock_auto.yaml
```

### 3. Home Assistant neu starten

**Settings > System > Restart** - ein YAML-Neuladen reicht nicht, da `input_*` Entities nur beim Systemstart geladen werden.

### 4. Schlösser konfigurieren

Nach dem Neustart in **Developer Tools > States**:

1. `input_text.door_lock_entities` - Lock-Entity-IDs komma-getrennt eintragen:
   ```
   lock.lock_pro_xxxx,lock.lock_pro_yyyy,lock.lock_pro_zzzz
   ```
2. `input_text.door_lock_notify_service` - Notify-Service setzen:
   ```
   notify.mobile_app_your_phone
   ```
3. `input_boolean.door_lock_auto_enabled` auf `on` setzen
4. Optional: Sperrzeit anpassen (Standard: 22:00 - 06:00)

### 5. Testen

Manuell über **Developer Tools > Events**:

```
Event Type: door_lock_force
```

Dies sperrt alle konfigurierten Schlösser sofort (unabhängig vom Master-Schalter).

## Konfigurierbare Parameter

| Entity | Beschreibung | Standard |
|--------|-------------|----------|
| `input_boolean.door_lock_auto_enabled` | Master-Schalter | `off` |
| `input_boolean.door_lock_notify_success` | Auch bei Erfolg benachrichtigen | `on` |
| `input_datetime.door_lock_schedule_time` | Sperrzeit Beginn | 22:00 |
| `input_datetime.door_lock_schedule_end` | Sperrzeit Ende | 06:00 |
| `input_number.door_lock_retry_count` | Max. Versuche pro Schloss | 3 |
| `input_number.door_lock_retry_delay` | Pause zwischen Versuchen | 30 Sek. |
| `input_number.door_lock_verify_timeout` | Timeout für Statusbestätigung | 15 Sek. |
| `input_number.door_lock_open_warning_minutes` | Tür-offen-Warnung nach | 15 Min. |
| `input_text.door_lock_entities` | Lock-Entity-IDs (komma-getrennt) | leer |
| `input_text.door_lock_notify_service` | Notify-Service für Push | leer |

## Status-Sensoren

| Entity | Beschreibung |
|--------|-------------|
| `binary_sensor.door_locks_all_secured` | `on` wenn alle Schlösser gesperrt |
| `sensor.door_locks_unlocked_list` | Namen der nicht gesperrten Schlösser |
| `sensor.door_locks_open_doors` | Namen der offenen Türen |

## Ablauf

```
22:00 Sperrzeit beginnt
  |
  v
Alle Schlösser parallel sperren (mit Retry)
  |
  +-- Erfolg -> Erfolgs-Push (optional)
  +-- Fehler -> Kritische Push-Warnung
  |
  v
Während der Sperrzeit (22:00 - 06:00):
  |
  +-- Schloss wird entriegelt:
  |     -> Info-Push "Schloss entriegelt"
  |     -> Warten bis Tür zufällt
  |     -> Auto-Relock mit Retry
  |     -> Bestätigungs-Push
  |
  +-- Tür steht > 15 Min. offen:
  |     -> Kritische Push-Warnung
  |     -> Warten bis Tür geschlossen
  |     -> Bestätigungs-Push
  |
  v
06:00 Sperrzeit endet
```

## Benachrichtigungen

| Szenario | Push-Titel | Kritisch? |
|----------|-----------|-----------|
| Alle gesperrt | "Türschlösser gesichert" | Nein |
| Sperren fehlgeschlagen | "Türschloss WARNUNG" | Ja |
| Schloss entriegelt (Sperrzeit) | "Türschloss entriegelt" | Nein |
| Auto-Relock erfolgreich | "Türschlösser gesichert" | Nein |
| Auto-Relock fehlgeschlagen | "Türschloss WARNUNG" | Ja |
| Tür steht zu lange offen | "Tür steht offen!" | Ja |
| Tür wieder geschlossen | "Tür geschlossen" | Nein |

Kritische Benachrichtigungen durchbrechen den "Nicht stören"-Modus:
- **iOS**: via `push.sound.critical` — erfordert in der HA Companion App unter *Benachrichtigungen > Kritische Benachrichtigungen* die Aktivierung je Kategorie
- **Android**: via Notification-Channel `alarm_stream_max` — nutzt den Alarm-Audiostream und übersteuert DND/Stumm

## Fehlerverhalten

| Szenario | Verhalten |
|----------|-----------|
| Schloss reagiert nicht | Retry nach Timeout, max. konfigurierbare Versuche |
| Alle Retries erschöpft | Kritische Push-Warnung mit Schloss-Name |
| Keine Schlösser konfiguriert | Script stoppt sofort |
| Entity existiert nicht | Wird als "nicht gesperrt" gemeldet |
| Automation läuft bereits | `mode: single` bzw. `mode: parallel` je nach Automation |
| Tür fällt nicht zu | Warnung nach konfigurierter Zeit |

## Lizenz

MIT
