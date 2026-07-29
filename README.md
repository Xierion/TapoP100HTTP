# Tapo P100 HTTP bridge for Moonraker

Moonraker has no built-in Tapo support, but its `power` component supports
a generic `type: http` device that hits arbitrary URLs. This bridge is a
small Flask server that translates those requests into calls against the
[almottier/TapoP100](https://github.com/almottier/TapoP100) (`PyP100`)
library.

## 1. Copy the files to the Pi

As user `streamer`:

```bash
mkdir -p ~/tapo_http_bridge
# copy app.py, config.example.json, requirements.txt, tapo-bridge.service
# into ~/tapo_http_bridge
cd ~/tapo_http_bridge
```

## 2. Create a venv and install deps

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
deactivate
```

## 3. Configure your plug + Tapo account

```bash
cp config.example.json config.json
nano config.json      # fill in ip, email, password
chmod 600 config.json # it's plaintext, so lock it down
```

`ip` is the plug's local IP (give it a DHCP reservation on your router so
it doesn't move). `email`/`password` are your TP-Link/Tapo **account**
credentials (the ones you use in the Tapo app), not a device password.

## 4. Quick manual test (before installing the service)

```bash
source venv/bin/activate
python3 app.py
```

In another terminal:

```bash
curl http://127.0.0.1:5111/status
curl http://127.0.0.1:5111/on
curl http://127.0.0.1:5111/off
```

You should get back `{"status": "on"}` / `{"status": "off"}`. Ctrl-C the
server once this works.

## 5. Install as a systemd service

```bash
sudo cp tapo-bridge.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now tapo-bridge.service
sudo systemctl status tapo-bridge.service
```

Logs: `journalctl -u tapo-bridge.service -f`

## 6. Add it to moonraker.conf

Append to `~/printer_data/config/moonraker.conf` (adjust the section name
`[power tapo_plug]` to whatever you want it called in Mainsail/Fluidd):

```ini
[power tapo_plug]
type: http
on_url: http://127.0.0.1:5111/on
off_url: http://127.0.0.1:5111/off
status_url: http://127.0.0.1:5111/status
response_template:
    {% set resp = http_request.last_response().json() %}
    {resp["status"]}
locked_while_printing: True
off_when_shutdown: False
```

Optional extras you may want (see Moonraker's power docs):

```ini
restart_klipper_when_powered: True
on_when_job_queued: True
```

Then restart Moonraker:

```bash
sudo systemctl restart moonraker
```

You should now see a power toggle for `tapo_plug` in Mainsail/Fluidd.

## Notes

- The bridge only listens on `127.0.0.1`, since Moonraker runs on the same
  Pi and the config file contains your Tapo account password. Don't expose
  port 5111 externally.
- Requests to the plug are serialized with a lock, since the Tapo protocol
  doesn't handle concurrent sessions well.
- If `pip install` for `PyP100` fails, make sure `git` is installed
  (`sudo apt install git`) since it's installed directly from GitHub.
