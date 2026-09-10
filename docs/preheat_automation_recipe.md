# Community Guide: Automating Pre-Peak Preheating for Hydro-Québec (Tarif Flex D / Crédit Hivernal)

This guide documents how to reliably trigger **pre-heating and pre-charging** 2 to 3 hours *before* a Hydro-Québec dynamic pricing peak event begins in Home Assistant.

---

## 1. The Challenge

Users of the [hydroqc/hydroqc-ha](https://github.com/hydroqc/hydroqc-ha) integration frequently ask:

> *"The peak binary sensor (`binary_sensor.hydro_quebec_*_critical_peak_active`) only turns `on` at 6:00 AM or 4:00 PM when the peak starts. How do I start pre-heating my thermostats and water heater 2 hours before the peak without hardcoding static schedules?"*

Attempting to rely strictly on the binary sensor means you cannot trigger actions *ahead* of the event. Furthermore, creating complex template sensors that calculate time differences can easily produce `unknown entity` or timestamp parsing errors.

---

## 2. The Solution: Calendar Triggers with Negative Offsets

Hydro-Québec peak events (morning: 6:00 AM – 9:00 AM; afternoon: 4:00 PM – 8:00 PM) are exposed as standard calendar events via `calendar.hydro_quebec_peak_calendar` (or via `ics_calendar`).

Home Assistant core natively supports negative time offsets on calendar triggers:

```yaml
triggers:
  - trigger: calendar
    event: start
    entity_id: calendar.hydro_quebec_peak_calendar
    offset: "-02:00:00"   # Fires exactly 2 hours BEFORE the peak event starts!
```

This trigger fires:
* At **4:00 AM** for the morning peak (6:00 AM – 9:00 AM).
* At **2:00 PM** for the evening peak (4:00 PM – 8:00 PM).
* **Only on confirmed peak days**, completely automatically.

---

## 3. Recommended 3-Stage Automation Lifecycle

### Stage 1: Pre-Heat Phase ($T - 2\\text{h}$)
* **Line-Voltage Thermostats (Sinopé / Mysa / Stelpro)**: Temporarily boost setpoint to **23.0°C** (storing heat in walls, floors, and furniture).
* **Water Heater (Sinopé RM3500WF / Calypso)**: Ensure the 240V breaker is **ON** so the water tank reaches maximum temperature.
* **Ventilation (HRV / ERV)**: Run pre-ventilation to flush $\\text{CO}_2$ and humidity before ventilation is paused.

### Stage 2: Peak Event (Drop Heavy Loads)
* **Thermostats**: Lower setpoint to **18.0°C – 18.5°C** (thermal inertia keeps the house comfortable; heaters stay off).
* **Water Heater**: Turn **OFF** the 50A breaker controller (saves 3,000 W – 4,500 W during the $0.50+/kWh peak).
* **Ventilation**: Turn **OFF** or set to standby/recirculation (prevents pulling $-20°C$ outdoor air into the house).
* **Non-Essential Loads**: Shed outdoor decorative lighting, pool shed heaters, and auxiliary plugs.

### Stage 3: Post-Peak Recovery
* **Thermostats**: Restore normal setpoint (**21.0°C – 21.5°C**).
* **Water Heater**: Restore breaker to **ON**.
* **Ventilation**: Restore normal air exchange.
* **Lighting**: Restore evening lighting routine if the sun is below the horizon.

---

## 4. Complete Automation Blueprint & YAML

For a turnkey implementation with a full UI configuration dialog, import our community blueprint:
`blueprints/automation/hydro_quebec_peak_manager.yaml`

Or implement the direct YAML automation:

```yaml
alias: "Hydro-Québec: Pre-Heat Phase (T-2h)"
description: "Boosts heating setpoints and preheats domestic water tank 2 hours before peak."
mode: restart
triggers:
  - trigger: calendar
    event: start
    entity_id: calendar.hydro_quebec_peak_calendar
    offset: "-02:00:00"
actions:
  # Boost thermostats to 23°C
  - action: climate.set_temperature
    target:
      entity_id:
        - climate.thermostat_living_room
        - climate.thermostat_bedrooms
        - climate.thermostat_bathrooms
    data:
      temperature: 23.0

  # Ensure water heater is charging
  - action: switch.turn_on
    target:
      entity_id: switch.water_heater_50a_breaker

  # Send push alert
  - action: notify.notify
    data:
      title: "⚡ Hydro-Québec: Pre-heating Active"
      message: "Pre-heating started. Thermostats set to 23°C and water heater charging before peak."
```

---

## 5. Frequently Asked Questions

**Q: What if the calendar integration does not populate early enough?**  
A: Hydro-Québec confirms peak events on the previous day by 5:00 PM. Ensure your calendar sync polling interval (`ics_calendar` or `hydroqc`) is set to poll at least once every 15–30 minutes in the late afternoon.

**Q: Will our hot water stay hot during the peak without power?**  
A: A modern insulated 40-to-60 gallon tank loses less than 0.5°C to 1°C per hour of standby loss. Pre-heating to 60°C guarantees ample hot water for hand washing and cooking throughout a 3-to-4 hour peak without the heating element firing.
