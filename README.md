# Growatt ShineWiFi-X2 → ESPHome

Local, cloud-free monitoring for **Growatt SPF** off-grid inverters, by reflashing the **ShineWiFi-X2** stick (ESP32-C3) with ESPHome.

> **Română:** [README.ro.md](README.ro.md) · [Ghid pas cu pas](docs/GHID.ro.md)

<p>
<img src="docs/img/page-now-en.png" width="260" alt="Web page, Now tab">
<img src="docs/img/page-details-en.png" width="260" alt="Web page, Details tab">
</p>

## What you get

- The stick reads the inverter every 10 seconds over Modbus: PV strings, battery (SOC, voltage, power, energy), AC output, temperatures, status, fault/warning codes and settings (output priority, charger source, battery type, max charge current).
- Two ways to receive the data:
  - **HiveMQ Cloud + a web page** on GitHub Pages: view from anywhere, no server needed. Works on 4G without a public IP.
  - **Home Assistant** through its Mosquitto broker, with automatic MQTT discovery.
- **History kept by the stick:** the last 24 hours at one point per minute, plus daily production and consumption for the last 30 days.
- **Wi-Fi set on site:** on first boot the stick opens a **DongleGrowatt** hotspot where you choose the network from your phone.
- **Updates over Wi-Fi** from the stick's own web page, after the first flash.
- A responsive **web page** with two tabs, Now and Details, in English or Romanian. It can be added to an iPhone Home Screen like an app.

## Hardware this was built and tested on

| | |
| --- | --- |
| Stick | Growatt ShineWiFi-X2, board **B700.0198800** |
| Module | Ai-Thinker ESP-C3-13U (ESP32-C3, 4 MB embedded flash) |
| USB side | CH340, inverter link on GPIO0 (TX) / GPIO1 (RX), 115200 8N1 |
| Inverter | Growatt **SPF 6000 ES Plus** (other SPF models with the same off-grid protocol should work) |

Older ShineWiFi-X sticks use an ESP8266 and different pins; this project is for the X2.

## Start here

**→ [Step-by-step guide](docs/GUIDE.en.md)**: from backing up the Growatt firmware to the web page on your phone.

Short version:

1. Open the stick, wire a 3.3 V USB-TTL adapter to the header (`3.3V · G · TX0 · RX0 · RST · IO9 · G`), bridge IO9 to G.
2. Check that Secure Boot and Flash Encryption are disabled, then **back up the stock firmware**.
3. Create a HiveMQ Cloud cluster (or use Home Assistant's Mosquitto).
4. Copy `firmware/secrets.example.yaml` to `secrets.yaml`, fill it in, compile `firmware/growatt-dongle.yaml` with ESPHome.
5. Flash `firmware.factory.bin`, connect to the **DongleGrowatt** hotspot, choose your Wi-Fi.
6. Plug the stick into the inverter.
7. Fill in the `CONFIG` block of `web/index.html` and publish it with GitHub Pages.

## Repository

| Path | Content |
| --- | --- |
| `firmware/growatt-dongle.yaml` | main firmware (HiveMQ Cloud / any TLS broker) |
| `firmware/growatt-dongle-ha.yaml` | Home Assistant variant (local Mosquitto, MQTT discovery) |
| `firmware/secrets.example.yaml` | template for your passwords and broker |
| `firmware/diag-pin-scan.yaml` | diagnostic: shows GPIO levels, to find the UART pins |
| `firmware/diag-uart-debug.yaml` | diagnostic: logs every byte to and from the inverter |
| `web/index.html` | the web page (edit the `CONFIG` block) |
| `docs/GUIDE.en.md` · `docs/GHID.ro.md` | full guide in English and Romanian |
| `docs/registers.md` | Modbus registers and MQTT topics |

## Safety and disclaimer

You modify your own hardware at your own risk. Always keep the backup of the stock firmware; with it you can restore the stick to factory state. Never connect 5 V to the stick's header and never run `espefuse burn` commands. This project only **reads** from the inverter and never writes settings.

Not affiliated with Growatt. Growatt, ShineWiFi and SPF are trademarks of their owners.

## Author

**Barbu Dan-Cristian**. Released under the [MIT License](LICENSE).
