# Part 1: How to Get the Device ID, Local Key, and IP Address

## 1 Purpose

This procedure gets the `device_id`, `local_key`, and IP address for a Tuya-based device. Use these values to connect the device to Home Assistant through LocalTuya, without the Tuya cloud account link.

This procedure applies to the **Cecotec BigDry 5000** dehumidifier. The app package name is `es.cecotec.forceclima`. This app is a white-labeled Tuya/SmartLife app. The procedure applies to other Tuya-based apps too.

Do not root your phone. Use an Android emulator on your PC instead.

## 2 Why This Procedure Is Necessary

LocalTuya needs three items for each device: the `device_id`, the `local_key`, and the local IP address. The standard method links your Tuya IoT Platform account to your phone app with a QR code. This QR code frequently fails with the error "QR code has expired." If the QR code method fails, use this procedure instead.

## 3 Install an Android Emulator With Root Access

Use LDPlayer. Do not use the Android Studio emulator: its system images are difficult to root, and a rooted image still crashes under Frida native hooks.

1. Download and install LDPlayer.

## 4 Install the App From the Play Store

Use a throwaway Google account for the Play Store login inside LDPlayer. Do not use your real Google account.

1. Sign in to the Play Store inside LDPlayer with the throwaway account.
2. Install the Cecotec app from the Play Store.
3. Open the app and log in with your real account. Use the account that is linked to your physical device. This step enters your real account password inside the emulator. Change this password at the end of the process, in Part 2, section 8.
4. Confirm that the device is visible in the app and that it responds to commands from the app.

## 5 Connect adb to the Emulator

1. Install Android platform-tools (`adb`) on your PC.
2. In LDPlayer, open **Settings → Other**.
3. Find the **ADB debugging** option. Set the value to **Enable remote connection**.
4. Find the **Root permission** option. Turn this option ON.
5. Restart the emulator. A restart is necessary after a change to the ADB debugging value or the Root permission value.
6. Run this command: `adb connect <ldplayer-ip>:5555`.
7. Run this command: `adb devices`. The device status must show `device`.
8. Run this command: `adb shell su -c whoami`. The output must show `root`.
9. If the device status remains `offline` after step 7, restart the emulator again.

## 6 Install frida-tools on Your PC

1. Create a Python virtual environment: `python3 -m venv venv`
2. Install the packages: `./venv/bin/pip install frida frida-tools`
3. Run `./venv/bin/frida --version`. Record this version number. The `frida-server` binary in section 7 must match this version exactly.

## 7 Run frida-server on the Emulator

1. Get the `frida-server` binary for the x86_64 architecture. Use the version recorded in section 6, step 3.
2. Download the file `frida-server-<version>-android-x86_64.xz` from the frida GitHub releases page.
3. Decompress the file with this command: `xz -d frida-server-<version>-android-x86_64.xz`. This command produces a file named `frida-server-<version>-android-x86_64`.
4. Push the file to the device with this command, renamed to `frida-server` on the device: `adb push frida-server-<version>-android-x86_64 /data/local/tmp/frida-server`
5. Set the file permissions with this command: `adb shell chmod 755 /data/local/tmp/frida-server`
6. Start the server with this command: `adb shell "su -c '/data/local/tmp/frida-server &'"`
7. Run `frida-server --version` on the device. Confirm that the version matches the version from section 6, step 3.
8. Open the device app in the emulator. Run `./venv/bin/frida-ps -U` on your PC. This command lists every running process on the emulator, with its PID and name. Find the device app in this list. The process name is not always the app's package name. For the Cecotec app, the process name is `Cecotec`. Record the PID: this PID is needed in section 8. If the device app is not in the list, check the adb connection.

## 8 Trace the App to Find the Local Key

Do not use full TLS-interception scripts. Do not use proxy override, native SSL hooking, or certificate-pinning bypass scripts. These scripts cause a crash on LDPlayer's ARM-translation layer (Houdini) when Frida hooks native code.

Use Java-only tracing instead. This method reads the key directly from Java memory, after the app has already decrypted it.

1. Get the PID recorded in section 7, step 8.
2. Run this command: `frida-trace -U -j "*!*ocalKey*" -p <app pid>`. Replace `<app pid>` with the PID from step 1. This command finds every Java method with "LocalKey" in its name, in every loaded class.
3. Open the device detail screen in the app, or turn the device on and off in the app. This action forces the app to fetch or send the key.
4. Read the trace output. The output shows lines like this:
   ```
   DeviceRespBean.getLocalKey()
   <= "xxxxxxxxxxxxxxxx"
   ```
5. Record the 16-character string. This string is the `local_key`.
6. Read the surrounding trace lines for the `device_id`. The `device_id` often appears in the same call, like this:
   ```
   ddqdbbd$bdpdqbp.getLocalKey("smart/mb/in/<device_id>")
   ```

If step 2 produces no output, repeat step 2 with these patterns instead: `*!*Encrypt*`, `*!*device*ist*`, `*!*encodeString*`. The pattern `*!*encodeString*` also shows the `device_id` and other cached device data, through Tuya's local MMKV storage.

## 9 Get the IP Address

1. Find the device IP address in your router's DHCP lease list, or in the device's own Wi-Fi settings screen.

## 10 Stop the Trace Tools

1. Stop `frida-trace` on your PC.
2. Kill `frida-server` on the device with this command: `adb shell "su -c 'pkill -f frida-server'"`.

## 11 Confirm the Collected Values

This procedure must produce these three values. Confirm that all three values are recorded before you start Part 2.

- `device_id`: from section 8.
- `local_key`: from section 8.
- IP address: from section 9.

Part 2 uses this Android emulator again, in its section 6. Do not change your Cecotec account password yet. Part 2 has the correct point in the process to change the password.

Next: [Part 2: Home Assistant Setup](02-home-assistant-setup.md)
