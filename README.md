# Wallbox-PV-Steuerung
# Wallbox PV-Steuerung – Installation auf einem frischen System

Diese Anleitung führt dich von einer leeren Home-Assistant-Installation zu
einer vollständig funktionierenden, konsolidierten Wallbox-Automation plus
Dashboard.

## Was du bekommst

| Datei | Zweck |
|---|---|
| `wallbox_pv_steuerung.yaml` | Der Blueprint. Ersetzt alle 11 einzelnen Automationen. |
| `wallbox_helpers.yaml` | Alle benötigten Helper (input_select, input_number, input_datetime) als packages-Datei. |
| `wallbox_automation_example.yaml` | Referenz, welche Werte du beim Ausfüllen des Blueprints eintragen musst. |
| `wallbox_dashboard.yaml` | Das Dashboard (unverändert aus deiner bestehenden Config, siehe Hinweise unten). |

---

## 0) Voraussetzungen, die NICHT Teil dieses Pakets sind

Deine ursprünglichen Automationen setzen einige Dinge voraus, die du selbst
schon hast bzw. weiterhin brauchst – die sind hier bewusst nicht mit
automatisiert, weil sie systemspezifisch sind:

- Eine **go-eCharger-Integration** (liefert `select.<name>_frc`,
  `select.<name>_psm`, `number.<name>_amp`, `binary_sensor.<name>_car_0`,
  `sensor.<name>_nrg_11`, `sensor.<name>_wh`)
- Einen **Template-Sensor**, der den Ziel-Überschuss für die Wallbox in
  Watt liefert (bei dir `sensor.wallbox_ziel_uberschuss`). Dieser rechnet
  vermutlich PV-Leistung minus Hausverbrauch minus Batterie-Anteil – diese
  Logik war in den ursprünglichen Automationen nicht enthalten und muss
  separat vorhanden sein.
- Optional ein zweiter Sensor für den rohen, geglätteten PV-Überschuss
  (`sensor.verfuegbarer_pv_ueberschuss_geglaettet`) – wird nur vom
  Dashboard angezeigt, nicht von der Automation ausgewertet.
- Ein Batterie-SOC-Sensor, falls du eine Hausbatterie hast
  (`sensor.battery_soc`).
- Für das persistente Logfile: eine `notify`-Entity, z. B. über die
  `file`-notify-Plattform, plus die HTML-Datei `wallbox_log_viewer.html`
  unter `/config/www/`. Beides ist optional – ohne `notify_target` läuft
  die Automation trotzdem, nur ohne persistentes Textlog (das normale
  Logbook-Log über `logbook.log` bleibt in jedem Fall aktiv).
- Für das Dashboard: die HACS-Frontend-Module `button-card`,
  `mini-graph-card` und `card-mod`.

---

## 1) Helper anlegen

**Variante A – über packages (empfohlen für Portierung):**

1. Falls noch nicht geschehen, in `configuration.yaml` packages aktivieren:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Ordner `/config/packages/` anlegen (falls nicht vorhanden).
3. `wallbox_helpers.yaml` dorthin kopieren.
4. Home Assistant neu starten (Helper über packages benötigen einen
   Neustart, kein reines YAML-Reload).

**Variante B – manuell über die UI:**

Gehe zu *Einstellungen → Geräte & Dienste → Helfer → Helfer hinzufügen*
und lege folgende vier Helper an:

| Typ | Entity-ID (Vorschlag) | Einstellungen |
|---|---|---|
| Dropdown (input_select) | `input_select.lademodus` | Optionen exakt: `Kein Laden`, `PV-Überschuss`, `Schnell`, `Manuell` |
| Zahl (input_number) | `input_number.batterie_priority_soc` | Min 0, Max 100, Schritt 1, Einheit % |
| Zahl (input_number) | `input_number.wallbox_manuelle_leistung` | Min 1.4, Max 22, Schritt 0.1, Einheit kW |
| Zahl (input_number) | `input_number.wallbox_anteil_ueberschuss` | Min 0, Max 100, Schritt 5, Einheit % *(nur fürs Dashboard, siehe Hinweis unten)* |
| Datum/Uhrzeit (input_datetime) | `input_datetime.letzter_phasenwechsel` | Datum + Uhrzeit aktivieren |

> Die früheren `input_boolean.wallbox_automatikmodus` und
> `input_boolean.wallbox_11kw_modus` werden **nicht mehr benötigt** – sie
> wurden in keiner der Automationen ausgelesen, nur gesetzt, und sind im
> Blueprint durch direkte Abfrage von `input_select.lademodus` ersetzt.

Trage danach die Startwerte ein, z. B. `batterie_priority_soc = 80`.

---

## 2) Blueprint importieren

1. Ordner `/config/blueprints/automation/<dein_name>/` anlegen, falls
   nicht vorhanden.
2. `wallbox_pv_steuerung.yaml` dort hineinkopieren.
3. In Home Assistant: *Einstellungen → Automationen & Szenen →
   Blueprints → neu laden* (oder Neustart).

