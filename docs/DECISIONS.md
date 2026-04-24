# Design-Entscheidungen

Log der bewussten Entscheidungen mit Begründung. Für die **Was**-Ebene der Architektur siehe [ARCHITECTURE.md](ARCHITECTURE.md).

Format: Kontext → Entscheidung → Konsequenz.

---

## 1. Sperrfenster mit Start + Ende (Variante A)

**Kontext.** Ursprünglich gab es nur einen einzelnen Zeitpunkt (`_time`) pro Lock. Unklar war, ob der Relock-Schutz auch ein Ende hat oder ob er bis zum manuellen Aufschließen gilt.

**Entscheidung.** Jedes Lock hat `_start` und `_end`. `_start` triggert das Sperren, `_end` ist der Grenzwert für den Auto-Relock-Schutz. Dazwischen wird jede Entriegelung zurückgesetzt; außerhalb bleibt manuelles Entriegeln unberührt.

**Warum.** Fixer Auto-Unlock am Morgen war explizit unerwünscht — Türen sollen nur zu definierten Tageszeiten "selbsttätig" sein. Eine klar definierte Schutz-Zone mit Ende ist vorhersehbarer als "bis jemand manuell aufschließt".

**Konsequenz.** Midnight-Crossing-Logik nötig (`start > end` → `(now >= start OR now < end)`). `_end` ist **kein** Trigger.

---

## 2. Modus pro Lock via `input_select`, nicht via Entity-Labels

**Kontext.** Frühere Version nutzte HA-Labels (`autolock_central`, `autolock_individual`) zur Konfiguration pro Lock.

**Entscheidung.** Jeder Lock hat `input_select.autolock_<slug>_mode` mit Optionen `off / central / individual`.

**Warum.** Labels sind nur über Settings erreichbar, nicht per Dashboard. Ein `input_select` erscheint direkt als Dropdown im Dashboard.

**Konsequenz.** `sensor.door_locks_active_config` liest den Select, Labels werden ignoriert. Kein Fallback — klarer Cut-over.

---

## 3. Cut-over statt Dual-Write-Migration

**Kontext.** Die Umstellung auf `input_select`-Modi und Start+Ende-Helper hätte als sanfte Migration (beide Varianten parallel) laufen können.

**Entscheidung.** Harter Cut-over. Alte `_time`-Helper und label-basierte Logik wurden in einem Zug entfernt.

**Warum.** Produktiv nur ein Benutzer; sanfte Migration hätte doppelte Template-Logik über lange Zeit bedeutet.

**Konsequenz.** Wer aktualisiert, muss Helper neu anlegen und Settings migrieren. Dokumentiert im README.

---

## 4. `sensor.door_locks_active_config` als Quelle der Wahrheit

**Kontext.** Jedes Script und jede Automation musste ursprünglich eigene Jinja-Logik zu Modus/Pause/Fenster-Berechnung duplizieren.

**Entscheidung.** Ein zentraler Template-Sensor mit `locks`-Attribut = Dict pro Lock. Alles Derived wird dort berechnet; Scripts und Automations lesen nur.

**Warum.** DRY. Ein Bug in der Modus-Berechnung fixt sich an einer Stelle. Außerdem Debug-freundlich: im Dev-Tool der Live-Zustand aller Locks sichtbar.

**Konsequenz.** Template-Sensor-Änderungen wirken erst nach YAML-Reload. Attribute sind die API — Umbenennungen brechen alle Consumer.

---

## 5. Per-Lock-State-Trigger statt globalem `all_secured`-Trigger

**Kontext.** `door_lock_unlocked_during_locktime` triggerte ursprünglich auf `binary_sensor.door_locks_all_secured` `on`→`off`.

**Problem.** Wenn Lock A im 60-minütigen Tür-offen-Wait hängt, bleibt der Sensor konstant `off`. Wird dann Lock B manuell entriegelt, gibt es keinen `on`→`off`-Übergang mehr — die Automation feuert nicht. Ein hängendes Lock blockierte effektiv alle anderen.

**Entscheidung.** Ein state-Trigger pro Lock (`lock.<slug>` `locked`→`unlocked`). Die Automation läuft `mode: parallel, max: 10`.

**Warum.** Jeder Lock-Run ist isoliert. Tür-offen-Wait in einem Run blockiert andere nicht. Trigger feuern bei jedem Einzel-Lock-Wechsel.

