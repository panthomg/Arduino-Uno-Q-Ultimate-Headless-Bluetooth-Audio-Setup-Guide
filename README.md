<img width="2317" height="1075" alt="Untitled Document 2 (3)" src="https://github.com/user-attachments/assets/5b226afa-d724-43c9-bfa1-a6b756eb9e5a" />


# Arduino Uno Q — Ultimate Headless Bluetooth Audio Setup Guide

This comprehensive guide walks you through fixing the stubborn `br-connection-profile-unavailable` or `Protocol not available` errors when pairing Bluetooth audio devices (speakers, headphones, amplifiers) to the **Arduino Uno Q** running a headless Debian distribution over SSH.

---

## 🛠️ The Architecture: Why Does it Fail out of the Box?

Modern Debian distributions (Trixie/Sid/Stable) use **PipeWire** as the core audio daemon and **WirePlumber** as the session manager.

By default, WirePlumber enforces **Seat Monitoring**. It assumes audio devices—specifically Bluetooth audio routing endpoints—should only be accessible if a graphical display manager (like GDM, LightDM, or an active HDMI X11/Wayland desktop session) owns the hardware "seat".

Because an Arduino Uno Q is typically operated headlessly over an SSH terminal:

1. WirePlumber flags your session as non-graphical.
2. It automatically disables its internal BlueZ monitoring modules.
3. BlueZ handles the Bluetooth connection but throws `br-connection-profile-unavailable` because the audio server refuses to register the profile.

This guide completely bypasses the seat constraint, forces system persistence, and unlocks native command-line multimedia control.

---

## 🏃‍♂️ Step-by-Step Implementation Blueprint

### Step 1: System Upgrades & Bluetooth Backends

Before configuring settings, update the package indexes, push system upgrades to sync the Qualcomm MPU firmware libraries, and install the standalone Sound Open Firmware (SPA) Bluetooth plugins.

```bash
sudo apt update && sudo apt dist-upgrade -y
sudo apt install -y pipewire-audio libspa-0.2-bluetooth wireless-regdb mpv

```

> ⚠️ **Note on Upgrades:** If a blue dialog box (`needrestart`) appears asking you which services to restart, press `Tab` to highlight `<Ok>`, hit `Enter`, and gracefully reboot the MPU:
> ```bash
> sudo reboot
> 
> ```
> 
> 

---

### Step 2: Configure System-Wide BlueZ Policies

We must explicitly tell the Bluetooth daemon (`bluetoothd`) to load the Advanced Audio Distribution Profile (A2DP) sinks, sources, and media control protocols.

1. Open the primary Bluetooth configuration file:
```bash
sudo nano /etc/bluetooth/main.conf

```


2. Locate the `[General]` section header at the very top of the file. Directly underneath it, append the following line:
```text
Enable=Source,Sink,Media,Socket

```


3. Save and close (`Ctrl + O`, `Enter`, then `Ctrl + X`).

---

### Step 3: Strip Out WirePlumber Seat Restrictions

This is the critical configuration that fixes the headless issue. We will drop a global drop-in rule that instructs WirePlumber to monitor BlueZ endpoints regardless of whether a graphical user desktop session exists.

1. Create the system-wide WirePlumber overrides directory:
```bash
sudo mkdir -p /etc/wireplumber/wireplumber.conf.d

```


2. Build and edit the custom headless profile file:
```bash
sudo nano /etc/wireplumber/wireplumber.conf.d/50-bluez-no-seat.conf

```


3. Paste this exact structural block into the editor:
```text
wireplumber.profiles = {
  main = {
    monitor.bluez.seat-monitoring = disabled
  }
}

```


4. Save and close the file (`Ctrl + O`, `Enter`, then `Ctrl + X`).

---

### Step 4: Automate the Audio Environment Bus

Because your terminal is isolated via SSH, you must bind your command shell session to the system's runtime user D-Bus sockets where PipeWire lives.

