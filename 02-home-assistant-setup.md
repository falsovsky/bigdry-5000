# Part 2: Home Assistant Setup

This procedure uses the `device_id`, `local_key`, and IP address from [Part 1](01-extracting-credentials.md). This procedure adds the device to Home Assistant, maps the data points (DPs), fixes a known write bug, and builds a dashboard with custom icons.

## 1 Add the Device to LocalTuya

1. Install the LocalTuya integration through HACS.
2. Open **Settings → Devices & Services → LocalTuya → Configure**.
3. Select **Add a new device**.
4. Select the manual entry option (shown as "...").
5. Enter the `device_id`, `local_key`, and IP address from Part 1.
6. Set the protocol version to `3.3`.
7. Submit the form.
8. If the connection fails with protocol version `3.3`, repeat steps 5 to 7 with protocol version `3.4`.
9. If the connection fails with protocol version `3.4`, repeat steps 5 to 7 with protocol version `3.1`.

If the values are correct, LocalTuya connects to the device and shows a list of raw DPs with their current values. The list looks like this:
```
1 (value: False)
2 (value: 2)
4 (value: 55)
...
```

## 2 Map the DPs

Do not guess the meaning of a DP from its position in the app menu. Use this test method instead:

1. Change one setting in the official app.
2. Read all the raw DP values again.
3. Find the DP that changed.
4. Record the DP number and the new value.
5. Repeat steps 1 to 4 for each app setting.

### 2.1 DP Map for the Cecotec BigDry 5000

This table applies to the Cecotec BigDry 5000 device. Use this table directly. Do not repeat the test method for this device.

| DP | Function | Values |
|----|----------|--------|
| 1 | Power | `False` / `True` |
| 2 | Mode | `0` = Normal, `1` = Continuous, `2` = Powerful, `3` = Dry clothes |
| 4 | Target humidity | integer, `30`–`80`, step `5` |
| 5 | Purify | `False` / `True` |
| 6 | Fan speed | `1` = Speed 1, `3` = Speed 3 (active only in Normal mode) |
| 7 | Lock | `False` / `True` |
| 102 | Sleep | `False` / `True` |
| 103 | Temperature | integer, °C |
| 104 | Humidity | integer, % |
| 11, 12, 101, 105 | Not used | value never changes |

The app's scheduled timers do not map to a DP. Tuya's cloud service controls the timers, not the device. Local control of the timers is not possible.

## 3 Set the Entity Type for Each DP

Open **Edit a device** in the LocalTuya configuration. Set the entity type for each DP. Do not leave a DP as a generic `sensor` if a better entity type exists.

- Use `switch` for a two-state on/off DP.
- Use `select` for an enum DP.
- Use `number` for an adjustable numeric DP.
- Use `sensor` for a read-only value. Set the correct `device_class` and `unit_of_measurement`.

### 3.1 The select_options Field Uses a Semicolon

The `select_options` and `select_options_friendly` fields use the semicolon character (`;`) as the separator. Do not use a comma. The two lists match by position.

Example, for DP 2 (Mode):
```
select_options: 0;1;2;3
select_options_friendly: Normal;Continuous;Powerful;Dry clothes
```

## 4 Fix the Number Entity Write Bug

A `number` entity in Home Assistant sends a float value. The example value is `65.0`. Many Tuya devices expect an integer value for an integer-type DP. The device receives the float value and rejects the command. The device does not show an error. The device sometimes plays a confirmation sound, but the value does not change.

LocalTuya provides a service named `localtuya.set_dp`. This service sends the value without a type change. Use this service through an automation.

1. Do not add the affected DP as a `number` entity. Leave the DP unmapped, or map it as a read-only `sensor` for reference.
2. Create an `input_number` helper. Set the `min`, `max`, and `step` values to match the DP. See the file `input_number.yaml` in this repository for the exact configuration used for DP 4.
3. Create an automation. This automation runs when the helper value changes. This automation calls `localtuya.set_dp` with the new value, converted to an integer with the Jinja filter `| int`.
4. Create a second automation. This automation runs when the real `number` entity value changes. This automation copies the new value to the helper, with the service `input_number.set_value`. This step keeps the helper synchronized with changes made from the official app.
5. Add the `input_number` helper to the dashboard. Do not add the raw `number` entity to the dashboard.

