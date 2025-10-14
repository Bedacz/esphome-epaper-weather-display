# ESPHome E-Paper Weather Display

A battery-powered, deep-sleeping E-Paper weather station built with an ESP32-C6 and ESPHome, pulling all data from Home Assistant.

![ESPHome E-Paper Weather Display](https://github.com/user-attachments/assets/40755404-2f58-46b8-8729-94c887d0f0c9)

---

## Features

- **5-Day Weather Forecast:** Displays the weather condition, min/max temperatures, and precipitation for the next five days.
- **Current Conditions:** Shows current outdoor temperature and humidity.
- **Power Efficient:** Utilizes deep sleep to maximize battery life, lasting for weeks or even months on a single charge.
- **Dynamic Night Sleep:** Automatically calculates the exact sleep duration overnight to wake up at a predefined time (e.g., 6 AM), saving even more power.
- **Smart Power Management:** Displays the battery level and a `+5V` indicator when charging. While charging, the device automatically stays awake and disables deep sleep. This makes it ideal for performing OTA (Over-the-Air) updates or flashing new firmware via a cable without interruption.
- **Home Assistant Integration:** All data is fetched directly from a Home Assistant instance.
- **Internationalized:** The script is fully commented in English and supports easy language switching for day names via the `substitutions` block.

## Hardware

- **MCU:** [ESP32-C6 super mini](https://www.espboards.dev/esp32/esp32-c6-super-mini/) (or any other compatible ESP32 variant).
- **Display:** [WeAct 2.9" E-Paper Module.](https://github.com/WeActStudio/WeActStudio.EpaperModule)
- **Power:** LiPo battery.
- **Resistors:** Voltage divider networks for battery and 5V sensing

## Wiring Diagram

<img width="606" height="467" alt="Circuit" src="https://github.com/user-attachments/assets/19487e7f-9607-44df-a915-0ed025fd224b" />

```
Power & Dividers (left)                             ESP32 (center)             E-Ink Display (right)
───────────────────────────                   ────────────────────────      ─────────────────────────────
                                                ┌──────────────────┐            WeAct 2.9"
                                                │      ESP32 C6    │            (3.3V logic)
     |────────|─────────────────────────────────┤◄─ 5V             │
  [20kΩ]   [20kΩ]                               │                  │
     |        |                                 │  GPIO17 ──────────────── CLK  ─────────────► SCK
     |    ──● Node A (GPIO4 sense) ─────────────┤◄─ GPIO4 (ADC 5V)|        (SPI clock)
     |        |                                 │                  |
     |     [30kΩ]                               │  GPIO14 ──────────────── MOSI ─────────────► DIN
     |        |                                 │                  |    (SPI data out)
     |       GND                                │                  |
     |                                          │  GPIO20 ───────────────── CS   ─────────────► CS
     |────────|                                 │                  |
          ──● Node B (GPIO7 wake) ──────────────┤◄─ GPIO7 (wakeup)|
              |                                 │                  |
           [30kΩ]                               │  GPIO19 ───────────────── DC   ─────────────► D/C
              |                                 │                  |
             GND                                │  GPIO18 ───────────────── RST  ─────────────► RST
                                                │                  |
      4.2V (LiPo)                               │  GPIO16  ◄──────────────── BUSY ─────────────◄ BUSY
              |                                 │                  |
           [36kΩ]                               │  3V3  ─────────────────── VCC  ─────────────► VCC (3.3V)
              |                                 │  GND  ─────────────────── GND  ─────────────► GND
          ──● Node C (GPIO3 batt) ──────────────┤◄─ GPIO3 (ADC batt)
              |                                 │                  |
           [100kΩ]                              │  GPIO15 ──► LED (Awake) ──►───┐
              |                                 │                  |       (LED)|
            GND                                 └──────────────────┘            └───[200Ω] GND

Legend:
  [value]   = resistor
  ● Node A  = divider tap for 5V sense (GPIO4)
  ● Node B  = divider tap / wake input (GPIO7)
  ● Node C  = divider tap for LiPo sense (GPIO3)
  LED       = “Awake LED” driven by GPIO15 (add a series resistor if needed)
```
* GPIO7 doubles as a wake pin for deep sleep. GPIO15 drives an LED to indicate awake state. Both battery and 5V input are monitored via resistor dividers.

## Power Management
The display is battery-powered and relies on ESPHome's deep sleep functionality:

- **Runs for ~1 minute** per wake cycle (configurable)
- **Daytime** sleep of ~10 minutes between updates
- **Night mode** Automatically calculates the exact sleep duration overnight to wake up at a predefined time (e.g., 6 AM), saving even more power.
- **ESP32 is underclocked** to 80 MHz for power savings

## Configuration for Home Assistant
* Home Assistant no longer provides forecast attributes directly within the weather entity. Instead, you must create a template sensor to call the weather.get_forecasts service.
* Add to "configuration.yaml" file.
* weather.home - If you have a different name for the weather entity in Home Assistant, change it.
```yam
template:
  - trigger:
      - platform: time_pattern
        minutes: /5
    action:
      - service: weather.get_forecasts
        data:
          type: daily
        target:
          entity_id: weather.home
        response_variable: forecast_daily
    sensor:
      - name: weather forecast daily
        unique_id: weather_forecast_daily
        state: "{{ now().isoformat() }}"
        attributes:
          forecast: "{{ forecast_daily['weather.home']['forecast'] }}"
```
## Software & Setup

This project is built entirely using **ESPHome**.

1.  **Secrets:** Create a `secrets.yaml` file in your ESPHome configuration directory to store your Wi-Fi credentials and OTA password.
    ```yaml
    # Example secrets.yaml
    wifi_ssid: "YourWiFiName"
    wifi_password: "YourWiFiPassword"
    ota_password: "YourOtaPassword"
    wifi_ap_password: "YourFallbackPassword"
    ```

2.  **Substitutions:** Modify the `substitutions` block at the top of the `.yaml` file to match your Home Assistant entity IDs and preferred settings.
    ```yaml
    substitutions:
      # ...
      temp_entity: sensor.outside_temperature
      humidity_entity: sensor.outside_humidity
      forecast_entity: sensor.weather_forecast_daily
    ```

3.  **Fonts:** Make sure to place the `materialdesignicons-webfont.ttf` file inside a `fonts` folder within your ESPHome configuration directory for this device.
- https://github.com/Templarian/MaterialDesign-Webfont

## Credits and Inspiration

- https://github.com/whitakerz/Battery-ESP32
- https://github.com/esphome/feature-requests/issues/1109#issuecomment-1059808670
- https://github.com/kotope/esphome_eink_dashboard
- https://gist.github.com/Plawasan/4ae826b05aaa7812f3a191714ca47a50
- https://github.com/hanspeda/esphome_homeassistant_display.git
- https://github.com/niahane/forecast-thermostat
- https://tatageek.blog/2022/01/28/predpoved-pocasi-na-epaper-pomoci-esphome/
- And many others from the ESPHome community.



## License

This project is licensed under the **GPL-3.0 license**. See the `LICENSE` file for details.
