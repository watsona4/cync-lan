# pycync_lan (cync_lan)

:warning: **DNS redirection required** :warning:

Async MQTT LAN controller for Cync/C by GE devices. **Local** only control of **most** Cync devices via Home Assistant.

**This is a work in progress, and may not work for all devices.** See [known devices](docs/known_devices.md) for more information.

Forked from [cync-lan](https://github.com/iburistu/cync-lan) and [cync2mqtt](https://github.com/juanboro/cync2mqtt) - All credit to [iburistu](https://github.com/iburistu) and [juanboro](https://github.com/juanboro)

## Prerequisites

- Cync account with devices added and configured
- MQTT broker (I recommend EMQX)
- Create self-signed SSL certs using `CN=*.xlink.cn` for the server. You can use the `create_certs.sh` script.
- A minimum of 1, non battery powered, Wi-Fi Cync device to act as the TCP <-> BT bridge. I recommend a plug or always powered wifi bulb (wired switch not tested yet) - *The wifi device' can control BT only bulbs*
- DNS override/redirection for `cm.gelighting.com` or `cm-ge.xlink.cn` to a local host that will run this script.

The only way local control will work is by re-routing DNS traffic from the Cync cloud server to your local network, you'll need some 
way to override DNS - a local DNS server; OPNsense/pfSense running unbound, Pi-Hole, etc. See the [DNS docs](docs/DNS.md) for more information.

## Installation
### Docker
A multi-arch image is available. It is based on `python:3.12.2-slim-bookworm` with a total size of < 60 MB. 
Images are hosted on Docker Hub and GitHub Container Registry. New images are automated using releases as a workflow trigger.

```bash
# docker.io or ghcr.io
docker pull baudneo/cync-lan:latest
```

Supported Architectures:
- `linux/arm/v7`
- `linux/arm64`
- `linux/amd64`

:warning: It is required to first **export a config from the cloud** and then bind mount the config into the volume. 
Please see the [docker-compose.yaml](./docker-compose.yaml) file for an example and the 
[export config](#export-config-from-cync-cloud-api) section for export instructions. :warning:

If your architecture is not supported, you can build the image yourself using the provided [Dockerfile](./Dockerfile).

```bash
docker build -t cync-lan:custom .
```

### Virtualenv
To install on your system instead of using the docker image, a python virtualenv is highly recommended.
You will also need to configure your own systemd `.service` or similar file to automate the script running on system boot.

System packages you will need (package names are from a debian based system):
- `openssl`
- `git`
- `python3`
- `python3-venv`
- `python3-pip`
- `python3-setuptools`
- You may also want `dig` and `socat` for **debugging**.

```bash
# Create dir for project and venv
mkdir ~/cync-lan && cd ~/cync-lan
python3 -m venv venv
# activate the venv
source ~/cync-lan/venv/bin/activate

# create self-signed key/cert pair, wget the bash script and execute
wget https://raw.githubusercontent.com/baudneo/cync-lan/python/create_certs.sh
bash ./create_certs.sh

# install python deps
pip install pyyaml requests uvloop wheel
pip install git+https://github.com/Yakifo/amqtt.git

# wget file
wget https://raw.githubusercontent.com/baudneo/cync-lan/python/src/cync-lan.py

# Run script to export cloud device config to ./cync_mesh.yaml
# It will ask you for email, password and the OTP emailed to you.
# --save-auth flag will save the auth data to its own file (./auth.yaml)
# You can supply the auth file in future export commands using -> export ./cync_mesh.yaml --auth ./auth.yaml
python3 ~/cync-lan/cync-lan.py export ~/cync-lan/cync_mesh.yaml --save-auth

# edit cync_mesh.yaml for your MQTT broker (mqtt_url:)

# Run the script to start the server, provide the path to the config file
# You can add --debug to enable debug logging
python3 ~/cync-lan/cync-lan.py run ~/cync-lan/cync_mesh.yaml
```

## Env Vars

| Variable     | Description                                  | Default                            |
|--------------|----------------------------------------------|------------------------------------|
| `MQTT_URL`   | URL of MQTT broker                           | `mqtt://homeassistant.local:1883/` |
| `CYNC_DEBUG` | Enable debug logging                         | `0`                                |
| `CYNC_CERT`  | Path to cert file                            | `certs/server.pem`                 |
| `CYNC_KEY`   | Path to key file                             | `certs/server.key`                 |
| `CYNC_PORT`  | Port to listen on                            | `23779`                            |
| `CYNC_HOST`  | Host to listen on                            | `0.0.0.0`                          |
| `CYNC_TOPIC` | MQTT topic                                   | `cync-lan`                         |
| `HASS_TOPIC` | Home Assistant topic                         | `homeassistant`                    |
| `MESH_CHECK` | Interval to check for online/offline devices | `30`                               |


## Re-routing / Overriding DNS
See [DNS docs](docs/DNS.md) for more information.

:warning: **After freshly redirecting DNS: Devices that are currently talking to Cync cloud will need to be power cycled before they make a DNS request and connect to the local server.** :warning:

As long as your DNS is correctly re-routed, you should be able to start the server and see devices connecting to it.
If you do not see any Cync devices connecting after power cycling them, you may need to check your DNS re-routing.

## Config file
:warning: **The config file will override any environment variables set.** :warning:

For the best user experience, query the Cync cloud API to export all your homes and the devices in each home. 
This requires your email, password and the code that will be emailed to you during export.

If you add or remove devices, you will need to re-export the config file and restart the server.

See the example [config file](./cync_mesh_example.yaml)

### Export config from Cync cloud API
There is an `export` sub command that will query the Cync cloud API and export the devices to a YAML file.

```bash
# If you used --save-auth in a previous export command, you can use --auth to 
# provide the auth file to skip asking for your credentials and OTP
python3 cync-lan.py export ./cync_mesh.yaml
```

### Manually adding devices
To manually add devices to the config file, look at the example and follow the template. 
From what I have seen the device ID starts at 1 and increments by 1 for each device added to the "home" 
(it follows the order you added the bulbs).

*It is unknown how removing a device and adding a device may affect the ID number, YMMV. 
Be careful when manually adding devices.*

:warning: By manually adding, I mean you added a device via the app and did not re-export a new config.

## Controlling devices

Devices are controlled by MQTT messages. This was designed to be used with Home Assistant, but you can use 
any MQTT client to send messages to the MQTT broker.

**Please see [Home Assistant MQTT documentation](https://www.home-assistant.io/integrations/light.mqtt/#json-schema) 
for more information on JSON payloads.** This repo will try to stay up to date with the latest Home Assistant MQTT JSON schema.

## Home Assistant

This script uses the MQTT discovery mechanism in Home Assistant to automatically add devices.
You can control the Home Assistant MQTT topic via the environment variable `HASS_TOPIC`.

## Debugging / socat

If the devices are not responding to commands, it's likely that the TCP communication on your
device is different. You can either open an issue and I can walk you through getting good debug logs, 
or you can use `socat` to inspect the traffic of the device communicating with the cloud server in real-time yourself by running:

```bash
# Older firmware devices
socat -d -d -lf /dev/stdout -x -v 2> dump.txt ssl-l:23779,reuseaddr,fork,cert=certs/server.pem,verify=0 openssl:34.73.130.191:23779,verify=0
# Newer firmware devices (Notice the last IP change)
sudo socat -d -d -lf /dev/stdout -x -v 2> dump.txt ssl-l:23779,reuseaddr,fork,cert=certs/server.pem,verify=0 openssl:35.196.85.236:23779,verify=0
```
In `dump.txt` you will see the back-and-forth communication between the device and the cloud server.
`>` is device to server, `<` is server to device.

Also, make sure to check that your DNS is actually routing to your local network. You can check by using `dig`:

 ```bash
# Older firmware
dig cm-ge.xlink.cn

# Newer firmware
dig cm.gelighting.com

# Example output with a local A record returned
; <<>> DiG 9.18.24 <<>> cm.gelighting.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 56237
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;cm.gelighting.com.             IN      A

;; ANSWER SECTION:
cm.gelighting.com.      3600    IN      A       10.0.1.9

;; Query time: 0 msec
;; SERVER: 10.0.1.1#53(10.0.1.1) (UDP)
;; WHEN: Fri Mar 29 08:26:51 MDT 2024
;; MSG SIZE  rcvd: 62
```

If the DNS request returns a non-local IP, DNS override is not set up correctly.

:warning: **If you are using selective DNS override via `views` in `unbound`, and you did not set up an override for your PC's IP,
your dig command will still return the Cync cloud IP. This is normal.**


# Power cycle devices after DNS re-route
Devices make a DNS query on first startup (or after a network loss, like AP reboot) - 
you need to power cycle all devices that are currently connected to the Cync cloud servers 
before they request a new DNS record and will connect to the local controller.