See the file `automations.yaml` in this repository for the exact automations used for DP 4.

A loop-prevention step is not necessary. Home Assistant sends a `state_changed` event only when a value changes. The second automation writes the same value back to the helper. This action does not trigger the first automation again.

## 5 Rename the Entities

LocalTuya adds the device name to the start of each entity's `friendly_name`. Example: "BigDry 5000 Power", "BigDry 5000 Mode". To remove this prefix, do not change the LocalTuya configuration. Change the display name in the entity registry instead.

1. Open **Settings → Devices & Services → Entities**.
2. Select an entity.
3. Set the **Name** field to the short name, without the device name prefix.
4. Repeat steps 2 and 3 for each entity.

## 6 Extract Icons From the App

Use this procedure to get the icon set for the dashboard in section 7, even if you have the exact BigDry 5000 device.

The app resource files inside the APK use obfuscated names. Do not extract icons from the APK for this reason. Use the device control panel bundle instead. Tuya downloads this bundle from a CDN URL at runtime. This bundle uses clear, readable file names.

### 6.1 Find and Download the Panel Bundle

1. Repeat Part 1, sections 6 to 8, to start `frida-server` on the emulator and `frida-trace` on your PC. Use the pattern `-j "*!*encodeString*"` in place of `-j "*!*ocalKey*"`.
2. Open the device control panel in the app. This action triggers the panel bundle download.
3. Read the trace output for a call with a `.tar.gz` URL in one of its arguments. The URL host is `images.tuyaeu.com` or a similar Tuya CDN host. The line looks like this:
   ```
   MMKV.encodeString(..., "panel_info_biz_prop_list", "[{...\"content\":\"https://images.tuyaeu.com/smart/ui/<id>-android_<version>.tar.gz\"...}]")
   ```
4. Copy the URL out of the trace line.
5. Download the URL with this command: `curl -O <url>`
6. Decompress the file with this command: `tar xzf <file>`
7. Go to the folder `drawable-xxhdpi` inside the decompressed files. This folder contains the highest-resolution icon set.

### 6.2 Recolor the Icons

The extracted icons are white shapes on a transparent background. Recolor each icon before use, because a white icon is not visible on a light-colored dashboard background. `dashboard.yaml` needs two colors for each icon used in the Mode, Wind Speed, and Power sections: a neutral color for the inactive state, and a blue color for the selected state. The Function section icons need only the neutral color.

This table maps each source file, from the `drawable-xxhdpi` folder, to the output file names that `dashboard.yaml` expects. The source file names are specific to the BigDry 5000 panel bundle.

| Source file | Neutral output | Active output |
|---|---|---|
| `src_res_power.png` | `power.png` | `power_active.png` |
| `src_res_mode_normal.png` | `mode_normal.png` | `mode_normal_active.png` |
| `src_res_mode_continuous.png` | `mode_continuous.png` | `mode_continuous_active.png` |
| `src_res_mode_powerful.png` | `mode_powerful.png` | `mode_powerful_active.png` |
| `src_res_mode_clothes.png` | `mode_clothes.png` | `mode_clothes_active.png` |
| `src_res_fan_low.png` | `fan_low.png` | `fan_low_active.png` |
| `src_res_fan_high.png` | `fan_high.png` | `fan_high_active.png` |
| `src_res_func_sleep.png` | `func_sleep.png` | not used |
| `src_res_func_clean.png` | `func_purify.png` | not used |
| `src_res_func_lock.png` | `func_lock.png` | not used |
| `src_res_nav_mode.png` | `nav_mode.png` | not used |
| `src_res_nav_fan.png` | `nav_fan.png` | not used |
| `src_res_nav_func.png` | `nav_func.png` | not used |

