# Home-Assistant-Spoolman-Script
A Home Assistant script to automatically track filament consumption from Bambu Lab printers in Spoolman.

# Home Assistant Script: Spoolman Multi-Color Update for Bambu Lab

A Home Assistant script to automatically track filament consumption from Bambu Lab printers in Spoolman.

Jump to your language:
[English Instructions](#english-instructions) | [Deutsche Anleitung](#deutsche-anleitung)

---

## English Instructions

This script for Home Assistant automates filament tracking for Bambu Lab printers with an AMS, in conjunction with a Spoolman database. After each completed print (whether single- or multi-color), the filament consumption is accurately deducted in grams from the corresponding spools in Spoolman.

### Prerequisites

Ensure the following components are installed and functional in your Home Assistant setup:

1.  **Home Assistant:** An up-to-date version.
2.  **Bambu Lab Printer with AMS:** The printer must be connected to your network.
3.  **Spoolman Add-on:** The Spoolman add-on must be installed via the Add-on Store and running.
4.  **Bambu Lab Integration:** The [Bambu Lab Integration](https://github.com/greghesp/ha-bambulab) must be installed via HACS and connected to your printer.
5.  **Spoolman Integration:** The [Spoolman Integration](https://github.com/Disane87/spoolman-homeassistant) must be installed via HACS and connected to your Spoolman server.

### Setup

#### Step 1: Spoolman Configuration

This script identifies spools via the RFID UID that is read from Bambu Lab filament spools.

-   **Enter the RFID UID** of your Bambu spools into the **"Lot Number" (`lot_nr`)** field in your Spoolman database.
-   For third-party filaments, you can write your own NFC tags and enter their IDs here.

#### Step 2: Create the Script in Home Assistant

1.  In Home Assistant, navigate to **Settings > Automations & Scenes > Scripts**.
2.  Create a new script and switch to YAML mode.
3.  Copy the entire content of the script below into the editor.
4.  **Customize the variables at the beginning of the script:** You must adapt the entity IDs for your `ams_trays`, your `weight_sensor`, and the technical `spoolman_device_id` to match your environment. Comments in the code explain how to find the correct values.
5.  Save the script with an `alias` like `Spoolman Final Update`.

```yaml
#
# Home Assistant Script: Spoolman Multi-Color Update for Bambu Lab Printers
#
# Author: Bastii420 (developed with the help of Google's Gemini)
# Version: 1.0
# Date: June 18, 2025
#
# Description: 
# This script is called after a 3D print is finished. It iterates through all 
# four slots of the Bambu Lab AMS, reads the filament consumption per slot, and 
# adds it to the existing used weight in the Spoolman database.
# It is suitable for both single- and multi-color prints.
#
# IMPORTANT: The RFID/UID of the Bambu filament spools must be entered into the
# "Lot Number" (lot_nr) field in Spoolman for the matching to work.
#
alias: update spoolman from bambu
description: Adds the new consumption to the existing used weight after a print (Multi-Color capable).
sequence:
  - variables:
      # =================================================================================
      # --- CUSTOMIZE (1/3): YOUR SENSOR NAMES ---
      # =================================================================================
      # Enter the exact entity IDs of your four AMS slot sensors here.
      # You can find them in Home Assistant's Developer Tools > States.
      ams_trays: &trays
        - sensor.p1s_01p00a380900656_ams_1_slot_1
        - sensor.p1s_01p00a380900656_ams_1_slot_2
        - sensor.p1s_01p00a380900656_ams_1_slot_3
        - sensor.p1s_01p00a380900656_ams_1_slot_4
        
      # Enter the entity ID of your weight sensor here.
      weight_sensor: 'sensor.p1s_01p00a380900656_gewicht_des_drucks'

      # =================================================================================
      # --- CUSTOMIZE (2/3): YOUR SPOOLMAN DEVICE ID ---
      # =================================================================================
      # This is the technical device ID of your Spoolman integration.
      # Find it using this code in the Developer Tools > Template editor:
      # {{ device_id('sensor.spoolman_spool_1') }} 
      # (This assumes you have at least one spool created in Spoolman).
      spoolman_device_id: 'b8990689ac4c22272096195e04f06214'

  # This loop iterates through the list of the four AMS sensors defined above.
  - repeat:
      for_each: *trays
      sequence:
        - variables:
            current_tray_sensor: "{{ repeat.item }}"
            slot_number: "{{ repeat.item.split('_')[-1] }}"
            weight_attribute_name: "AMS 1 Tray {{ slot_number }}"
            filament_used_this_print: "{{ state_attr(weight_sensor, weight_attribute_name) | float(0) }}"
            tag_uid: "{{ state_attr(current_tray_sensor, 'tag_uid') }}"
            # Find the matching Spoolman entity (not just the ID)
            matching_entity: >-
              {% set ns = namespace(entity=none) %}
              {% for spool_entity in device_entities(spoolman_device_id) %}
                {% if tag_uid | string == state_attr(spool_entity, 'lot_nr') | string %}
                  {% set ns.entity = spool_entity %}
                  {% break %}
                {% endif %}
              {% endfor %}
              {{ ns.entity }}

        # Choose an action based on conditions.
        - choose:
            # This action is only executed if filament was used AND we found a matching spool.
            - conditions:
                - "{{ filament_used_this_print > 0 }}"
                - "{{ matching_entity != none }}"
              sequence:
                # CORRECT ADDITION LOGIC:
                - variables:
                    # Read the already used weight from the found Spoolman sensor.
                    current_used_weight: "{{ state_attr(matching_entity, 'used_weight') | float(0) }}"
                    # Add the new consumption from the current print.
                    new_total_used: "{{ current_used_weight + filament_used_this_print }}"
                    spool_id_for_service: "{{ state_attr(matching_entity, 'id') }}"
                # Call the service with the new, correct sum.
                - service: spoolman.patch_spool
                  data:
                    id: "{{ spool_id_for_service }}"
                    used_weight: "{{ new_total_used }}" # The sum is passed here
                    last_used: "{{ now().isoformat() }}"
# The 'parallel' mode allows, in theory, for all spool updates to be sent concurrently.
mode: parallel
max: 10
```

#### Step 3: Create the Automation

This script is called by a simple automation when a print job is completed.

1.  Navigate to **Settings > Automations & Scenes > Automations**.
2.  Create a new, empty automation in YAML mode.
3.  Add the following code:

    ```yaml
    alias: Print finished -> Start final Spoolman update
description: When a print is finished, the final Spoolman script is called to update the consumption.
trigger:
  # This trigger reacts to the "Print finished" event directly from your printer.
  # We are using the "device" trigger you had yourself, as it is very reliable.
  - platform: device
    device_id: afbc3be07f08653fe1ce9632c30cd2aa # This is the unique ID of your printer device in HA
    domain: bambu_lab
    type: event_print_finished
action:
  # Step 1: Wait for 5 seconds.
  # This gives Home Assistant and all integrations (especially Spoolman) enough time
  # to synchronize their data after the print has finished. This solves our timing issue.
  - delay:
      seconds: 5
      
  # Step 2: Call the script.
  - service: script.spoolman_finales_update
mode: single
    ```
4.  **Adapt the `entity_id` in the `trigger` and the `service` call to match your entities,** then save the automation.

Done! The system is now fully functional.

---
<br>

## Deutsche Anleitung

Dieses Skript für Home Assistant automatisiert das Filament-Tracking für Bambu Lab Drucker mit AMS in Verbindung mit einer Spoolman-Datenbank. Nach jedem fertigen Druck (egal ob ein- oder mehrfarbig) wird der Filamentverbrauch grammgenau von den entsprechenden Spulen in Spoolman abgezogen.

### Voraussetzungen

Stelle sicher, dass die folgenden Komponenten in deinem Home Assistant installiert und funktionsfähig sind:

1.  **Home Assistant:** Eine aktuelle Version.
2.  **Bambu Lab Drucker mit AMS:** Der Drucker muss mit dem Netzwerk verbunden sein.
3.  **Spoolman Add-on:** Das Spoolman-Add-on muss über den Add-on Store installiert und gestartet sein.
4.  **Bambu Lab Integration:** Die [Bambu Lab Integration](https://github.com/greghesp/ha-bambulab) muss über HACS installiert und mit deinem Drucker verbunden sein.
5.  **Spoolman Integration:** Die [Spoolman Integration](https://github.com/Disane87/spoolman-homeassistant) muss über HACS installiert und mit deinem Spoolman-Server verbunden sein.

### Einrichtung

#### Schritt 1: Spoolman Konfiguration

Dieses Skript identifiziert die Spulen über die RFID-UID, die von Bambu Lab Filamenten ausgelesen wird.

-   **Trage die RFID-UID** deiner Bambu-Spulen in deiner Spoolman-Datenbank in das Feld **"Chargennummer" (Lot Number)** ein.
-   Für Filamente von Drittherstellern kannst du eigene NFC-Tags beschreiben und deren ID hier eintragen.

#### Schritt 2: Skript in Home Assistant erstellen

1.  Gehe in Home Assistant zu **Einstellungen > Automatisierungen & Szenen > Skripte**.
2.  Erstelle ein neues Skript und wechsle in den YAML-Modus.
3.  Kopiere den gesamten Inhalt des Skripts von oben in den Editor.
4.  **Passe die Variablen am Anfang des Skripts an:** Du musst die Entitäts-IDs deiner AMS-Sensoren, deines Gewicht-Sensors und die technische Geräte-ID deiner Spoolman-Integration an deine Umgebung anpassen. Kommentare im Code erklären, wie du die richtigen Werte findest.
5.  Speichere das Skript mit einem `alias` wie `Spoolman Finales Update`.

```yaml

#
# Home Assistant Skript: Spoolman Multi-Color Update für Bambu Lab Drucker
#
# Autor: Bastii420 (erarbeitet mit Hilfe von Google's Gemini)
# Version: 1.0
# Datum: 18. Juni 2025
#
# Beschreibung: 
# Dieses Skript wird nach einem fertigen 3D-Druck aufgerufen. Es durchläuft alle 
# vier Slots des Bambu Lab AMS, liest den Filamentverbrauch pro Slot aus und 
# addiert diesen zum bestehenden Verbrauch in der Spoolman-Datenbank.
# Es ist für Ein- und Mehrfarbdrucke geeignet.
#
# WICHTIG: Die RFID/UID der Bambu-Filamentspulen muss im Feld "Chargennummer" (lot_nr)
# in Spoolman eingetragen sein, damit die Zuordnung funktioniert.
#
alias: update spoolman from bambu
description: Addiert den neuen Verbrauch zum bestehenden Verbrauch hinzu (Multi-Color fähig).
sequence:
  - variables:
      # =================================================================================
      # --- ANPASSEN (1/3): DEINE SENSOR-NAMEN ---
      # =================================================================================
      # Trage hier die exakten Entitäts-IDs deiner vier AMS-Slot-Sensoren ein.
      # Du findest sie in den Home Assistant Entwicklerwerkzeugen > Zustände.
      ams_trays: &trays
        - sensor.p1s_01p00a380900656_ams_1_slot_1
        - sensor.p1s_01p00a380900656_ams_1_slot_2
        - sensor.p1s_01p00a380900656_ams_1_slot_3
        - sensor.p1s_01p00a380900656_ams_1_slot_4
        
      # Trage hier die Entitäts-ID deines Sensors ein, der den Gewichtsverbrauch anzeigt.
      weight_sensor: 'sensor.p1s_01p00a380900656_gewicht_des_drucks'

      # =================================================================================
      # --- ANPASSEN (2/3): DEINE SPOOLMAN GERÄTE-ID ---
      # =================================================================================
      # Dies ist die technische Geräte-ID deiner Spoolman-Integration.
      # Finde sie mit diesem Code im Entwicklerwerkzeug > Template:
      # {{ device_id('sensor.spoolman_spool_1') }} 
      # (Vorausgesetzt, du hast schon eine Spule in Spoolman angelegt)
      spoolman_device_id: 'b8990689ac4c22272096195e04f06214'

  # Diese Schleife geht die Liste der vier AMS-Sensoren von oben durch.
  - repeat:
      for_each: *trays
      sequence:
        - variables:
            current_tray_sensor: "{{ repeat.item }}"
            slot_number: "{{ repeat.item.split('_')[-1] }}"
            weight_attribute_name: "AMS 1 Tray {{ slot_number }}"
            filament_used_this_print: "{{ state_attr(weight_sensor, weight_attribute_name) | float(0) }}"
            tag_uid: "{{ state_attr(current_tray_sensor, 'tag_uid') }}"
            matching_entity: >-
              {% set ns = namespace(entity=none) %}
              {% for spool_entity in device_entities(spoolman_device_id) %}
                {% if tag_uid | string == state_attr(spool_entity, 'lot_nr') | string %}
                  {% set ns.entity = spool_entity %}
                  {% break %}
                {% endif %}
              {% endfor %}
              {{ ns.entity }}

        # Wähle eine Aktion basierend auf Bedingungen.
        - choose:
            # Diese Aktion wird nur ausgeführt, wenn Filament verbraucht wurde UND wir eine passende Spule gefunden haben.
            - conditions:
                - "{{ filament_used_this_print > 0 }}"
                - "{{ matching_entity != none }}"
              sequence:
                # KORREKTE LOGIK ZUM ADDIEREN:
                - variables:
                    # Lese den bereits verbrauchten Wert aus dem gefundenen Spoolman-Sensor.
                    current_used_weight: "{{ state_attr(matching_entity, 'used_weight') | float(0) }}"
                    # Addiere den neuen Verbrauch vom aktuellen Druck hinzu.
                    new_total_used: "{{ current_used_weight + filament_used_this_print }}"
                    spool_id_for_service: "{{ state_attr(matching_entity, 'id') }}"
                # Rufe den Dienst mit der neuen, korrekten Summe auf.
                - service: spoolman.patch_spool
                  data:
                    id: "{{ spool_id_for_service }}"
                    used_weight: "{{ new_total_used }}" # Hier wird die Summe übergeben
                    last_used: "{{ now().isoformat() }}"
mode: parallel
max: 10
```

#### Schritt 3: Automatisierung erstellen

Dieses Skript wird durch eine einfache Automatisierung aufgerufen, wenn ein Druck beendet ist.

1.  Gehe zu **Einstellungen > Automatisierungen & Szenen > Automatisierungen**.
2.  Erstelle eine neue, leere Automatisierung im YAML-Modus.
3.  Füge den folgenden Code ein:

    ```yaml
    alias: Druck fertig -> Starte finales Spoolman Update
description: Wenn ein Druck fertig ist, wird das finale Spoolman-Skript aufgerufen, um den Verbrauch zu aktualisieren.
trigger:
  # Dieser Trigger reagiert auf das "Druck beendet"-Ereignis direkt von deinem Drucker.
  # Wir verwenden hier den "device"-Trigger, den du selbst schon hattest, da er sehr zuverlässig ist.
  - platform: device
    device_id: afbc3be07f08653fe1ce9632c30cd2aa # Dies ist die einzigartige ID deines Drucker-Geräts in HA
    domain: bambu_lab
    type: event_print_finished
action:
  # Schritt 1: 5 Sekunden warten.
  # Dies gibt Home Assistant und allen Integrationen (besonders Spoolman) genug Zeit,
  # um nach dem Druck alle ihre Daten zu synchronisieren. Dies löst unser Timing-Problem.
  - delay:
      seconds: 5
      
  # Schritt 2: Das Skript aufrufen.
  - service: script.spoolman_finales_update
mode: single
    ```
4.  **Passe die `entity_id` im `trigger` und den `service`-Aufruf an deine Gegebenheiten an** und speichere die Automatisierung.

Fertig! Das System ist nun voll funktionsfähig.
