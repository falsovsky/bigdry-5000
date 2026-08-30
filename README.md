# Cecotec BigDry 5000: Local Control in Home Assistant

This repository contains the full procedure to control the Cecotec BigDry 5000 dehumidifier locally, through Home Assistant's LocalTuya integration, without the Tuya cloud account.

## Contents

- [`01-extracting-credentials.md`](01-extracting-credentials.md): how to get the `device_id`, `local_key`, and IP address from the Android app, with an Android emulator and Frida. No cloud account link, no phone root.
- [`02-home-assistant-setup.md`](02-home-assistant-setup.md): how to add the device to LocalTuya, the full DP (data point) map for this device, entity configuration, a workaround for a number-entity write bug, and how to build a dashboard with the app's own icons.
- [`dashboard.yaml`](dashboard.yaml): the Lovelace dashboard configuration used for this device.
- [`automations.yaml`](automations.yaml): the two automations for the target-humidity write workaround.
- [`input_number.yaml`](input_number.yaml): the helper entity used by the same workaround.

## Why This Exists

The standard way to get a device's `local_key` is the Tuya IoT Platform, with a QR code that links your developer account to your phone app. This QR code is known to fail for many users, with the error "QR code has expired." When that method fails, the only remaining option is to read the key directly from the app's own memory, with an Android emulator and Frida.

This repository documents the full process, start to finish, for one specific device. The method in Part 1 works for any Tuya/SmartLife-based app, not only this one.
