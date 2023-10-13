![Bermuda Logo](img/logo@2x.png)

# Bermuda BLE Trilateration

- Track bluetooth devices by Area in [HomeAssistant](https://home-assistant.io/), using [ESPHome](https://esphome.io/) [bluetooth_proxy](https://esphome.io/components/bluetooth_proxy.html) devices.

- (eventually) Triangulate device positions! Maybe.

[![GitHub Release][releases-shield]][releases]
[![GitHub Activity][commits-shield]][commits]
[![License][license-shield]](LICENSE)

[![pre-commit][pre-commit-shield]][pre-commit]
[![Black][black-shield]][black]

[![hacs][hacsbadge]][hacs]
[![Project Maintenance][maintenance-shield]][user_profile]
[![BuyMeCoffee][buymecoffeebadge]][buymecoffee]

[![Discord][discord-shield]][discord]
[![Community Forum][forum-shield]][forum]

## What it does:

- Area-based device location (ie, device-level room prescence) is working reasonably well.
- Creates sensors for Area and Distance for devices you choose
- Creates `device_tracker` entities for chosen devices, which can be linked to "Person"s for Home/Away tracking
- Configurable settings for rssi reference level, environmental attenuation, max tracking radius
- Provides a json/yaml dump of devices and their distances from each bluetooth
  receiver, via the `bermuda.dump_devices` service.

## What you need:

- One or more devices providing bluetooth proxy information to HA. I like the D1-Mini32 boards because
  they're cheap and easy to deploy. The Shelly bluetooth proxy devices should also work but I don't have any
  so can't test them myself. Issue reports with debug info welcome.

- USB Bluetooth on your HA host is not ideal, since it does not timestamp the advertisement packets.
  However it can be used for simple "Home/Not Home" tracking, just not Area/Room-based tracking at this time.

- Some bluetooth BLE devices you want to track. Smart watches, beacon tiles, thermometers etc

## What it does not do

- It does not (yet) provide location relative to multiple beacons, but that is the ultimate
  goal. RSSI is extremely "noisy" as data goes, but the hope is to find ways to smooth that
  out a bit and get a workable impression of where devices and scanners are in relation to each
  other. Math / Geometry geeks are very welcome to assist, I am well out of my depth here!

- As yet it doesn't know how to handle rotating MAC addresses, hopefully it can integrate with
  the new integration that does that.

## What you won't need (if this works for you)

- Can replace `bluetooth_ble_tracker` by creating `device_tracker` entities for home/not_home
  for selected BLE devices, which can be used for Person home/away sensing.
  This is the "Zone" element of homeassistant localisation, where "home" is
  one Zone, and "work" or other large geographic areas might be others.

## How it Works

This integration uses the advertisement data gathered by your esphome or
Shelley bluetooth-proxy deployments into Homeassistant to track or ultimately
triangulate (more correctly, trilaterate) the relative positions of any BLE devices
observed around your home.

Note that this is more properly called "Tri*lateration*", as we are not
measuring the angles, but instead measuring distances. The bottom line
is that triangulation is more likely to hit people's search terms so we'll
probably bandy that term about a bit :-)

The integration gathers the advertisement data from the bluetooth integration,
and uses it to glean location/area info for all devices.

You can view the internal state by calling the `bermuda.dump_devices` service.

Currently it munges this into three entities for each tracked device:

- A `device_tracker` entity, which exposes a "home/not home" state. This entity can be mapped
  to a `person` to indicate if they are home (eg, by tracking their smart watch).
  This integration gives you two forms of presence tracking.

- An `Area` sensor. This gives the area name of the nearest bluetooth proxy. If you have a
  proxy in each room, you can use this to know which room a given device is currently in.

- A `Distance` sensor which gives the estimated distance from the nearest bluetooth proxy.
  This may help give a more relative indication of presence.

Ultimately, it is hoped to also provide a mud-map of the home, where relative positions of
proxies and devices can be visually expressed. This assumes that we can (with some level of reliability)
compute the layout with trilateration. That is, by measuring the distances between devices and scanners it
is hoped to approximate a solution for the entire "network" of devices. This won't be lidar-level accurate,
but it might mean you'd only need proxies in every second room, say.

## Screenshots

After installing, the integration should be visible in Settings, Devices & Services

![The integration, in Settings, Devices & Services](img/screenshots/integration.png)

Press the `CONFIGURE` button to see the configuration dialog. At the bottom is a field
where you can enter/list any bluetooth devices the system can see. Choosing devices
will add them to the configured devices list and creating sensor entities for them.

![Bermuda integration configuration option flow](img/screenshots/configuration.png)

Choosing the device screen shows the current sensors and other info.

![Screenshot of device information view](img/screenshots/deviceinfo.png)

The sensor information also includes attributes for which scanner detected the device,
the measured rssi etc. (these will probably be removed from the Area sensor to reduce
database history churn, but will probably stay for the Distance sensor)

![Bermuda sensor information](img/screenshots/sensor-info.png)

