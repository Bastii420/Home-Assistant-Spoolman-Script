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
4.  **Bambu Lab Integration:** The [Bambu Lab Integration](https://github.com/greghesp/bambu-lab-hass-integration) must be installed via HACS and connected to your printer.
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
# Author: [Your Name/GitHub Username]
# Version: 1.0 (developed with the help of Google's Gemini)
# Date: June 18, 2025
#
# Description: 
# This script is called after a 3D print is finished. It iterates through all 
# four slots of the Bambu Lab AMS, reads the filament consumption per slot, and 
# updates the corresponding entry in the Spoolman database.
# It is suitable for both single- and multi-color prints.
#
# IMPORTANT: The RFID/UID of the Bambu filament spools must be entered into the
# "Lot Number" (lot_nr) field in Spoolman for the matching to work.
#
alias: Spoolman Final Update
description: Updates all used spools in Spoolman after a print (Multi-Color capable).
sequence:
  - variables:
      # =================================================================================
      # --- CUSTOMIZE (1/3): YOUR SENSOR NAMES ---
      # =================================================================================
      # Enter the exact entity IDs of your four AMS slot sensors here.
      # You can find them in Home Assistant's Developer Tools > States.
      ams_trays: &trays
        - sensor.YOUR_PRINTER_ams_1_slot_1
        - sensor.YOUR_PRINTER_ams_1_slot_2
        - sensor.YOUR_PRINTER_ams_1_slot_3
        - sensor.YOUR_PRINTER_ams_1_slot_4
        
      # Enter the entity ID of your weight sensor here.
      weight_sensor: 'sensor.YOUR_PRINTER_gewicht_des_drucks'

      # =================================================================================
      # --- CUSTOMIZE (2/3): YOUR SPOOLMAN DEVICE ID ---
      # =================================================================================
      # This is the technical device ID of your Spoolman integration.
      # Find it using this code in the Developer Tools > Template editor:
      # {{ device_id('sensor.spoolman_spool_1') }} 
      # (This assumes you have at least one spool created in Spoolman).
      spoolman_device_id: 'YOUR_SPOOLMAN_DEVICE_ID'

  # This loop iterates through the list of the four AMS sensors defined above.
  - repeat:
      for_each: *trays
      sequence:
        - variables:
            current_tray_sensor: "{{ repeat.item }}"
            slot_number: "{{ repeat.item.split('_')[-1] }}"
            weight_attribute_name: "AMS 1 Tray {{ slot_number }}"
            filament_used: "{{ state_attr(weight_sensor, weight_attribute_name) | float(0) }}"
            tag_uid: "{{ state_attr(current_tray_sensor, 'tag_uid') }}"
            # Find the matching spool in Spoolman by comparing the printer's RFID with the spool's Lot Number.
            matching_spool_id: >-
              {% set ns = namespace(id=none) %} 
              {% for spool_entity in device_entities(spoolman_device_id) %}
                {# This is where the comparison happens #}
                {% if tag_uid | string == state_attr(spool_entity, 'lot_nr') | string %}
                  {% set ns.id = state_attr(spool_entity, 'id') %}
                  {% break %}
                {% endif %}
              {% endfor %} 
              {{ ns.id }}

        # Choose an action based on conditions.
        - choose:
            # This action is only executed if...
            - conditions:
                # ...filament was consumed (more than 0g) AND...
                - "{{ filament_used > 0 }}"
                # ...we found a matching spool ID in Spoolman.
                - "{{ matching_spool_id != none }}"
              # If both conditions are true, execute this sequence:
              sequence:
                # Call the Spoolman service to update the weight and last used time.
                - service: spoolman.patch_spool
                  data:
                    id: "{{ matching_spool_id }}"
                    used_weight: "{{ filament_used }}"
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
    alias: "Print Finished -> Start Final Spoolman Update"
    trigger:
      # This is a generic trigger. Users only need to adapt the sensor name.
      - platform: state
        entity_id: sensor.YOUR_PRINTER_print_status
        to: "finished"
    action:
      # Add a short delay for stability, ensuring all sensor data is available.
      - delay:
          seconds: 5
      # Call the script. The name must match the 'alias' of your script.
      - service: script.spoolman_final_update
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
4.  **Bambu Lab Integration:** Die [Bambu Lab Integration](https://github.com/greghesp/bambu-lab-hass-integration) muss über HACS installiert und mit deinem Drucker verbunden sein.
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

#### Schritt 3: Automatisierung erstellen

Dieses Skript wird durch eine einfache Automatisierung aufgerufen, wenn ein Druck beendet ist.

1.  Gehe zu **Einstellungen > Automatisierungen & Szenen > Automatisierungen**.
2.  Erstelle eine neue, leere Automatisierung im YAML-Modus.
3.  Füge den folgenden Code ein:

    ```yaml
    alias: "Druck fertig -> Starte finales Spoolman Update"
    trigger:
      # Dieser Trigger ist allgemeingültig. 
      # Nutzer müssen nur den Namen ihres Drucker-Status-Sensors anpassen.
      - platform: state
        entity_id: sensor.DEIN_DRUCKER_print_status
        to: "finished"
    action:
      # Füge eine kleine Verzögerung hinzu, um sicherzustellen, dass alle Daten verfügbar sind.
      - delay:
          seconds: 5
      # Rufe das Skript auf. Der Name muss mit dem 'alias' deines Skripts übereinstimmen.
      - service: script.spoolman_finales_update
    mode: single
    ```
4.  **Passe die `entity_id` im `trigger` und den `service`-Aufruf an deine Gegebenheiten an** und speichere die Automatisierung.

Fertig! Das System ist nun voll funktionsfähig.
