# Introduction
All of the files related to messing about with a voice assistant, based on gerganov's marvellous [whisper.cpp](https://github.com/ggerganov/whisper.cpp/tree/master), running on an 8 gigabyte Raspberry Pi 5 equipped with a [Codec Zero](https://thepihut.com/products/iqaudio-codec-zero) audio board.

# Installation
Use the [Raspberry PI Imager](https://www.raspberrypi.com/news/raspberry-pi-imager-imaging-utility/) to write the headless Raspbian distribution for a Pi 5 to SD card (with SSH access enabled and your Wifi details pre-entered for ease of first use).  Insert the SD card into the Pi 5, plug the audio board onto the Pi 5 header, wire a small speaker to the speaker output terminals of the audio board (good enough for testing) and power up the Pi.

`ssh` (or [PuTTY](https://www.putty.org/) if you prefer) is used to access the Pi and `sftp` (or [FileZilla](https://filezilla-project.org/) if you prefer) is used to exchange files with the Pi, so make sure that both of those work to access the Pi from another computer on your LAN.

Log into the Pi over `ssh`.  Make sure the Pi is up to date with:

```
sudo apt update
sudo apt upgrade
```

To avoid future hassles with Python "externally managed environments", create and activate a virtual Python environment to work in:

```
python -m venv cerberus-env --system-site-packages
source cerberus-env/bin/activate
```

Note: `--system-site-packages` allows the `cerberus-env` virtual environment to see the system-wide Python packages; if you later want to install a package locally in the virtual environment, overriding a system one, use `pip install --ignore-installed blah`.

Set the Pi up to use the audio settings from the audio board by entering:

```
sudo nano /boot/firmware/config.txt
```

...and putting a `#` before the `dtparam` entry like so:

```
#dtparam=audio=on
```

Then `CTRL+X`, `Y` and `<enter>` to save the file and `sudo reboot` to restart the Pi with this new setting.

`ssh` into the Pi once more and run the following command:

```
grep -a . /proc/device-tree/hat/*
```

If the `vendor` entry says "IQaudIO Limited" (as opposed to "Raspberry Pi Ltd") then edit `/boot/firmware/config.txt` once more and add the following just after the `#`-ed out `dtparam=audio=on` line:

```
# Some magic to prevent the normal HAT overlay from being loaded
dtoverlay=
# CodecZero overlay for IQAudIO board
dtoverlay=rpi-codeczero
```

Save the file, `sudo reboot` and then `ssh` into the Pi once more.

In order to set up the correct mixer settings for the audio board you will need to install `git`:

```
sudo apt install git
```

Download the ALSA audio mixer scripts with:

```
git clone https://github.com/raspberrypi/Pi-Codec.git
```

The file in the newly  populated `~/Pi-Codec` directory that will set up the audio board to use its built-in microphone and send output to the speaker you have attached to the speaker terminals is `Codec_Zero_OnboardMIC_record_and_SPK_playback.state`.  Try it out first with:

```
sudo alsactl restore -f ~/Pi-Codec/Codec_Zero_OnboardMIC_record_and_SPK_playback.state
```

You might see something like:

```
alsa-lib main.c:1541:(snd_use_case_mgr_open) error: failed to import hw:0 use case configuration -2
alsa-lib main.c:1541:(snd_use_case_mgr_open) error: failed to import hw:1 use case configuration -2
No state is present for card vc4hdmi0
alsa-lib main.c:1541:(snd_use_case_mgr_open) error: failed to import hw:1 use case configuration -2
Found hardware: "vc4-hdmi" "" "" "" ""
Hardware is initialized using a generic method
No state is present for card vc4hdmi0
alsa-lib main.c:1541:(snd_use_case_mgr_open) error: failed to import hw:2 use case configuration -2
No state is present for card vc4hdmi1
alsa-lib main.c:1541:(snd_use_case_mgr_open) error: failed to import hw:2 use case configuration -2
Found hardware: "vc4-hdmi" "" "" "" ""
Hardware is initialized using a generic method
No state is present for card vc4hdmi1
```

All of these errors can be ignored: only "Remote I/O error" would represent a problem.

Check that recording is working by entering the following and speaking into `MIC2` on the board:

```
 arecord --channels=2 --vumeter=mono --format=S16_LE --duration=5 test.wav
 ```

Play the recorded file back over the speaker you have connected with:

```
aplay test.wav
```

To make the mixer settings stick on a reboot, create a unit file named `/etc/systemd/system/alsa_restore.service` with the following contents, editing it replace `<username>` with your user name:

```
# Set the ALSA mixer up correctly for the Codec Zero audio board

[Unit]
Description=Codec Zero configuration for ALSA
Requires=

[Service]
Type=oneshot
RemainAfterExit=no
# The - below is to ignore errors since alsactl restore generates spurious ones
ExecStart=-/usr/sbin/alsactl restore -f /home/<username>/Pi-Codec/Codec_Zero_OnboardMIC_record_and_SPK_playback.state

[Install]
WantedBy=sound.target
```

Then run:

```
sudo systemctl start alsa_restore
sudo systemctl enable alsa_restore
```

Finally, to make the sound board the default audio device, create a file named `.asoundrc` as follows:

```
sudo nano .asoundrc
```

...with contents:

```
pcm.!default {
        type hw
        card Zero
}
```

It is probably then a good idea to `sudo reboot` and re-run the recording/playback check.

To be continued...