In Settings, People, you can define any Bermuda device to track home/away status
for any person/user.

![Assign a Bermuda sensor for Person tracking](img/screenshots/person-tracker.png)

## FAQ

### Why do my bluetooth devices have only the address and no name?

- you need to tell your bluetooth proxies to send an inquiry in response to
  advertisements. In esphome, this is done by adding `active: true` to the
  `bluetooth_proxy` section (this is separate from the active property of
  the `esp32_ble_tracker` section, which controls outbound client connections).

- Also, when you first restart homeassitant after loading the integration it may
  take a minute or so for the system to collect the names of the devices it sees.

### Isn't mmWave better?

- mmWave is definitely _faster_, but it will only tell you "someone" has entered
  a space, while Bermuda can tell you _who_ (or what) is in a space.

### What about PIR / Infrared?

- PIR is also likely faster than bluetooth, but again it only tells you that
  someone / something is present, but doesn't tell you who/what.

So how does that help?

- If the home knows who is in a given room, it can set the thermostat to their
  personal preferences, or perhaps their lighting settings. This might be
  particularly useful for testing automations on yourself before unleashing them
  on to your housemates, so they don't get annoyed while you iron out the bugs :-)

- If you have BLE tags on your pets you can have automations specifically for them,
  and/or you can exclude certain automations, for example don't trigger a light from
  an IR sensor if it knows it's just your cat, say.

### How quickly does it react?

- That will mainly depend on how often your beacon transmits advertisements, however
  right now the integration only re-calculates on a timed basis. This should be changed
  to a realtime recalculation based on incoming advertisements soon.

### How is the distance calculated?

- Currently, we use the relatively simple equation:
  `distance = 10 ** ((ref_power - rssi) / (10 * attenuation))`

  - `ref_power` is the rssi value you get when the device is 1 metre from the receiver.
    Currently you can configure this as a global setting in the options.
  - `rssi` is the "received signal strength indicator", being a measurement of RF power
    expressed in dB. rssi will usually range from -30 or so "up" to -100 or more. Numbers
    closer to zero are "stronger" or closer.
  - `attenuation` is a constant for the losses in the environment (humidity, air pressure(!),
    mammals etc). It's a "fudge factor". This is also set in the options. Experimentation with values
    ranging from 1.something to 3ish will usually get you somewhere.
  - `distance` is the resulting distance in metres.

- The values won't be suitable for all use-cases. Apart from the environmental factors we can't calculate
  (like walls, reflective surfaces etc), each device might transmit a different power level, and every transmitter
  and receiver might have antennae that perform differently. Because of this it is planned to allow separate
  calibration of scanners and devices to account for the variances.

### How do I choose values for Attenuation and Ref_Power?

- Soon you'll be able to set this per-device to account for variations in circuits, antennas and cases, but
  currently there are only the global defaults to fiddle with. Anyway, the idea is:
  - Place a transmitter 1 metre (just over 39 inches) from a scanner (bluetooth proxy)
  - In the attributes section of the transmitter's Distance sensor, watch the "Area rssi" value. Get a feel
    for what you consider to be an average. This will be your "Reference power at 1m" value.
  - Now move the transmitter away some distance (and measure that distance). Having a clear line-of-sight between
    the transmitter and the scanner is a good idea.
  - Now watch the "Distance" value. You want it to average around the right distance (but error towards a higher
    value, since a shorter measured distance is statistically less likely). That sentence is deliberately coy, since
    RF is a black art and I am not an ordained sorcerer. Also, some reflections might be in phase, most will not.
  - If the distance measured is always too high, decrease your attenuation figure. If it's too short, increase it.
    Repeat this procedure until you decide nothing works, the universe is pure chaos and it's time to give up.

## TODO / Ideas

- [x] Basic `bermuda.dump_devices` service that responds with measurements.
- [x] `area` if a device is within a max distance of a receiver
- [x] An interface to choose which devices should have sensors created for them
- [x] Sensors created for selected devices, showing their estimated location
- [ ] Switch to performing updates on receipt of advertisements, instead of periodic polling
- [ ] "Solve" realtime approximation of inter-proxy distances using Triangle Inequality
- [ ] Resolve x/y co-ordinates of all scanners and proxies (!)
- [ ] Some sort of map, just pick two proxies as an x-axis vector and go
- [ ] Config setting to define absolute locations of two proxies
- [ ] Support some way to "pin" more than two proxies/tags, and have it not break.
- [ ] An interface to define Areas in relation to the pinned proxies
- [x] Create entities (use `device_tracker`? or create own?) for each detected beacon
- [ ] Experiment with some of
      [these algo's](https://mdpi-res.com/d_attachment/applsci/applsci-10-02003/article_deploy/applsci-10-02003.pdf?version=1584265508)
      for improving accuracy (too much math for me!). Particularly weighting shorter
      distances higher and perhaps the cosine similarity fingerprinting, possibly against
      fixed beacons as well to smooth environmental rssi fluctuations.

## Hacking tips

Wanna improve this? Awesome! Bear in mind this is my first ever HA
integration, and I'm much more greybeard sysadmin than programmer, so ~~if~~where
I'm doing stupid things I really would welcome some improvements!

You can start by using the service `bermuda.dump_devices` to examine the
internal state.

## Prior Art

The `bluetooth_tracker` and `ble_tracker` integrations are only built to give a "home/not home"
determination, and don't do "Area" based location. (nb: "Zones" are places outside the
home, while "Areas" are rooms/areas inside the home). I wanted to be free to experiement with
this in ways that might not suit core, but hopefully at least some of this could find
a home in the core codebase one day.

