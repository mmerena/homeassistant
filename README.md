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

      - name: FoxESS 48010-48019 Dynamic Time Group Raw
        unique_id: foxess_48010_48019_dynamic_time_group_raw
        slave: 247
        address: 48010
        input_type: holding
        data_type: custom
        structure: ">10H"
        count: 10
        scan_interval: 60
```

Dekodowanie Dynamic Time Group:
```yaml
template:

  - sensor:

      - name: FoxESS Dynamic Time Group Start Time
        state: >
          {% set r = state_attr('sensor.foxess_48010_48019_dynamic_time_group_raw','registers') %}
          {{ "%02d:%02d"|format((r[1] >> 8),(r[1] & 255)) }}

      - name: FoxESS Dynamic Time Group End Time
        state: >
          {% set r = state_attr('sensor.foxess_48010_48019_dynamic_time_group_raw','registers') %}
          {{ "%02d:%02d"|format((r[2] >> 8),(r[2] & 255)) }}

      - name: FoxESS Dynamic Time Group Work Mode
        state: >

          {% set r = state_attr('sensor.foxess_48010_48019_dynamic_time_group_raw','registers') %}
          {% set v = r[3] %}

          {% if v == 1 %}Self Use
          {% elif v == 2 %}Feed In
          {% elif v == 6 %}Force Charge
          {% elif v == 7 %}Force Discharge
          {% else %}Unsupported
          {% endif %}

      - name: FoxESS Dynamic Time Group Max SOC
        unit_of_measurement: "%"
        state: >
          {{ state_attr('sensor.foxess_48010_48019_dynamic_time_group_raw','registers')[4] }}

      - name: FoxESS Dynamic Time Group Min SOC
        unit_of_measurement: "%"
        state: >
          {{ state_attr('sensor.foxess_48010_48019_dynamic_time_group_raw','registers')[5] }}

      - name: FoxESS Dynamic Time Group FD SOC
        unit_of_measurement: "%"
        state: >
          {{ state_attr('sensor.foxess_48010_48019_dynamic_time_group_raw','registers')[6] }}

      - name: FoxESS Dynamic Time Group FD Power
        unit_of_measurement: W
        state: >
          {{ state_attr('sensor.foxess_48010_48019_dynamic_time_group_raw','registers')[7] }}
```

Parametry EMS (sterowanie):
```yaml
input_number:

  foxess_ems_max_soc:
    name: EMS Max SOC
    min: 10
    max: 100
    step: 1
    unit_of_measurement: "%"

  foxess_ems_min_soc:
    name: EMS Min SOC
    min: 10
    max: 100
    step: 1
    unit_of_measurement: "%"

  foxess_ems_fd_soc:
    name: EMS Force Discharge SOC
    min: 10
    max: 100
    step: 1
    unit_of_measurement: "%"

  foxess_ems_fd_power:
    name: EMS Discharge Power
    min: 0
    max: 10000
    step: 100
    unit_of_measurement: W
```

Analiza ceny energii:
```yaml
template:

  - sensor:

      - name: EMS Price Level

        state: >

          {% set price = states('sensor.energy_price')|float %}

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

          {% set soc = states('sensor.battery_soc')|float %}

          {% if soc < 20 %}
            critical

          {% elif soc > 90 %}
            full

          {% else %}
            normal
          {% endif %}
```

Główna strategia EMS:
```yaml
template:

  - sensor:

      - name: EMS Energy Strategy

        state: >

          {% set price = states('sensor.ems_price_level') %}
          {% set battery = states('sensor.ems_battery_state') %}
          {% set pv = states('sensor.pv_power')|float %}

          {% if price == 'cheap' and battery != 'full' %}
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
      start_hour:
      start_min:
      end_hour:
      end_min:
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

            - "{{ enable }}"
            - "{{ (start_hour << 8) + start_min }}"
            - "{{ (end_hour << 8) + end_min }}"
            - "{{ work_mode }}"
            - "{{ max_soc }}"
            - "{{ min_soc }}"
            - "{{ fd_soc }}"
            - "{{ fd_power }}"
            - 0
            - 0
```

Automatyczny kontroler EMS:
```yaml
automation:

  - alias: EMS FoxESS Controller

    trigger:

      - platform: state
        entity_id:

          - sensor.ems_energy_strategy
          - sensor.ems_foxess_mode

    action:

      - service: script.foxess_dynamic_time_group_write

        data:

          enable: 1

          start_hour: 0
          start_min: 0

          end_hour: 23
          end_min: 59

          work_mode: "{{ states('sensor.ems_foxess_mode')|int }}"

          max_soc: "{{ states('input_number.foxess_ems_max_soc')|int }}"
          min_soc: "{{ states('input_number.foxess_ems_min_soc')|int }}"
          fd_soc: "{{ states('input_number.foxess_ems_fd_soc')|int }}"
          fd_power: "{{ states('input_number.foxess_ems_fd_power')|int }}"
```

Self Use:
```yaml
service: script.foxess_dynamic_time_group_write

data:

  enable: 1
  start_hour: 0
  start_min: 0
  end_hour: 23
  end_min: 59

  work_mode: 1

  max_soc: 100
  min_soc: 20
  fd_soc: 20
  fd_power: 0
```

Force Charge:
```yaml
service: script.foxess_dynamic_time_group_write

data:

  enable: 1
  start_hour: 0
  start_min: 0
  end_hour: 23
  end_min: 59

  work_mode: 6

  max_soc: 90
  min_soc: 20
```

Force Discharge:
```yaml
service: script.foxess_dynamic_time_group_write

data:

  enable: 1
  start_hour: 0
  start_min: 0
  end_hour: 23
  end_min: 59

  work_mode: 7

  fd_soc: 30
  fd_power: 5000
```

Przykład: ładowanie z taniej taryfy
```yaml
service: script.foxess_dynamic_time_group_write

data:

  enable: 1

  start_hour: 0
  start_min: 0

  end_hour: 6
  end_min: 0

  work_mode: 6

  max_soc: 90
  min_soc: 20

  fd_soc: 20
  fd_power: 5000
```

Przykład: eksport przy drogiej energii
```yaml
service: script.foxess_dynamic_time_group_write

data:

  enable: 1

  start_hour: 17
  start_min: 0

  end_hour: 21
  end_min: 0

  work_mode: 7

  max_soc: 100
  min_soc: 30

  fd_soc: 30
  fd_power: 5000
```
