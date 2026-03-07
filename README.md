# homeassistant
EMS (Energy Management System) / Dynamic Time Group (48010–48019).

Architektura:
```text
FOXESS MODBUS
      ↓
REGISTER PARSER
      ↓
EMS ANALYTICS
(PV / SOC / Tariff / Export Price)
      ↓
EMS STRATEGY
      ↓
SAFETY CONTROL
      ↓
FOXESS CONTROLLER
```

Mapowanie rejestrów:
```text
| idx | rejestr | nazwa         | typ | opis                                                                                        |
| --- | ------- | ------------- | --- | ------------------------------------------------------------------------------------------- |
| 0   | 48010   | Enable        | U16 | 0 disable / 1 enable                                                                        |
| 1   | 48011   | Start Time    | U16 | High byte = hour, Low byte = minute                                                         |
| 2   | 48012   | End Time      | U16 | High byte = hour, Low byte = minute                                                         |
| 3   | 48013   | Work Mode     | U16 | 1 Self-Use, 2 Feed-In, [ 3 Backup, 4 Peak Shaving, 5 Power Station ], 6 Charge, 7 Discharge |
| 4   | 48014   | MaxSoC        | U8  | 10-100%                                                                                     |
| 5   | 48015   | MinSoC OnGrid | U8  | 10-100%                                                                                     |
| 6   | 48016   | FDSOC         | U16 | SOC dla Force Discharge                                                                     |
| 7   | 48017   | FDPWR         | U16 | moc W                                                                                       |
| 8   | 48018   | reserved      | U16 | —                                                                                           |
| 9   | 48019   | reserved      | U16 | —                                                                                           |
```

Sensory wejściowe EMS:
```text
sensor.energy_price
sensor.pv_power
sensor.grid_power
sensor.battery_soc
sensor.battery_power
```

Modbus – Dynamic Time Group:
```yaml
modbus:
  - name: foxess_modbus
    type: tcp
    host: 192.168.1.100
    port: 502

    sensors:

      - name: FoxESS Dynamic Time Group Raw
        slave: 247
        address: 48010
        input_type: holding
        data_type: custom
        structure: ">10H"
        count: 10
        scan_interval: 60
```

Parser FoxESS:
```yaml
template:

  - trigger:
      - platform: state
        entity_id: sensor.foxess_dynamic_time_group_raw

    sensor:

      - name: FoxESS Dynamic Time Group Parsed

        state: >
          {% set r = state_attr('sensor.foxess_dynamic_time_group_raw','registers') %}
          {{ 'ok' if r is not none and r | length >= 8 else 'unavailable' }}

        attributes:

          work_mode: >
            {% set r = state_attr('sensor.foxess_dynamic_time_group_raw','registers') %}
            {{ r[3] if r is not none and r|length > 3 else none }}

          max_soc: >
            {% set r = state_attr('sensor.foxess_dynamic_time_group_raw','registers') %}
            {{ r[4] if r is not none and r|length > 4 else none }}

          min_soc: >
            {% set r = state_attr('sensor.foxess_dynamic_time_group_raw','registers') %}
            {{ r[5] if r is not none and r|length > 5 else none }}

          fd_soc: >
            {% set r = state_attr('sensor.foxess_dynamic_time_group_raw','registers') %}
            {{ r[6] if r is not none and r|length > 6 else none }}

          fd_power: >
            {% set r = state_attr('sensor.foxess_dynamic_time_group_raw','registers') %}
            {{ r[7] if r is not none and r|length > 7 else none }}
```

Sensory diagnostyczne FoxESS:
```yaml
template:

  - sensor:

      - name: FoxESS Work Mode
        state: >
          {{ state_attr('sensor.foxess_dynamic_time_group_parsed','work_mode') }}

      - name: FoxESS Max SOC
        unit_of_measurement: "%"
        state: >
          {{ state_attr('sensor.foxess_dynamic_time_group_parsed','max_soc') }}

      - name: FoxESS Min SOC
        unit_of_measurement: "%"
        state: >
          {{ state_attr('sensor.foxess_dynamic_time_group_parsed','min_soc') }}

      - name: FoxESS FD SOC
        unit_of_measurement: "%"
        state: >
          {{ state_attr('sensor.foxess_dynamic_time_group_parsed','fd_soc') }}

      - name: FoxESS FD Power
        unit_of_measurement: W
        state: >
          {{ state_attr('sensor.foxess_dynamic_time_group_parsed','fd_power') }}
```

Parametry EMS:
```yaml
input_number:

  energy_export_price:
    name: Energy Export Price
    min: 0
    max: 2
    step: 0.01
    unit_of_measurement: PLN/kWh

  foxess_ems_max_soc:
    name: EMS Max SOC
    min: 10
    max: 100
    step: 1

  foxess_ems_min_soc:
    name: EMS Min SOC
    min: 10
    max: 100
    step: 1

  foxess_ems_fd_soc:
    name: EMS Force Discharge SOC
    min: 10
    max: 100
    step: 1

  foxess_ems_fd_power:
    name: EMS Discharge Power
    min: 0
    max: 10000
    step: 100
```

Helper write-if-changed:
```yaml
input_text:

  foxess_last_write_signature:
    name: FoxESS Last Write Signature
    max: 255
```

Taryfa G13:
```yaml
template:

  - sensor:

      - name: Energy Tariff

        state: >

          {% set h = now().hour %}
          {% set d = now().weekday() %}

          {% if d >= 5 %}
            cheap
          {% elif h < 6 %}
            cheap
          {% elif h >= 16 and h < 22 %}
            peak
          {% else %}
            normal
          {% endif %}
```

