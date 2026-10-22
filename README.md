# ESPHome Camper LTE Monitor

**English summary:** An ESP32 with LTE modem (LilyGO T-A7670E) running ESPHome
wakes from deep sleep, reads Victron (SmartShunt, MPPT), a BMS, a gas level
sensor and temperature sensors via Bluetooth, sends everything through a
WireGuard tunnel to Home Assistant at home and goes back to sleep. Built for
monitoring a campervan in winter storage without keeping a Cerbo GX and router
running. Documentation below is in German.

---

Fernüberwachung für Camper, Wohnmobil oder Boot im Stand: Ein LilyGO T-A7670E
mit ESPHome wacht regelmäßig auf und liest per Bluetooth Victron, BMS, Gas und
Temperaturen. Dann schickt er alles per LTE durch einen WireGuard-Tunnel an
Home Assistant zu Hause und geht wieder in den Deep Sleep.

Bastelprojekt, kein Fertigprodukt. Alles ist auf mein Fahrzeug zugeschnitten
und muss angepasst werden. Kein Support, Nutzung auf eigenes Risiko.

## Inhalt

| Datei | Wozu |
|---|---|
| `esphome/poessl-monitor.yaml` | Firmware für den LilyGO |
| `esphome/secrets.example.yaml` | Vorlage für die secrets.yaml |
| `homeassistant/poessl_monitor.yaml` | Package: Webhook, Sensoren aus dem Monitor, Warnungen, Wochenbericht, Benachrichtigungs-Skript |
| `homeassistant/poessl_fallback.yaml` | Package: Sensoren fürs Dashboard, die zwischen Cerbo und Monitor umschalten |
| `homeassistant/dashboards/*.yaml` | Dashboard-Ansichten Strom, Information, Meldungen |
| `node-red/nodered-esp-truma-cerbo.json` | Optional: Node-RED-Flow für den Cerbo GX (virtuelle Temperatur- und Tankgeräte) |

Dateinamen und Entitäten tragen noch das Präfix `poessl`/`possl` aus meinem
Fahrzeug (Pössl). Wer will, ersetzt es per Suchen und Ersetzen in allen
Dateien gleichzeitig.

## Ablauf eines Zyklus

1. Aufwachen, Strom am Shunt messen (Ruhestrom, bevor das Modem startet)
2. Modem einschalten, `ATE0` senden (siehe Stolpersteine), LTE verbinden
3. WireGuard-Tunnel zum Router zu Hause (bei mir Fritz!Box)
4. Bluetooth: Victron (Instant Readout), BMS (aktive Abfrage), Mopeka, Inkbird
5. JSON per HTTP an den Webhook von Home Assistant
6. Modem aus (`AT+CPOF`), Deep Sleep

Intervall (in `substitutions` einstellbar): 30 min, nachts 120 min,
bei niedrigem Ladestand 15 min. Ein Zyklus dauert etwa 40 s.

Endet ein Lauf mit Brownout, Absturz oder Watchdog, setzt die
Fehlstart-Sperre einen Zyklus aus, statt sofort neu zu starten. So entsteht
keine Neustart-Schleife, die die Batterie leert. Home Assistant meldet das
beim nächsten Kontakt.

## Hardware

- LilyGO T-A7670E (die Variante ohne GPS reicht), Versorgung mit 5 V
- SIM: bei mir 1NCE (500 MB / 10 Jahre); Verbrauch etwa 2–3 kB pro Meldung
- Bei mir per Bluetooth: Victron SmartShunt, Victron SmartSolar MPPT,
  CBT-Power-BMS, Mopeka Standard (Gas), vier Inkbird IBS-TH1 mini
- Ruhestrom des ganzen Systems (Victron-Geräte, BMS, Monitor) bei mir etwa 2 W

## Einrichtung

### ESPHome
1. `secrets.example.yaml` nach `secrets.yaml` kopieren und ausfüllen.
2. In `poessl-monitor.yaml` die Platzhalter-MACs `AA:BB:CC:DD:EE:0x` durch die
   eigenen ersetzen (BMS in `substitutions`, Mopeka und Inkbird weiter unten).
3. `peer_allowed_ips` im WireGuard-Block auf das eigene Heimnetz anpassen.
4. Nicht benötigte Sensoren rauswerfen, und zwar auch im JSON-Block
   (`root["..."]`), sonst bricht der Build ab.
5. Flashen per USB: `esphome run poessl-monitor.yaml`. Das Gerät hat kein
   WLAN und kein OTA.

