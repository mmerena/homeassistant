# homeassistant
EMS (Energy Management System) / Dynamic Time Group (48010–48019).

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

Modbus – Dynamic Time Group RAW:
```yaml
modbus:
  - name: foxess_modbus
    type: tcp
    host: 192.168.1.100
    port: 502
    timeout: 5
    retries: 3
    delay: 0
    message_wait_milliseconds: 30

    sensors:

      - name: FoxESS Dynamic Time Group Raw
        unique_id: foxess_dynamic_time_group_raw
        slave: 247
        address: 48010
        input_type: holding
        data_type: custom
        structure: ">10H"
        count: 10
        scan_interval: 60
```

Parsowanie Dynamic Time Group:
```yaml
template:

  - trigger:

      - platform: state
        entity_id: sensor.foxess_dynamic_time_group_raw

    sensor:

      - name: FoxESS Dynamic Time Group Parsed
        unique_id: foxess_dynamic_time_group_parsed

        state: >
          {% set r = state_attr('sensor.foxess_dynamic_time_group_raw','registers') %}
          {{ 'ok' if r is not none and r | length >= 8 else 'unavailable' }}

        attributes:

          enable: >
            {% set r = state_attr('sensor.foxess_dynamic_time_group_raw','registers') %}
            {{ r[0] if r is not none else none }}

          start_hour: >
            {% set r = state_attr('sensor.foxess_dynamic_time_group_raw','registers') %}
            {{ (r[1] // 256) if r is not none else none }}

          start_min: >
            {% set r = state_attr('sensor.foxess_dynamic_time_group_raw','registers') %}
            {{ (r[1] % 256) if r is not none else none }}

          end_hour: >
            {% set r = state_attr('sensor.foxess_dynamic_time_group_raw','registers') %}
            {{ (r[2] // 256) if r is not none else none }}

          end_min: >
            {% set r = state_attr('sensor.foxess_dynamic_time_group_raw','registers') %}
            {{ (r[2] % 256) if r is not none else none }}

          work_mode: >
            {% set r = state_attr('sensor.foxess_dynamic_time_group_raw','registers') %}
            {{ r[3] if r is not none else none }}

          max_soc: >
            {% set r = state_attr('sensor.foxess_dynamic_time_group_raw','registers') %}
            {{ r[4] if r is not none else none }}

          min_soc: >
            {% set r = state_attr('sensor.foxess_dynamic_time_group_raw','registers') %}
            {{ r[5] if r is not none else none }}

          fd_soc: >
            {% set r = state_attr('sensor.foxess_dynamic_time_group_raw','registers') %}
            {{ r[6] if r is not none else none }}

          fd_power: >
            {% set r = state_attr('sensor.foxess_dynamic_time_group_raw','registers') %}
            {{ r[7] if r is not none else none }}
```

Dekodowanie Dynamic Time Group:
```yaml
template:

  - sensor:

      - name: FoxESS Dynamic Start Time
        state: >
          {% set h = state_attr('sensor.foxess_dynamic_time_group_parsed','start_hour') %}
          {% set m = state_attr('sensor.foxess_dynamic_time_group_parsed','start_min') %}
          {{ "%02d:%02d"|format(h,m) if h is not none else 'unknown' }}

      - name: FoxESS Dynamic End Time
        state: >
          {% set h = state_attr('sensor.foxess_dynamic_time_group_parsed','end_hour') %}
          {% set m = state_attr('sensor.foxess_dynamic_time_group_parsed','end_min') %}
          {{ "%02d:%02d"|format(h,m) if h is not none else 'unknown' }}

      - name: FoxESS Dynamic Work Mode
        state: >

          {% set v = state_attr('sensor.foxess_dynamic_time_group_parsed','work_mode') %}

          {% if v == 1 %}Self Use
          {% elif v == 2 %}Feed In
          {% elif v == 6 %}Force Charge
          {% elif v == 7 %}Force Discharge
          {% else %}Unsupported
          {% endif %}

      - name: FoxESS Dynamic Max SOC
        unit_of_measurement: "%"
        state: >
          {{ state_attr('sensor.foxess_dynamic_time_group_parsed','max_soc') }}

      - name: FoxESS Dynamic Min SOC
        unit_of_measurement: "%"
        state: >
          {{ state_attr('sensor.foxess_dynamic_time_group_parsed','min_soc') }}

      - name: FoxESS Dynamic FD SOC
        unit_of_measurement: "%"
        state: >
          {{ state_attr('sensor.foxess_dynamic_time_group_parsed','fd_soc') }}

      - name: FoxESS Dynamic FD Power
        unit_of_measurement: W
        state: >
          {{ state_attr('sensor.foxess_dynamic_time_group_parsed','fd_power') }}
```

Parametry EMS (sterowanie):
```yaml
input_number:

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

Helper dla write-if-changed:
```yaml
input_text:

  foxess_last_write_signature:
    name: FoxESS Last Write Signature
    max: 255
```

Analiza ceny energii:
```yaml
template:

  - sensor:

      - name: EMS Price Level
        state: >

          {% set price = states('sensor.energy_price') | float(0) %}

          {% if price < 0.30 %}
            cheap
          {% elif price > 0.80 %}
            expensive
          {% else %}
            normal
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
          {% elif soc > 90 %}
            full
          {% else %}
            normal
          {% endif %}
```

Strategia EMS:
```yaml
template:

  - sensor:

      - name: EMS Energy Strategy

        state: >

          {% set price = states('sensor.ems_price_level') %}
          {% set battery = states('sensor.ems_battery_state') %}
          {% set pv = states('sensor.pv_power') | float(0) %}

          {% if price == 'unknown' or battery == 'unknown' %}
            self_use
          {% elif price == 'cheap' and battery != 'full' %}
            charge
          {% elif price == 'expensive' and battery != 'critical' %}
            discharge
          {% elif pv > 3000 %}
            export
          {% else %}
            self_use
          {% endif %}
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

Automatyczny kontroler EMS (write-if-changed):
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
