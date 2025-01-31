# Introduction
All of the files related to messing about with a voice assistant, based on gerganov's marvellous [whisper.cpp](https://github.com/ggerganov/whisper.cpp/tree/master), running on an 8 gigabyte Raspberry Pi 5 equipped with a [Codec Zero](https://thepihut.com/products/iqaudio-codec-zero) audio board.

# Installation
## Pi Basics
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

## Pi Audio
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

To make the mixer settings stick on a reboot, create a unit file named `/etc/systemd/system/alsa_restore.service` with the following contents, editing it to replace `<username>` with your user name:

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

Finally, to make the sound board the default audio device:

```
nano /etc/asound.conf
```

...and add contents:

```
pcm.!default {
    type hw
    card Zero
}
ctl.!default {
    type hw
    card Zero
}
```

It is probably then a good idea to `sudo reboot` and re-run the recording/playback check.

## `whisper.cpp`
Install some pre-requisites (`libsdl2-dev` to capture microphone input):

```
sudo apt install cmake libsdl2-dev
```

You will also need `ffmpeg` to convert `.wav` files into 16-bit format (which is what `whisper.cpp` swallows); this you will need to build enough of with:

```
git clone git://source.ffmpeg.org/ffmpeg --depth=1
cd ffmpeg
./configure --extra-ldflags="-latomic" --arch=arm64 --target-os=linux
make -j4
sudo make install
```