**Konsequenz.** Pro Lock muss in `door_lock_auto_local.yaml` eine Trigger-Zeile ergänzt werden (HA-Trigger sind load-time-statisch, kein Templating möglich). Dokumentiert in [ARCHITECTURE.md](ARCHITECTURE.md#neuer-lock--checkliste).

---

## 6. 60-Sekunden-Puffer vor Auto-Relock-Wait

**Kontext.** Nach einem manuellen `unlock` meldet der BT-Türkontakt die "Tür offen"-Transition erst nach mehreren Sekunden Verzögerung.

**Problem.** Das direkt nach `unlock` aufgerufene `wait_template: "{{ is_state(contact, 'off') }}"` sah den Kontakt noch als `off` ("Tür zu"), passierte also sofort → 3s Delay → Lock wurde wieder zugesperrt, obwohl die Tür gerade aufgemacht wurde.

**Entscheidung.** Zwischen Info-Push und `wait_template` ein fester `delay: seconds: 60`.

**Warum.** Pragmatisch. Alternative wäre ein State-Wait mit `for: seconds: N` auf den Türkontakt, aber das ist komplexer und die BT-Latenz ist nicht deterministisch.

**Konsequenz.** Wer nach Entriegeln die Tür **nicht** öffnet, sieht nach 60s + 3s den Auto-Relock. Das ist akzeptabel — im Normalfall wird nach Unlock auch geöffnet.

---

## 7. Failure-Pushes pro Lock statt aggregiert

**Kontext.** `door_lock_group` verschickte früher eine einzige "Folgende Schlösser konnten nicht gesperrt werden: …"-Meldung am Ende.

**Entscheidung.** `door_lock_single_with_retry` pusht **selbst** eine kritische Meldung nach erschöpften Retries. `door_lock_group` aggregiert nur noch die opt-in Erfolgs-Meldung.

**Warum.** Sofort sichtbar, welches Schloss betroffen ist. Außerdem: Die Group-Aggregation wurde aus Timing-Gründen erratisch, weil `wait_template` auf `script.door_lock_single_with_retry: 'off'` nicht zuverlässig wartete, wenn mehrere Runs parallel liefen.

**Konsequenz.** Eine Failure-Meldung pro betroffenem Lock. Bei vielen gleichzeitig fehlgeschlagenen Locks mehrere Pushes — bewusst in Kauf genommen.

---

## 8. Lokale Dateien für Entity-IDs, öffentliche Dateien nur mit Platzhaltern

**Kontext.** Das Repo ist öffentlich auf GitHub. Konkrete Lock-IDs (mit MAC-Kurzform im Namen) sind zwar nicht direkt sicherheitskritisch, aber Produktiv-Metadaten.

**Entscheidung.** Zweiteilung analog zum Python-Pattern `settings.py` ↔ `settings_local.py`:

- Öffentlich committed: `door_lock_auto.yaml`, `door_lock_auto_local.yaml.example`, `door_lock_helpers.yaml.example`
- Gitignored: `door_lock_auto_local.yaml`, `door_lock_helpers_local.yaml`, `dashboards/`
- Platzhalter in öffentlichen Dateien: `lock.lock_pro_xxxx`, `lock.lock_ultra_yyyy`

**Warum.** Klare Trennung. CI-fähig (die öffentlichen Dateien laden standalone, wenn auch ohne Per-Lock-Automations).

**Konsequenz.** Drei `!include`-Einträge in `configuration.yaml`. Neue Locks werden immer in zwei Dateien gepflegt (helpers + auto_local). Dokumentiert in [ARCHITECTURE.md](ARCHITECTURE.md).

---

## 9. Automations `mode: parallel, max: 10`

**Kontext.** Mehrere Locks können gleichzeitig im Tür-offen-Wait hängen oder unabhängig voneinander Retry-Loops durchlaufen.

**Entscheidung.** `single_with_retry`, `pause_expired`, `scheduled_individual`, `unlocked_during_locktime` und `door_open_warning` laufen `mode: parallel, max: 10`. `door_lock_group` bleibt `mode: single`, weil es nur orchestriert.

**Warum.** Parallelität ist die natürliche Semantik — Lock A darf nicht auf Lock B warten.

**Konsequenz.** `max: 10` ist weit oberhalb realistischer Schloss-Zahlen, reicht als Schutz gegen runaway-Trigger.

---

## 10. Dashboard nicht im öffentlichen Repo

**Kontext.** Das Produktiv-Dashboard enthält pro Lock eine eigene Section mit konkreten Entity-IDs — bei 4 Locks entsprechend sichtbar.

**Entscheidung.** `dashboards/` komplett in `.gitignore`. Im README wird die Struktur nur beschrieben; kein fertiges Template committed.

**Warum.** Ein Template-Dashboard wäre wertvoll für Nachnutzer, aber der Maintainer hat explizit keine Produktiv-IDs im Repo. Kein guter Proxy für ein generisches Template gefunden, das nicht trotzdem wie ein Fingerabdruck wirkt.

**Konsequenz.** Nachnutzer müssen das Dashboard selbst schreiben. Falls sich jemand damit schwertut, kann später ein anonymisiertes Template-Dashboard nachgezogen werden.

---

## Änderungen an diesem Dokument

Neue Einträge am Ende anhängen und durchnummerieren. Bestehende Einträge ändern, wenn eine Entscheidung revidiert wird — dann aber den Grund der Revision dokumentieren, damit sichtbar bleibt, warum der alte Ansatz verworfen wurde.
