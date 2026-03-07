# homeassistant
Dynamic Time Group

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
Mapowanie rejestrów (Dynamic Time Group):
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
