---
title: Making bluetooth headphones work with ANBERNIC RG-34XXSP
published: true
---

## Introduction

Initially I wanted to buy [Game Boy Advance SP](https://ru.wikipedia.org/wiki/Game_Boy_Advance_SP), but after studying some details and looking at its price, I decided to look for a new portable console that can emulate it, which would be as similar to it as possible. I chose [ANBERNIC RG-34XXSP](https://anbernic.com/products/rg34xxsp).

During use, I encountered the fact that it is impossible to use Bluetooth headphones on the standard firmware. When trying to connect, I was simply thrown into the Bluetooth settings menu!

I have researched this issue and found some workaround. But this workaround will work `ONLY for RetroArch` other apps will still output audio to the built-in speaker.

| ![Game Boy Advance SP](./assets/anbernic-rg-34xxsp/gba-sp.jpg "Game Boy Advance SP") | ![ANBERNIC RG-34XXSP](./assets/anbernic-rg-34xxsp/anbernic.jpg "ANBERNIC RG-34XXSP") |
| Game Boy Advance SP | ANBERNIC RG-34XXSP |

## Causes of bluetooth issue

By default, **ANBERNIC RG-34XXS** has **Ubuntu 22.04 (Jammy Jellyfish)** installed and the **SSH** daemon is running (**user** and **password**: `root`, standard port `22`).

Since I couldn't connect via the UI, I connected via **SSH**.

> **<font color="Chartreuse">[TIP]</font>**
> Disable **Lock Screen** in the settings, otherwise the ssh-connection will hang when the screen is locked.

And used the `bluetoothctl` utility from the `bluez` package to connect the headphones manually:

```
bluetoothctl# scan on
[CHG] Controller 68:8F:C9:6B:BF:9C Discovering: yes
[NEW] Device 18:26:54:B6:23:CE Galaxy Buds2 (23CE)
bluetoothctl# trust 18:26:54:B6:23:CE
bluetoothctl# pair 18:26:54:B6:23:CE
bluetoothctl# connect 18:26:54:B6:23:CE
```

The connection failed and I got an error:
```
Failed to connect: org.bluez.Error.Failed br-connection-profile-unavailable
```
In general, this is the main reason why the connection does not occur through the UI.

## Solution to the issue with bluetooth

I found out that the system does not use `pulsaudio`, and only `alsa` is available. I also found out that for `bluez` to work with `alsa` with Bluetooth, the [bluez-alsa](https://github.com/arkq/bluez-alsa) daemon is required to be running. But since it is not in the system, you will have to install it.

> **<font color="Fuchsia">[IMPORTANT]</font>**
> Before installing the package, set the time and date correctly and, just in case, the time zone. If you haven't done this yet, you can use the [timedatectl](https://www.freedesktop.org/software/systemd/man/latest/timedatectl.html) utility:
>
> Get a list of time zones:
> ```sh
> timedatectl list-timezones
> ```
>
> Set the desired time zone, for example `Europe/Berlin`:
> ```sh
> timedatectl set-timezone Europe/Berlin
> ```
>
> Synchronize date and time with the Internet:
> ```sh
> timedatectl set-ntp true
> ```
> Or set it manually via parameter `set-time`.
>
> You may also need to update the `ca-certificates` package. Otherwise, you will get an error from `apt` like this: `Certificate verification failed: The certificate is NOT trusted. The certificate chain uses not yet valid certificate.  Could not handshake: Error in the certificate verification`.

Now install `bluez-alsa-utils`:
```sh
apt update
apt install bluez-alsa-utils
```

Add the daemon to startup and run it:
```sh
systemctl enable bluez-alsa
systemctl start bluez-alsa
```

Now try the `bluetoothctl` utility again as described in the [Causes of bluetooth issue](#causes-of-bluetooth-issue) section. But this time the error should not appear.
This should also work in the UI now.

## Adding a new sound device

Now after a successful connection, you need to add a new device for alsa (unfortunately, the configuration for each new headphone will have to be entered manually, so far I have not found another way).

Display available Bluetooth audio devices:
```bash
bluealsa-aplay -L
```

Copy what you need.

> **<font color="Orange">[WARNING]</font>**
> Before making changes to configuration files, make backups of them!

Open `/etc/asound.conf` in a text editor (in my case it's `nano`):
```sh
nano /etc/asound.conf
```

And add to the end of the config:
```conf
pcm.buds2 {
    type plug
    slave.pcm "bluealsa:SRV=org.bluealsa,DEV=18:26:54:B6:23:CE,PROFILE=a2dp"
}

ctl.buds2 {
    type bluealsa
}
```
Where the value `slave.pcm` is the needed device obtained from `bluealsa-aplay -L` and `buds2` is a custom name for your device.


> **<font color="Fuchsia">[IMPORTANT]</font>**
> I also recommend using `alsamixer` to set the headphone volume to `100%`, since **RetroArch** will still only adjust it in software. The device will probably not show up in `alsamixer`. Therefore, press **F6** in its TUI and select **enter device name...**.

Now run the test. If everything is done correctly, then the phrases **Front Left** and **Front Right** should be heard in turn in the corresponding earphone:
```sh
speaker-test -t wav -D buds2 -p 10 -c 2
```

## RetroArch Fix

After all these manipulations, your headphones should now appear in the **RetroArch** settings. However, there will be no sound played in them (how unexpected).

| ![RetroArch settings](./assets/anbernic-rg-34xxsp/retroarch-settings.jpg "RetroArch settings") | ![RetroArch sound devices](./assets/anbernic-rg-34xxsp/retroarch-sound-devices.jpg "RetroArch sound devices") |

Using logging I found out that **RetroArch** requires `/usr/lib/arm-linux-gnueabihf/alsa-lib/libasound_module_pcm_bluealsa.so`.

This can be fixed by installing the package `libasound2-plugin-bluez:armhf`:
```sh
dpkg --add-architecture armhf
apt update
apt install libasound2-plugin-bluez:armhf
```
After that, restart **RetroArch** and select your headphones in the settings. And the sound should appear.

## Known Issues

- Sometimes when loading the device `bluez-alsa` the daemon terminates and the issue with Bluetooth appears again (the reason is not clear yet, it can be fixed by rebooting);
- In **RetroArch** `alsa` device is not saved on restart (seems to be a **ANBERNIC** or **RetroArch** issue);
- Headphones won't automatically connect to portable console (**ANBERNIC** issue).