Alternativ direkt über die UI importieren: *Einstellungen → Automationen
& Szenen → Blueprints → Blueprint importieren* und den Inhalt der Datei
einfügen bzw. eine Rohdatei-URL angeben, falls du sie z. B. in einem
privaten Git-Repo hostest.

---

## 3) Automation aus dem Blueprint erstellen

1. *Einstellungen → Automationen & Szenen → Automation erstellen →
   aus Blueprint verwenden* → „Wallbox PV-Überschuss Steuerung (go-e
   Charger)" auswählen.
2. Die Eingabefelder ausfüllen. `wallbox_automation_example.yaml` zeigt
   dir, welche Werte du (bei identischer Wallbox-Konfiguration wie im
   Original) eintragen würdest – auf einem fremden System trägst du
   einfach die dort vorhandenen Entity-IDs ein.
3. Wichtig: Bei `Notify-Entity für persistentes Log` leer lassen, wenn
   du kein `notify.wallbox_log` eingerichtet hast.
4. Speichern.

Damit ersetzt diese **eine** Automation alle 11 ursprünglichen. Du kannst
die alten Automationen jetzt deaktivieren oder löschen.

---

## 4) Dashboard einbinden

1. Stelle sicher, dass über HACS folgende Frontend-Ressourcen installiert
   sind: `button-card`, `mini-graph-card`, `card-mod`.
2. Neues Dashboard anlegen (oder bestehendes um eine View erweitern) und
   im YAML-Modus den Inhalt aus `wallbox_dashboard.yaml` einfügen.
3. **Wichtig – Portierung:** Das Dashboard ist (anders als die
   Automation) NICHT parametrisiert, da Lovelace-Dashboards keine
   Blueprint-Inputs unterstützen. Wenn deine Entity-IDs auf dem neuen
   System anders heißen (z. B. andere Wallbox-Seriennummer als
   `goe_501900_*`), musst du im Dashboard-YAML einmalig per
   Suchen-&-Ersetzen die Entity-IDs anpassen. Betroffene Muster u. a.:
   - `binary_sensor.goe_501900_car_0`
   - `select.goe_501900_frc` / `select.goe_501900_psm`
   - `number.goe_501900_amp`
   - `sensor.goe_501900_nrg_11` / `sensor.goe_501900_wh`
   - `sensor.pv_fronius_und_mptt`
   - `sensor.victron_mqtt_c0619ab6af4d_*`
   - `sensor.garage_r5_batterie`, `number.garage_r5_*`
   - `sensor.wallbox_ziel_uberschuss`,
     `sensor.verfuegbarer_pv_ueberschuss_geglaettet`
   - `sensor.battery_soc`
4. Der `iframe`-Card am Ende referenziert `/local/wallbox_log_viewer.html`
   – falls du das Logfile-Feature nutzt, muss diese Datei unter
   `/config/www/wallbox_log_viewer.html` liegen. Sonst diese Card einfach
   entfernen.

---

## 5) Testen

- Fahrzeug an-/abstecken und prüfen, ob `select.goe_501900_frc` und
  `number.goe_501900_amp` korrekt reagieren.
- Lademodus über das Dashboard durchschalten (Kein Laden / PV-Überschuss
  / Schnell / Manuell) und Logbook (*Einstellungen → System → Logbuch*)
  beobachten – dort tauchen alle `logbook.log`-Einträge auf.
- `input_number.wallbox_manuelle_leistung` im Modus „Manuell" ändern und
  prüfen, ob sich der Ladestrom live anpasst.
- Optional: In der Automation unter *Traces* den Ausführungsverlauf pro
  `trigger.id` nachvollziehen – das ersetzt das Debuggen einzelner
  Automationen.

---

## Design-Entscheidungen, die du kennen solltest

- **`mode: queued, max: 10`**: Einige Aktionsblöcke enthalten Delays
  (z. B. 3s beim Phasenwechsel). Bei `mode: restart` (wie in einigen
  deiner Originalautomationen) hätte ein neuer Trigger – z. B. die
  Dynamische Anpassung alle 2 Minuten – einen gerade laufenden
  Phasenwechsel abbrechen können. `queued` stellt sicher, dass jeder
  Trigger vollständig abgearbeitet wird, ohne dass die Wallbox in einem
  inkonsistenten Zustand hängen bleibt.
- Die Schwellwerte (1380 W, 3500 W, 4200 W, Cooldowns etc.) sind jetzt
  Blueprint-Inputs mit den Originalwerten als Default – du kannst sie pro
  Installation direkt in der UI anpassen, ohne YAML zu editieren.
- `input_number.wallbox_anteil_ueberschuss` wird aktuell nur im Dashboard
  zur Visualisierung der Energieverteilung genutzt, aber (wie schon im
  Original) in keiner Automation ausgewertet. Falls du das ändern willst,
  sag Bescheid – das lässt sich leicht als zusätzlicher Blueprint-Input
  in die `dynamic_adjust`-Berechnung einbauen.
