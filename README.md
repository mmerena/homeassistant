# homeassistant
Dynamic Time Group

yaml'''
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
'''