With that done, clone the [whisper.cpp](https://github.com/ggerganov/whisper.cpp) repo to the Pi:

```
git clone https://github.com/ggerganov/whisper.cpp.git
cd whisper.cpp
```

Next, download a speech model; question is, [which one](https://github.com/openai/whisper#available-models-and-languages)?  The options seemed to be size, quantization (integers to improve efficiency), language (I only needed `.en`) and whether diarization (recognising different speakers) is supported (few seemed to include this).  Given the RAM and speed differences, I stuck with just the `base` for now: I downloaded the most recently updated quantized `.en` one in [the list](https://huggingface.co/ggerganov/whisper.cpp/tree/main), which was `ggml-base.en-q8_0.bin`.

```
curl -L --output models/ggml-base.en-q8_0.bin https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.en-q8_0.bin
```

Build the code with:

```
cmake -B build
cmake --build build --config Release
```

Convert the sample you recorded earlier into 16-bit format with:

```
ffmpeg -i ~/test.wav -ar 16000 -ac 1 -c:a pcm_s16le ~/test_16.wav
```

...then apply the downloaded speech model to it with:

```
./build/bin/whisper-cli -m models/ggml-base.en-q8_0.bin -f ~/test_16.wav
```

You should see on the screen whatever you said in that sample, printed out.  Make it more interesting by asking `whisper.cpp` to print the text in traffic-light confidence colours:

```
./build/bin/whisper-cli -m models/ggml-base.en-q8_0.bin -f ~/test_16.wav --print-colors
```

Now build the [real-time command example](https://github.com/ggerganov/whisper.cpp/tree/master/examples/command) with:

```
cmake -B build -DWHISPER_SDL2=ON
cmake --build build --config Release
```

Run it with:

```
./build/bin/whisper-command -m ./models/ggml-base.en-q8_0.bin -ac 768 -t 5
```

...and say the training phrase that is required at the start.  Then say whatever you like and it should appear as text on the screen.

To put an LLM behind this, the above command will already have built the [talk-llama](https://github.com/ggerganov/whisper.cpp/tree/master/examples/talk-llama) example, but you need to [download a GGUF model file](https://github.com/ggerganov/llama.cpp?tab=readme-ov-file#obtaining-and-quantizing-models) to run it with.  There are a gazillion of these, all trained on specific datasets, that I don't pretend to understand.  All I know is that I needed something pretty small to fit into the RAM left on an 8 gigabyte Pi with the speech recognition in there as well; I found [some guidance on that here](https://huggingface.co/bartowski/sum-small-GGUF#which-file-should-i-choose).  I randomly downloaded Bartowski's [Llama-3.2-1B-Instruct-f16.gguf](https://huggingface.co/bartowski/Llama-3.2-1B-Instruct-GGUF/resolve/main/Llama-3.2-1B-Instruct-f16.gguf), built on Meta's Llama-3.2-1B [trained for dialogue](https://huggingface.co/meta-llama/Llama-3.2-1B-Instruct#training-data), as it fitted into 2.5 gigabytes of RAM:

```
curl -L --output models/Llama-3.2-1B-Instruct-f16.gguf https://huggingface.co/bartowski/Llama-3.2-1B-Instruct-GGUF/resolve/main/Llama-3.2-1B-Instruct-f16.gguf
```

Run it with:

```
./build/bin/whisper-talk-llama --session ./session_file -mw ./models/ggml-base.en-q8_0.bin -ac 768 -ml ./models/Llama-3.2-1B-Instruct-f16.gguf -p "Rob" -t 8
```

...and you can have a conversation with a `LLaMa`, where it will reply in text form.  To have it reply with a voice, activate the Python virtual environment and install `piper-tts`:

```
source ~/cerberus-env/bin/activate
pip install piper-tts
```

...download a voice:

```
curl -L --output ~/en_GB-alan-medium.onnx https://huggingface.co/rhasspy/piper-voices/resolve/v1.0.0/en/en_GB/alan/medium/en_GB-alan-medium.onnx
curl -L --output ~/en_GB-alan-medium.onnx.json https://huggingface.co/rhasspy/piper-voices/resolve/v1.0.0/en/en_GB/alan/medium/en_GB-alan-medium.onnx.json
```

...and edit the file `examples/talk-lama/speak` to point it at the voice, so:

```
cat $2 | piper --model ~/en_US-lessac-medium.onnx --output-raw | aplay -q -r 22050 -f S16_LE -t raw -
```

...becomes:

```
cat $2 | piper --model ~/en_GB-alan-medium.onnx --output-raw | aplay -D plughw:CARD=Zero,DEV=0 -q -r 22050 -f S16_LE -t raw -
```

Note: the `-D` part is required because the output from `piper` is a mono stream and when `aplay` is configured to use sound HW directly it can do no software conversions, no matter how trivial.  Using `plughw` instead provides conversion of the mono stream to the output of the sound board which is stereo; Deep Seek helped me get the syntax of the `aplay` command correct :-).

Run `whisper-talk-llama` again and now you can speak to the LLaMa on the Pi and it will respond in kind.

## Other LLMs
Since running with Bartowski's [Llama-3.2-1B-Instruct-f16.gguf](https://huggingface.co/bartowski/Llama-3.2-1B-Instruct-GGUF/resolve/main/Llama-3.2-1B-Instruct-f16.gguf) took just 2.7 gigabytes of RAM in total, I tried updating to [Meta-Llama-3.1-8B-Instruct-Q6_K_L.gguf](https://huggingface.co/bartowski/Meta-Llama-3.1-8B-Instruct-GGUF/resolve/main/Meta-Llama-3.1-8B-Instruct-Q6_K_L.gguf), described as "very high quality, near perfect, recommended".  This ran in 6.9 gigabytes of RAM and the answers were improved (for instance, it knew about timezones when the previous one did not, though it still thought I could drive from London to New York) but it was a _lot_ slower.

I wanted to try the ARM optimised version of the same thing, [Meta-Llama-3.1-8B-Instruct-Q4_0_4_4.gguf](https://huggingface.co/bartowski/Meta-Llama-3.1-8B-Instruct-GGUF/resolve/main/Meta-Llama-3.1-8B-Instruct-Q4_0_4_4.gguf), but that failed to load with the error "`gguf_init_from_file_impl: tensor 'blk.0.attn_k.weight' of type 31 (TYPE_Q4_0_4_4 REMOVED, use Q4_0 with runtime repacking) has 4096 elements per row, not a multiple of block size (0)`", a problem being discussed [here](https://github.com/ggerganov/llama.cpp/issues/10757) I think.

I also wanted to try Bartowski's ARM-optimized version of [DeepSeek-R1-Distill-Qwen-7B-IQ4_NL.gguf](https://huggingface.co/bartowski/DeepSeek-R1-Distill-Qwen-7B-GGUF/resolve/main/DeepSeek-R1-Distill-Qwen-7B-IQ4_NL.gguf) but that suffers from a different loading error "`llama_model_load: error loading model: error loading model vocabulary: unknown pre-tokenizer type: 'deepseek-r1-qwen'`", being discussed [here](https://github.com/oobabooga/text-generation-webui/issues/6679).

All quite bleeding-edge then; I'll leave it a little while to settle.

## Other Speech Models
Given that I couldn't get far with other LLMs, I thought I'd try some different speech models to see what difference they might make to transcription accuracy.  I downloaded the top-end one, [ggml-large-v3-turbo-q8_0.bin](https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-large-v3-turbo-q8_0.bin), but it really didn't seem very turbo, taking more than 20 seconds to process any command.  I went back to `base` and tried the non-quantized version [ggml-base.bin](https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.bin), which was very slightly slower (2ish rather than 1.7ish seconds) and not significantly better.  All of which made me try [ggml-tiny.en-q8_0.bin](https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-tiny.en-q8_0.bin), and that seemed pretty good actually, with a response time approaching a second.  It is possible, of course, that the quality of the microphone, my position relative to it, etc. introduces more variation in quality than the speech model; the cost in latency with the larger models is what I'm tending to notice most.

