# CLAUDE.md — ha-door-lock

## Projekt

HA Package für automatisches Sperren von SwitchBot Lock Pro/Ultra Schlössern mit Retry-Logik, Auto-Relock und Push-Benachrichtigungen.

## Aktueller Stand

- `packages/door_lock_auto.yaml` — fertig implementiert
- `README.md` — fertig
- GitHub: https://github.com/Badspawn1/ha-door-lock (Branch: `master`)
- **Noch nicht auf HA deployed/getestet**

## Entity-Schema

Türkontaktsensor wird automatisch vom Lock-Entity abgeleitet — kein Mapping nötig:
```
lock.lock_pro_xxxx  →  binary_sensor.lock_pro_xxxx
```

## Nach Deployment konfigurieren

In Developer Tools > States:
1. `input_text.door_lock_entities` = komma-getrennte Lock-IDs, z.B. `lock.lock_pro_4afb`
2. `input_text.door_lock_notify_service` = z.B. `notify.mobile_app_your_phone`
3. `input_boolean.door_lock_auto_enabled` = `on`
4. Testen: Event `door_lock_force` feuern

## Wichtige Implementierungsdetails

- Parallele Ausführung: `door_lock_single_with_retry` läuft mit `mode: parallel` (max 10)
- Orchestrator `door_lock_all` feuert alle Schlösser per `script.turn_on` (fire-and-forget), wartet dann auf `script.door_lock_single_with_retry` state == `off`
- Sperrzeit kann über Mitternacht gehen (Template berücksichtigt start > end)
- Auto-Relock: wartet auf Türschließen, sperrt dann automatisch nach
- Kritische Notifications durchbrechen iOS "Nicht stören" via `push.sound.critical: 1`

## Git-Hinweis

Repo wurde einmal neu erstellt um persönliche Entity-IDs aus der History zu entfernen. Keine echten Geräte-IDs oder Notify-Services in den Dateien — immer Platzhalter wie `lock.lock_pro_xxxx` verwenden.