Instead of typing this out every time you connect, append it to your user's shell profile so it **persists forever**:

```bash
echo 'export XDG_RUNTIME_DIR=/run/user/1000' >> ~/.bashrc
echo 'export DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus' >> ~/.bashrc
source ~/.bashrc

```

Now, cycle the daemons to ingest all the configuration overrides:

```bash
systemctl --user restart pipewire wireplumber
sudo systemctl restart bluetooth

```

---

## 🔗 Step 5: The Hardware Pairing & Verification Sequence

1. **Verify the Audio Engine Bridge:**
Run this diagnostic probe to confirm WirePlumber has loaded the BlueZ SPA driver libraries into memory:
```bash
pw-cli list-objects | grep -i blue

```


*If you see active output lines like `node.name = "bluez_midi.server"` or structural `bluez` references, your audio server is ready.*
2. **Execute Connection Matrix:**
Launch the interactive BlueZ shell:
```bash
bluetoothctl

```


Execute the following macro, substituting `XX:XX:XX:XX:XX:XX` with your target speaker's actual MAC address:
```text
power on
agent on
default-agent
untrust XX:XX:XX:XX:XX:XX
trust XX:XX:XX:XX:XX:XX
connect XX:XX:XX:XX:XX:XX

```


Your speaker will chime, and the terminal will output:
`[CHG] Device XX:XX:XX:XX:XX:XX Connected: yes` followed by `Connection successful`.
Type `exit` to return to your normal prompt.

---

## 🎵 Step 6: Headless Media Playback Playground

Since the audio framework is fully operational directly from the shell, you do not need a heavy, slow graphical web browser or Remote Desktop (VNC/RDP) running to stream content.

### 1. Play Local Audio Assets

Route standard digital master files or sound effects natively through the ALSA/PipeWire abstraction layers:

```bash
aplay /path/to/sound.wav

```

For compressed formats like MP3s, use the pre-installed lightweight engine:

```bash
mpv --no-video /path/to/track.mp3

```

### 2. Stream Live YouTube Audio via SSH Terminal

You can use `mpv` to capture raw video network streams, strip out the heavy video frames completely to save CPU cycles on the MPU, and pipe the sound directly to your Bluetooth speaker over the internet:

```bash
mpv --no-video "https://www.youtube.com/watch?v=dQw4w9WgXcQ"

```

### 3. Stream Web/Internet Radio Stations

Pass any raw Icecast or network audio stream URL directly to the board:

```bash
mpv --no-video "https://icecast.radio-stream-url.com/stream.mp3"

```

---

## 🎛️ Step 7: Headless Volume & Mixer Control

To adjust the gain levels of your Bluetooth device without a graphical UI, use the interactive terminal mixer interface:

```bash
alsamixer

```

* Press **`F6`** to select the sound card engine.
* Select **`PipeWire Sound Server`**.
* Use the **Up/Down Arrow Keys** to scale the systemic volume db curve safely.

---

## 🚨 Troubleshooting Checklist

* **Error: `Failed to connect: org.bluez.Error.Failed**`
* *Fix:* The speaker timed out or paired to another device (like your phone). Put the speaker back into discovery mode, run `bluetoothctl`, type `disconnect XX:XX:XX:XX:XX:XX`, then try `connect` again.


* **Error: `No such entity` or `Connection refused` when running `pw-cli**`
* *Fix:* Your session context lost the D-Bus pathways. Manually re-run:
```bash
export XDG_RUNTIME_DIR=/run/user/1000 && export DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus

```




* **Audio is Connected but Sound is Silenced/Muted**
* *Fix:* Make sure the user running the command belongs to the audio peripheral group:
```bash
sudo usermod -aG audio,bluetooth arduino

```





---

*Maintained by the Arduino Uno Q Embedded Audio Open Source Community. Contributions, issues, and documentation adjustments are welcome via pull requests!*
