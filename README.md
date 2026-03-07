# FoxESS EMS for Home Assistant

Profesjonalny **Energy Management System (EMS)** dla falowników
**FoxESS** oparty o **Home Assistant + Modbus TCP**.

System zarządza:

-   produkcją PV
-   magazynem energii
-   zakupem energii z sieci (taryfa G13)
-   sprzedażą energii
-   trybem pracy falownika

Architektura jest **modułowa**, dzięki czemu system można łatwo
rozwijać.

------------------------------------------------------------------------

# Architektura EMS

    FoxESS Modbus
          │
    EMS Sources
          │
    EMS Analysis
          │
    Energy State Engine
          │
    Tariff Engine (G13)
          │
    Dynamic Battery SOC
          │
    EMS Strategy Engine
          │
    FoxESS Controller

------------------------------------------------------------------------

# Struktura projektu

Pliki znajdują się w katalogu:

    config/packages/

## Lista plików

    foxess_modbus.yaml
    ems_sources.yaml
    ems_analysis.yaml
    ems_energy_state.yaml
    ems_tariff_g13.yaml
    ems_dynamic_soc.yaml
    ems_strategy.yaml
    ems_control.yaml
    ems_helpers.yaml

------------------------------------------------------------------------

# FoxESS Modbus

Komunikacja z falownikiem przez **Modbus TCP**.

Odczytywane dane:

  sensor                  opis
  ----------------------- ----------------
  foxess_pv_total_power   produkcja PV
  foxess_grid_power       moc sieci
  foxess_load_power       zużycie domu
  foxess_battery_power    moc baterii
  foxess_battery_soc      poziom baterii

------------------------------------------------------------------------

# EMS Sources

Warstwa abstrakcji nad danymi falownika.

    sensor.pv_power
    sensor.load_power
    sensor.grid_power
    sensor.battery_power
    sensor.battery_soc

------------------------------------------------------------------------

# EMS Analysis

Oblicza parametry energetyczne systemu.

    sensor.ems_pv_surplus
    sensor.ems_pv_surplus_level
    sensor.ems_battery_state
    sensor.ems_grid_state
    sensor.ems_power_balance

------------------------------------------------------------------------

# Energy State Engine

Kluczowy sensor:

    sensor.ems_energy_state

Możliwe stany:

    grid_import
    pv_charge
    pv_export
    battery_support
    grid_export
    balanced

------------------------------------------------------------------------

# Tariff Engine (G13)

  czas           taryfa
  -------------- --------
  Weekend        cheap
  00:00--07:00   cheap
  07:00--13:00   normal
  13:00--16:00   cheap
  16:00--21:00   peak
  21:00--24:00   cheap

Sensor:

    sensor.energy_tariff_g13

------------------------------------------------------------------------

# Dynamic Battery SOC

  czas     max SOC
  -------- ---------
  00--06   90
  06--10   70
  10--15   95
  15--18   100
  18--23   90
  23--24   80

Sensor:

    sensor.ems_dynamic_max_soc

------------------------------------------------------------------------

# EMS Strategy Engine

Decyzje EMS:

    charge
    self_use
    export
    discharge

------------------------------------------------------------------------

# FoxESS Controller

Zapisywany blok rejestrów:

    48010–48019

Mechanizmy bezpieczeństwa:

-   write-if-changed
-   debounce 5 minut

------------------------------------------------------------------------

# Helpery

    input_number.foxess_ems_min_soc
    input_number.foxess_ems_fd_soc
    input_number.foxess_ems_fd_power
    input_text.foxess_last_signature
    input_datetime.ems_last_mode_change

------------------------------------------------------------------------

# Cechy systemu

✔ inteligentne ładowanie baterii\
✔ wykorzystanie nadwyżek PV\
✔ obsługa taryfy G13\
✔ dynamiczne sterowanie SOC\
✔ stabilne przełączanie trybów falownika\
✔ minimalizacja zapisów Modbus

------------------------------------------------------------------------

# Wymagania

-   Home Assistant
-   integracja Modbus
-   falownik FoxESS
-   dostęp Modbus TCP

------------------------------------------------------------------------

# Status projektu

    EMS Core: Completed
    FoxESS Integration: Completed
    Strategy Engine: Completed
    Control Layer: Completed
