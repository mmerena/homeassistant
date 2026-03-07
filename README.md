# homeassistant
Dynamic Time Group

Blok RAW:
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

      - name: FoxESS 48010-48019 Time & Control Raw Block
        unique_id: foxess_48010_48019_time_control_raw_block
        slave: 247
        address: 48010
        input_type: holding
        data_type: custom
        structure: ">10H"
        count: 10
        scan_interval: 60
```
Mapowanie rejestrów:
```text
| idx | rejestr | nazwa         | typ | opis                                                                   |
| --- | ------- | ------------- | --- | ---------------------------------------------------------------------- |
| 0   | 48010   | Enable        | U16 | 0 disable / 1 enable                                                   |
| 1   | 48011   | Start Time    | U16 | High byte = hour, Low byte = minute                                    |
| 2   | 48012   | End Time      | U16 | High byte = hour, Low byte = minute                                    |
| 3   | 48013   | Work Mode     | U16 | 1 Self-Use, 2 Feed-In, 3 Backup, 4 Peak Shaving, 6 Charge, 7 Discharge |
| 4   | 48014   | MaxSoC        | U8  | 10-100%                                                                |
| 5   | 48015   | MinSoC OnGrid | U8  | 10-100%                                                                |
| 6   | 48016   | FDSOC         | U16 | SOC dla Force Discharge                                                |
| 7   | 48017   | FDPWR         | U16 | moc W                                                                  |
| 8   | 48018   | reserved      | U16 | —                                                                      |
| 9   | 48019   | reserved      | U16 | —                                                                      |
```

Dekodowanie:
```yaml
template:
  - sensor:

      - name: FoxESS Dynamic Time Group Enabled
        unique_id: foxess_dynamic_time_group_enabled
        state: >
          {{ state_attr('sensor.foxess_48010_48019_time_control_raw_block','registers')[0] }}

      - name: FoxESS Dynamic Time Group Start Time
        unique_id: foxess_dynamic_time_group_start_time
        state: >
          {% set r = state_attr('sensor.foxess_48010_48019_time_control_raw_block','registers') %}
          {% set h = (r[1] >> 8) %}
          {% set m = (r[1] & 255) %}
          {{ "%02d:%02d"|format(h,m) }}

      - name: FoxESS Dynamic Time Group End Time
        unique_id: foxess_dynamic_time_group_end_time
        state: >
          {% set r = state_attr('sensor.foxess_48010_48019_time_control_raw_block','registers') %}
          {% set h = (r[2] >> 8) %}
          {% set m = (r[2] & 255) %}
          {{ "%02d:%02d"|format(h,m) }}


      - name: FoxESS Dynamic Time Group Work Mode
        unique_id: foxess_dynamic_time_group_work_mode
        state: >

          {% set r = state_attr('sensor.foxess_48010_48019_time_control_raw_block','registers') %}
          {% set v = r[3] %}

          {% if v == 1 %}Self Use
          {% elif v == 2 %}Feed In
          {% elif v == 3 %}Backup
          {% elif v == 4 %}Peak Shaving
          {% elif v == 6 %}Force Charge
          {% elif v == 7 %}Force Discharge
          {% else %}Unknown
          {% endif %}

      - name: FoxESS Dynamic Time Group Max SOC
        unit_of_measurement: "%"
        state: >
          {{ state_attr('sensor.foxess_48010_48019_time_control_raw_block','registers')[4] }}

      - name: FoxESS Dynamic Time Group Min SOC OnGrid
        unit_of_measurement: "%"
        state: >
          {{ state_attr('sensor.foxess_48010_48019_time_control_raw_block','registers')[5] }}

      - name: FoxESS Dynamic Time Group Force Discharge SOC
        unit_of_measurement: "%"
        state: >
          {{ state_attr('sensor.foxess_48010_48019_time_control_raw_block','registers')[6] }}

      - name: FoxESS Dynamic Time Group Discharge Power
        unit_of_measurement: W
        state: >
          {{ state_attr('sensor.foxess_48010_48019_time_control_raw_block','registers')[7] }}

```

Zapis Dynamic Time Group:
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