Wichtig: Die Modem-Komponente kommt aus dem offenen Pull Request
[esphome#6721](https://github.com/esphome/esphome/pull/6721) und ist auf einen
festen Commit gepinnt (`external_components`). Mit offiziellem ESPHome allein
läuft das nicht. Victron über
[esphome-victron_ble](https://github.com/Fabian-Schmidt/esphome-victron_ble).

### BMS
Die Abfrage ist für CBT Power nachgebaut, nach dem Protokoll aus der
Python-Bibliothek [aiobmsble](https://github.com/patman15/aiobmsble)
(Modul `cbtpwr_bms`). Für ein anderes BMS den Block `ble_client` und den
Sensor `bms_rx` anhand des passenden aiobmsble-Moduls umschreiben. Mit
KI-Unterstützung (z. B. Claude) geht das gut. Wer kein BMS auslesen will,
löscht die BMS-Teile komplett.

### Home Assistant
1. Packages aktivieren (`configuration.yaml`):
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. `poessl_monitor.yaml` und `poessl_fallback.yaml` nach `packages/` kopieren.
3. In `poessl_monitor.yaml`:
   - `webhook_id` auf eine eigene, nicht erratbare ID setzen (gleiche ID in
     der secrets.yaml des ESP)
   - `notify.mobile_app_HANDY_1/2` durch die eigenen Dienste ersetzen
   - `DEINE_PANDORA_ID`: Entitäten meiner Pandora-Alarmanlage
     (Starterbatterie, Türen). Ohne Pandora diese Stellen entfernen.
4. In `poessl_fallback.yaml` die Cerbo-Entitäten anpassen. Sie kommen aus der
   Victron-MQTT-Integration, und die Nummern (z. B. `solarcharger_278`,
   `battery_279`) sind die VRM-Instanzen meiner Geräte. Ohne Cerbo die
   Cerbo-Zweige entfernen, dann gelten nur die Monitor-Werte.
5. Dashboards: Im Dashboard-Editor über „Rohdaten-Konfiguration“ die Ansicht
   einfügen. Benötigte Karten aus HACS: power-flow-card-plus, weather-card.

### Umschaltung Cerbo / Monitor
`binary_sensor.possl_cerbo_live` ist an, wenn sich irgendein
`sensor.victron_mqtt_*` in den letzten 15 Minuten **geändert** hat.
Geprüft wird `last_changed`, nicht `last_reported`: Die Integration meldet
alte Werte weiter, auch wenn der Cerbo längst aus ist. Die Fallback-Sensoren
nehmen bei „live“ die Cerbo-Werte, sonst die Werte des Monitors, solange
dessen letzte Meldung jünger als 2 h 15 min ist. Beim Solarertrag gilt das
Maximum aus beiden Quellen, weil der Cerbo nach einem Neustart 0 meldet.

### Node-RED auf dem Cerbo GX (optional)
Venus OS Large mit Node-RED. Der Flow liest Temperaturen und Gas vom
lokalen MQTT-Broker und legt sie mit den `victron-virtual`-Bausteinen
(node-red-contrib-victron) als virtuelle Geräte an. Das übersteht Neustarts
und Firmware-Updates und erscheint im VRM. Die MQTT-Topics stammen von
meinem zweiten ESP (Truma-Steuerung) und müssen angepasst werden.

## Stolpersteine

- **AT-Echo:** Der A7670E hat ab Werk `ATE1`. Die Moduserkennung der
  Komponente hält das zurückgespiegelte Paket für den Datenmodus und
  blockiert 20 s. Lösung: `ATE0` in jedem Zyklus vor `modem.enable`.
  `AT&W` wird vom A7670E abgelehnt, das Echo lässt sich also nicht dauerhaft
  abschalten.
- **Modem im Schlaf:** Bei Versorgung über VBUS schaltet GPIO12 das Modem
  nicht ab, daher `AT+CPOF`. Reset- und PWRKEY-Pin im Schlaf auf LOW halten.
- **WireGuard** braucht eine gültige Uhrzeit (SNTP über LTE).
- **Ruhestrom:** Gemessen wird vor dem Modemstart. Sonst sieht man nur den
  Wachzustand.
- **Mopeka:** Kalibrierung (`CUSTOM`, voll/leer in mm, Propananteil) an die
  eigene Flasche anpassen.

## Lizenz

MIT, siehe [LICENSE](LICENSE).