The "monitor" script uses standalone Pi's to gather bluetooth data and then pumps it into
MQTT. It doesn't use the `bluetooth_proxy` capabilities which I feel are the future of
home bluetooth networking (well, it is for my home, anyway!).

ESPrescence looks cool, but I don't want to dedicate my nodes to non-esphome use, and again
it doesn't leverage the bluetooth proxy features now in HA. I am probably reinventing
a fair amount of ESPrescense's wheel.

**This component will set up the following platforms.**

| Platform         | Description                                                                                                                                                            |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `sensor`         | For any bluetooth devices you select in the integration's configuration, a device will be created with two sensors, tracking the "Area" and "Distance" from that area. |
| `device_tracker` | A device tracker entity will be created for each configured bluetooth address, which you can use to influence the "Home/Away" state of a "Person".                     |

## Installation

Definitely use the HACS interface! Once you have HACS installed, go to `Integrations`, click the
meatballs menu in the top right, and choose `Custom Repositories`. Paste `agittins/bermuda` into
the `Repository` field, and choose `Integration` for the `Category`. Click `Add`.

You should now be able to add the `Bermuda BLE Trilateration` integration. Once you have done that,
you need to restart Homeassistant, then in `Settings`, `Devices & Services` choose `Add Integration`
and search for `Bermuda BLE Trilateration`.

In the `Configuration` dialog, you can choose which bluetooth devices you would like the integration to track.

The instructions below are the generic notes from the template:

1. Using the tool of choice open the directory (folder) for your HA configuration (where you find `configuration.yaml`).
2. If you do not have a `custom_components` directory (folder) there, you need to create it.
3. In the `custom_components` directory (folder) create a new folder called `bermuda`.
4. Download _all_ the files from the `custom_components/bermuda/` directory (folder) in this repository.
5. Place the files you downloaded in the new directory (folder) you created.
6. Restart Home Assistant
7. In the HA UI go to "Configuration" -> "Integrations" click "+" and search for "Bermuda BLE Trilateration"

<!---->

## Contributions are welcome!

If you want to contribute to this please read the [Contribution guidelines](CONTRIBUTING.md)

## Credits

This project was generated from [@oncleben31](https://github.com/oncleben31)'s [Home Assistant Custom Component Cookiecutter](https://github.com/oncleben31/cookiecutter-homeassistant-custom-component) template.

Code template was mainly taken from [@Ludeeus](https://github.com/ludeeus)'s [integration_blueprint][integration_blueprint] template
[Cookiecutter User Guide](https://cookiecutter-homeassistant-custom-component.readthedocs.io/en/stable/quickstart.html)\*\*

---

[integration_blueprint]: https://github.com/custom-components/integration_blueprint
[black]: https://github.com/psf/black
[black-shield]: https://img.shields.io/badge/code%20style-black-000000.svg?style=for-the-badge
[buymecoffee]: https://www.buymeacoffee.com/AshleyGittins
[buymecoffeebadge]: https://img.shields.io/badge/buy%20me%20a%20coffee-donate-yellow.svg?style=for-the-badge
[commits-shield]: https://img.shields.io/github/commit-activity/y/agittins/bermuda.svg?style=for-the-badge
[commits]: https://github.com/agittins/bermuda/commits/main
[hacs]: https://hacs.xyz
[hacsbadge]: https://img.shields.io/badge/HACS-Custom-orange.svg?style=for-the-badge
[discord]: https://discord.gg/Qa5fW2R
[discord-shield]: https://img.shields.io/discord/330944238910963714.svg?style=for-the-badge
[exampleimg]: example.png
[forum-shield]: https://img.shields.io/badge/community-forum-brightgreen.svg?style=for-the-badge
[forum]: https://community.home-assistant.io/
[license-shield]: https://img.shields.io/github/license/agittins/bermuda.svg?style=for-the-badge
[maintenance-shield]: https://img.shields.io/badge/maintainer-%40agittins-blue.svg?style=for-the-badge
[pre-commit]: https://github.com/pre-commit/pre-commit
[pre-commit-shield]: https://img.shields.io/badge/pre--commit-enabled-brightgreen?style=for-the-badge
[releases-shield]: https://img.shields.io/github/release/agittins/bermuda.svg?style=for-the-badge
[releases]: https://github.com/agittins/bermuda/releases
[user_profile]: https://github.com/agittins
