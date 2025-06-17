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