Run this script from inside the `drawable-xxhdpi` folder. This script reads the table above and creates every output file, in a new `out` folder.
```bash
#!/bin/bash
set -e
mkdir -p out

recolor() {
  magick "$1" \( +clone -fill "$2" -colorize 100% \) -compose SrcIn -composite "out/$3"
}

pairs="
src_res_power.png:power
src_res_mode_normal.png:mode_normal
src_res_mode_continuous.png:mode_continuous
src_res_mode_powerful.png:mode_powerful
src_res_mode_clothes.png:mode_clothes
src_res_fan_low.png:fan_low
src_res_fan_high.png:fan_high
"
for pair in $pairs; do
  src="${pair%%:*}"
  base="${pair##*:}"
  recolor "$src" "#5c6b73" "${base}.png"
  recolor "$src" "#2196f3" "${base}_active.png"
done

neutral_only="
src_res_func_sleep.png:func_sleep
src_res_func_clean.png:func_purify
src_res_func_lock.png:func_lock
src_res_nav_mode.png:nav_mode
src_res_nav_fan.png:nav_fan
src_res_nav_func.png:nav_func
"
for pair in $neutral_only; do
  src="${pair%%:*}"
  base="${pair##*:}"
  recolor "$src" "#5c6b73" "${base}.png"
done
```

### 6.3 Confirm the Output

1. Count the files in the `out` folder. The count must be 20: 7 icons with 2 colors each, plus 6 icons with 1 color each.
2. Open two or three files at random, to confirm each icon is a visible gray or blue shape, not a blank image.

## 7 Build the Dashboard

The file `dashboard.yaml` in this repository contains the full dashboard configuration used for the BigDry 5000, without the history graph. `mushroom-template-card` type supports a `picture` field and a `color` field. Both fields accept a Jinja template. Home Assistant evaluates the template on the backend, so the dashboard updates in real time when a DP value changes.

Do not use the `custom:button-card` card set for a picture that must change with the entity state. In this project, the JavaScript template feature (`[[[ ]]]`) of `button-card` did not re-render after the first page load. The Mushroom card did not have this problem.

To use the `dashboard.yaml` file:

1. Install the **Mushroom** card set through HACS. `dashboard.yaml` does not render correctly without this step.
2. Copy the files from the `out` folder from section 6 to the Home Assistant `www` folder, for example `config/www/bigdry/`.
3. Restart Home Assistant. Home Assistant registers the `/local/` static path only at startup, so a new `www` subfolder is not visible until a restart.
4. Open **Settings → Dashboards**.
5. Select **+ Add Dashboard**, then **New dashboard from scratch**.
6. Open the new dashboard.
7. Select the three-dot menu, top right, then **Edit Dashboard**.
8. Select the three-dot menu again, then **Raw configuration editor**.
9. Delete the existing content.
10. Paste the content of `dashboard.yaml`.
11. Select **Save**.

The dashboard has these sections:

1. **Power**: a `mushroom-template-card`, with the `power.png` / `power_active.png` icon pair.
2. **Status**: a standard `entities` card, with the temperature and humidity sensors. This section is shown at all times, even when the device is off.
3. **Dehumidification settings**: four `mushroom-template-card` cards in a grid, one for each mode. Each card calls the `select.select_option` service.
4. **Wind Speed** and **Target Humidity**: inside a `conditional` card. The condition is `select.bigdry_5000_mode` equal to `Normal`. This section is not shown in other modes, because the device does not accept a fan speed or humidity target outside Normal mode.
5. **Function**: three `mushroom-template-card` cards, for Sleep, Purify, and Lock.

Sections 3, 4, and 5 are each inside a `conditional` card. The condition is `switch.bigdry_5000_power` equal to `on`. These three sections are hidden when the device is off, because their controls have no effect while the device is off.

To reproduce this dashboard with a different device name, change every entity ID in `dashboard.yaml` to match your own entity IDs.

## 8 Change Your Account Password

You entered your real Cecotec account password inside an Android emulator, in Part 1, section 4. Section 6 of this document was the last step that used this emulator. Change the account password now.
