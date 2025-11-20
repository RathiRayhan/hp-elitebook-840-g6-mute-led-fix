# HP Elitebook 840 G6 Mute LED Fix for Linux

This repository documents a permanent fix for the persistent **Microphone Mute LED (Orange Light)** issue on the **HP Elitebook 840 G6** running Linux (tested on Arch Linux, Kernel 6.x).

## 💻 System Information

* **Device:** HP Elitebook 840 G6
* **Audio Codec:** Realtek ALC215
* **Subsystem ID:** `0x103c854d`
* **OS:** Arch Linux (Kernel 6.x)
* **Driver:** `snd-hda-intel` (Legacy)

## ⚠️ The Problem

The Microphone Mute LED (F8 Key) remains illuminated (Orange) regardless of the actual mute state in ALSA/PulseAudio/PipeWire.

* **Symptom 1:** The light stays on after boot.
* **Symptom 2:** Even if manually turned off using `hda-verb`, the light turns back on immediately when audio playback starts (YouTube, Music, etc.) or when the system wakes from sleep. This is caused by aggressive **Audio Power Saving** features resetting the GPIO state.

## 🛠️ The Solution

The fix involves three parts:
1.  **Driver Enforcement:** Forcing the legacy `snd-hda-intel` driver instead of the modern SOF driver to ensure GPIO compatibility.
2.  **Power Management:** Disabling audio power-saving to prevent the codec from resetting the LED state during playback/idle transitions.
3.  **Automation:** A script and Systemd service to forcefully toggle the specific GPIO pin (`0x04`) to OFF on boot and resume.

### Prerequisites

You need `alsa-tools` installed to use `hda-verb`.

* **Arch Linux:** `sudo pacman -S alsa-tools`
* **Ubuntu/Debian:** `sudo apt install alsa-tools`

---

### Step 1: Force Legacy Driver (GRUB)

The modern SOF driver (`sof-audio-pci-intel-cnl`) often conflicts with GPIO LED mapping on this model. We need to enforce the legacy driver.

1.  Edit your GRUB configuration:
    ```bash
    sudo nano /etc/default/grub
    ```

2.  Add `snd_intel_dspcfg.dsp_driver=1` to the `GRUB_CMDLINE_LINUX_DEFAULT` line. It should look something like this:
    ```bash
    GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet snd_intel_dspcfg.dsp_driver=1"
    ```

3.  Update GRUB:
    ```bash
    # For Arch Linux
    sudo grub-mkconfig -o /boot/grub/grub.cfg
    
    # For Ubuntu/Debian
    sudo update-grub
    ```

### Step 2: Disable Audio Power Saving

This is crucial. Without this, the LED will turn back on whenever you play a video or song.

1.  Create a modprobe configuration file:
    ```bash
    sudo nano /etc/modprobe.d/audio_powersave.conf
    ```

2.  Add the following line:
    ```ini
    options snd_hda_intel power_save=0 power_save_controller=N
    ```

### Step 3: Create the Fix Script

This script uses `hda-verb` to manually set the **GPIO Pin 0x04** (associated with the LED) to **Low (Off)**.

1.  Create the script:
    ```bash
    sudo nano /usr/local/bin/hp-led-fix.sh
    ```

2.  Paste the following content:
    ```bash
    #!/bin/bash
    
    # Wait for sound card to initialize completely
    sleep 5

    # Target Node 0x01 (Audio Function Group)
    # SET_GPIO_MASK (0x716) -> 0x04 (Pin 2)
    hda-verb /dev/snd/hwC0D0 0x01 0x716 0x04

    # SET_GPIO_DIRECTION (0x717) -> 0x04 (Output)
    hda-verb /dev/snd/hwC0D0 0x01 0x717 0x04

    # SET_GPIO_DATA (0x715) -> 0x00 (Low/Off)
    hda-verb /dev/snd/hwC0D0 0x01 0x715 0x00
    ```

3.  Make the script executable:
    ```bash
    sudo chmod +x /usr/local/bin/hp-led-fix.sh
    ```

### Step 4: Automate with Systemd

We need this script to run on **Boot** and after **Wake from Sleep/Hibernate**.

1.  Create a systemd service file:
    ```bash
    sudo nano /etc/systemd/system/hp-led-fix.service
    ```

2.  Paste the following content:
    ```ini
    [Unit]
    Description=HP Elitebook Mute LED Fix
    After=sound.target suspend.target hibernate.target

    [Service]
    Type=oneshot
    ExecStart=/usr/local/bin/hp-led-fix.sh

    [Install]
    WantedBy=multi-user.target suspend.target hibernate.target
    ```

3.  Enable and start the service:
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable hp-led-fix.service
    sudo systemctl start hp-led-fix.service
    ```

---

## ✅ Verification

1.  **Reboot** your system.
2.  Wait ~5 seconds after the login screen appears. The Orange LED should turn off automatically.
3.  Play a video on YouTube or a local audio file. The light should **remain off**.
4.  **Suspend (Sleep)** the laptop and wake it up. The light should turn off again automatically after a few seconds.
