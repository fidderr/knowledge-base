# HomeWizard P1: set up Wi-Fi without the app or an account (fully local, no cloud)

**Problem:** A new HomeWizard P1 meter (P1 dongle) normally has to be put on
your Wi-Fi with the HomeWizard Energy phone app. You may not want to install
the app, make an account, or let the device talk to the HomeWizard cloud.

**Result:** You can join the P1 to your Wi-Fi from any computer with two HTTPS
requests, give it a fixed IP, add it to Home Assistant, and switch the cloud
off. No app, no account.

## Tested on

- HomeWizard P1 meter (P1 dongle), firmware **6.0208**
- Home Assistant 2026.10, built-in HomeWizard integration
- One device, one attempt that worked. Other firmware versions may differ.

The two requests below were found by reading how the official Android app
(HomeWizard Energy 2.6.x) provisions a device in setup mode. Nothing here
bypasses a login or a cloud check: the device accepts the request itself.

## What you need

- The P1 plugged into the meter's P1 port, and a computer with Wi-Fi
- Your Wi-Fi name and password. **2.4 GHz only**, the P1 does not do 5 GHz
- `curl` (built into Windows 10/11, macOS and Linux). In Windows PowerShell type `curl.exe`, because plain `curl` is an alias for something else

## Steps

1. **Enter setup mode.** Hold the button on the P1 until the light ring blinks
   blue. The device now broadcasts an open setup network called
   `P1meterSetup` (default password `SetupPass`).
2. **Join that network** with your computer. You will get `192.168.4.x`; the
   P1 is `192.168.4.1`. Only HTTPS on port 443 is available and the
   certificate is self-signed, so use `-k` / ignore certificate errors. *Windows tip:* PowerShell mangles quotes in inline JSON. Save each JSON body in a file (for example `challenge.json`) and use `curl.exe -sk -X POST URL -H "Content-Type: application/json" -d "@challenge.json"` on one line.
3. **Send a challenge.** The device accepts any random hex string:

   ```sh
   curl -sk -X POST https://192.168.4.1/api/challenge \
     -H 'Content-Type: application/json' \
     -d '{"challenge":"00112233445566778899aabbccddeeff","response":null}'
   ```

   Expected: HTTP 200 with the same challenge and a `response` value.
4. **Send your Wi-Fi details:**

   ```sh
   curl -sk -X POST https://192.168.4.1/wifi/connect \
     -H 'Content-Type: application/json' \
     -d '{"ssid":"YOUR_SSID","password":"YOUR_WIFI_PASSWORD"}'
   ```

   Expected: `{"status":"connecting","ssid":"YOUR_SSID"}`.
5. **Reconnect your computer to your normal Wi-Fi.** The P1 leaves setup mode
   and joins your network. Find it in your router's client list; it is named
   `HW-p1meter-xxxxxx`.
6. **Give it a fixed IP** with a DHCP reservation in your router, so Home
   Assistant keeps finding it.
7. **Add it to Home Assistant:** Settings → Devices & services → HomeWizard.
   When asked, press the button on the P1 once (this authorizes Home
   Assistant; it must be done within about a minute).
8. **Turn the cloud off.** In the HomeWizard integration, switch off
   *Cloud connection* (entity `switch.<device>_cloud_connection`). The device
   then no longer talks to HomeWizard servers.

Check your Wi-Fi password for typos first: if it
is wrong the P1 simply never appears on your network, and you repeat from step 1.

## Pitfalls (these cost us a crash)

- **Do not scan or probe the device in setup mode.** Guessing URL paths,
  running a port scan or opening many connections at once made the P1 hang
  and reboot. Send exactly the two requests above, one connection each.
- Other paths in setup mode (`/api/system`, `/wifi`, …) return 404. Do not
  hunt for them.
- 5 GHz-only or "same SSID on 2.4 and 5 GHz with band steering" networks can
  make the join fail. A plain 2.4 GHz network is the safe choice.
- Putting the P1 back into setup mode, or a factory reset, switches the cloud
  connection **on** again. Turn it off afterwards.
- With the cloud off the device will not get firmware updates, and the
  official app can no longer reach it.
- The local API (version 1) is open to everyone on your LAN. Use a trusted
  network, and keep IoT devices off the guest network if you want discovery
  to work.

## Why this works

In setup mode the P1 exposes a small HTTPS API for its own provisioning. The
app only sends the Wi-Fi name and password there. The challenge step does not
require a HomeWizard account or an internet connection; the device accepts a
challenge you made up yourself. Everything after that (Home Assistant
pairing and local API) is HomeWizard's documented local API.

## Reports

Worked or failed on another firmware, another HomeWizard device (water meter,
kWh meter, energy socket)? Open an issue with the model and firmware version.