Analiza PV:
```yaml
template:

  - sensor:

      - name: EMS PV Level

        state: >

          {% set pv = states('sensor.pv_power') | float(0) %}

          {% if pv > 4000 %}
            high
          {% elif pv > 1500 %}
            medium
          {% else %}
            low
          {% endif %}
```

Ocena baterii:
```yaml
template:

  - sensor:

      - name: EMS Battery State

        state: >

          {% set soc = states('sensor.battery_soc') | float(0) %}

          {% if soc < 20 %}
            critical
          {% elif soc > 95 %}
            full
          {% else %}
            normal
          {% endif %}
```

Opłacalność sprzedaży:
```yaml
template:

  - sensor:

      - name: EMS Export Profitability

        state: >

          {% set price = states('input_number.energy_export_price') | float(0) %}

          {% if price > 0.80 %}
            high
          {% elif price > 0.40 %}
            medium
          {% else %}
            low
          {% endif %}
```

Strategia EMS:
```yaml
template:

  - sensor:

      - name: EMS Energy Strategy

        state: >

          {% set pv = states('sensor.ems_pv_level') %}
          {% set battery = states('sensor.ems_battery_state') %}
          {% set tariff = states('sensor.energy_tariff') %}
          {% set export = states('sensor.ems_export_profitability') %}

          {% if battery == 'critical' %}
            charge

          {% elif pv == 'high' and battery != 'full' %}
            self_use

          {% elif tariff == 'cheap' and battery != 'full' %}
            charge

          {% elif export == 'high' and battery != 'critical' %}
            discharge

          {% elif pv == 'high' and battery == 'full' %}
            export

          {% else %}
            self_use
          {% endif %}
```

EMS System Mode (diagnostyka):
```yaml
template:

  - sensor:

      - name: EMS System Mode

        state: >

          {% set s = states('sensor.ems_energy_strategy') %}

          {% if s == 'charge' %}
            GRID_CHARGE
          {% elif s == 'discharge' %}
            BATTERY_EXPORT
          {% elif s == 'export' %}
            PV_EXPORT
          {% else %}
            SELF_USE
          {% endif %}
```

Kontekst decyzji EMS:
```yaml
template:

  - sensor:

      - name: EMS Decision Context

        state: >

          PV={{ states('sensor.pv_power') }}
          SOC={{ states('sensor.battery_soc') }}
          Tariff={{ states('sensor.energy_tariff') }}
          ExportPrice={{ states('input_number.energy_export_price') }}
```

Mapowanie strategii → tryb FoxESS:
```yaml
template:

  - sensor:

      - name: EMS FoxESS Mode

        state: >

          {% set s = states('sensor.ems_energy_strategy') %}

          {% if s == 'charge' %}
            6
          {% elif s == 'discharge' %}
            7
          {% elif s == 'export' %}
            2
          {% else %}
            1
          {% endif %}
```

Skrypt zapisu Dynamic Time Group:
```yaml
script:

  foxess_dynamic_time_group_write:

    fields:
      enable:
      work_mode:
      max_soc:
      min_soc:
      fd_soc:
      fd_power:

    sequence:

      - service: modbus.write_registers
        data:

          hub: foxess_modbus
          slave: 247
          address: 48010

          values:

            - "{{ enable | default(1) }}"
            - "{{ (0 * 256) + 0 }}"
            - "{{ (23 * 256) + 59 }}"
            - "{{ work_mode | default(1) }}"
            - "{{ max_soc | default(100) }}"
            - "{{ min_soc | default(20) }}"
            - "{{ fd_soc | default(20) }}"
            - "{{ fd_power | default(0) }}"
            - 0
            - 0
```

Kontroler EMS (write-if-changed):
```yaml
automation:

  - alias: EMS FoxESS Controller

    trigger:

      - platform: state
        entity_id:
          - sensor.ems_energy_strategy
          - sensor.ems_foxess_mode
          - input_number.foxess_ems_max_soc
          - input_number.foxess_ems_min_soc
          - input_number.foxess_ems_fd_soc
          - input_number.foxess_ems_fd_power

    condition:

      - condition: template
        value_template: >

          {% set work_mode = states('sensor.ems_foxess_mode') | int(1) %}
          {% set max_soc = states('input_number.foxess_ems_max_soc') | int(100) %}
          {% set min_soc = states('input_number.foxess_ems_min_soc') | int(20) %}
          {% set fd_soc = states('input_number.foxess_ems_fd_soc') | int(20) %}
          {% set fd_power = states('input_number.foxess_ems_fd_power') | int(0) %}

          {% set signature = work_mode ~ '-' ~ max_soc ~ '-' ~ min_soc ~ '-' ~ fd_soc ~ '-' ~ fd_power %}

          {{ signature != states('input_text.foxess_last_write_signature') }}

    action:

      - variables:

          work_mode: "{{ states('sensor.ems_foxess_mode') | int(1) }}"
          max_soc: "{{ states('input_number.foxess_ems_max_soc') | int(100) }}"
          min_soc: "{{ states('input_number.foxess_ems_min_soc') | int(20) }}"
          fd_soc: "{{ states('input_number.foxess_ems_fd_soc') | int(20) }}"
          fd_power: "{{ states('input_number.foxess_ems_fd_power') | int(0) }}"

          signature: >
            {{ work_mode ~ '-' ~ max_soc ~ '-' ~ min_soc ~ '-' ~ fd_soc ~ '-' ~ fd_power }}

      - service: script.foxess_dynamic_time_group_write
        data:

          enable: 1
          work_mode: "{{ work_mode }}"
          max_soc: "{{ max_soc }}"
          min_soc: "{{ min_soc }}"
          fd_soc: "{{ fd_soc }}"
          fd_power: "{{ fd_power }}"

      - service: input_text.set_value
        data:

          entity_id: input_text.foxess_last_write_signature
          value: "{{ signature }}"
```
