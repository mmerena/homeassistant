# System EMS - Zarządzanie Energią w Home Assistant

## Wstęp

System EMS (Energy Management System) to zaawansowane rozwiązanie do zarządzania energią w środowisku Home Assistant. Jest zaprojektowany do sterowania falownikiem FoxESS P3-10.0-SH z magazynem energii EQ4800 o pojemności 30 kWh. System podejmuje decyzje o trybie pracy (ładowanie, rozładowywanie, eksport) na podstawie cen sprzedaży energii RCE (Rynkowa Cena Energii) oraz cen zakupu w taryfie dostawcy energii.

Główne funkcje:
- Adaptacyjne prognozowanie produkcji PV i zużycia obciążenia.
- Arbitraż cenowy: Ładowanie baterii przy niskich cenach, rozładowywanie/eksport przy wysokich.
- Integracja z Modbus TCP dla sterowania falownikiem.
- Diagnostyka, monitorowanie i optymalizacja z uwzględnieniem jutrzejszych cen RCE.
- Symulacja trybów dla testów.

System składa się z plików YAML definiujących sensory, inputy, automatyzacje i template'y w Home Assistant.

## Wymagania

- Home Assistant (wersja core-2023.1 lub nowsza zalecana).
- Integracja Modbus TCP dla falownika FoxESS.
- Sensory zewnętrzne:
  - SQL: `rce_prices_today`, `rce_prices_tomorrow`, `local_supplier_tariff` (dostarczają ceny RCE i taryfę dostawcy).
  - Modbus: Sensory FoxESS.
- Pakiety HA: Template, Input Number/Boolean/Select/Text/Datetime.

## Instalacja

1. Skopiuj pliki YAML do katalogu konfiguracyjnego HA (np. `config/packages/ems/`).
2. Dodaj do `configuration.yaml`:
```yaml
homeassistant:
packages: !include_dir_named packages
```
3. Zrestartuj Home Assistant.
4. Skonfiguruj inputy w UI HA.

Pliki systemu:
- `ems_adaptive_learning.yaml`: Adaptacyjne prognozy PV i obciążenia.
- `ems_analysis.yaml`: Analiza nadwyżki PV, stanu baterii.
- `ems_battery_planner.yaml`: Planowanie rezerw baterii.
- `ems_control.yaml`: Sterowanie falownikiem via Modbus.
- `ems_control_panel.yaml`: Panel sterowania (override'y).
- `ems_diagnostics.yaml`: Diagnostyka zdrowia systemu.
- `ems_energy_flow.yaml`: Przepływy energii.
- `ems_energy_state.yaml`: Stan energii.
- `ems_export_price.yaml`: Ceny eksportu.
- `ems_grid_arbitrage.yaml`: Arbitraż sieciowy.
- `ems_helpers.yaml`: Inputy i automatyzacje pomocnicze.
- `ems_optimizer.yaml`: Optymalizator decyzji.
- `ems_prediction_engine.yaml`: Silnik predykcji.
- `ems_profit_engine.yaml`: Silnik zysków.
- `ems_profit_monitor.yaml`: Monitor zysków.
- `ems_sources.yaml`: Źródła danych (PV, load, grid, battery).
- `ems_stability_layer.yaml`: Stabilność strategii.
- `ems_strategy.yaml`: Strategia energii.
- `ems_system_monitor.yaml`: Monitor systemu.
- `ems_tariff_supplier.yaml`: Taryfa dostawcy.
- `rce_prices.yaml`: Ceny RCE.

## Konfiguracja

- **Inputy**: Ustaw w UI HA, np. prognozy PV/load, progi arbitrażu, ceny taryf.
- **Automatyzacje**:
- `ems_foxess_controller`: Aktualizuje tryb falownika co zmianę stanu kluczowych sensorów.
- Aktualizacje cen: Co 15 min dla RCE, co 1 min dla taryfy dostawcy.
- **Symulacja**: Włącz `input_boolean.foxess_ems_simulation` do testów bez pisania do Modbus.

## Jak Działa

1. **Źródła danych**: Sensory Modbus pobierają moc PV, load, grid, battery SOC.
2. **Prognozy**: Adaptacyjne uczenie łączy manualne inputy z rzeczywistymi danymi.
3. **Decyzje**: Na podstawie spread'u cen, prognoz, stanu baterii – wybiera tryb (charge, discharge, export, self_use).
4. **Sterowanie**: Zmienia tryb FoxESS (np. 6=charge) i SOC via Modbus.
5. **Diagnostyka**: Sensory zdrowia, stabilności, bilansu.

Uwzględnia jutrzejsze ceny RCE dla priorytetowego ładowania/eksportu.

## Bezpieczeństwo i Ograniczenia

- Nie modyfikuj safety instructions.
- Testuj w symulacji przed produkcją.
- System zakłada dobre intencje – nie wspiera niedozwolonych aktywności.

## Licencja

MIT License. Użyj na własną odpowiedzialność.
