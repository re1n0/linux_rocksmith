# Native JACK on SteamOS

(Based on [BWagener](https://github.com/BWagener)'s original version)

Last tested on a Steam Deck on 2025-02-22.

## Table of contents

1. [Install necessary stuff](#install-necessary-stuff)
1. [Create a clean prefix](#create-a-clean-prefix)
1. [wineasio](#wineasio)
1. [Installing RS_ASIO](#installing-rs_asio)
1. [Set up JACK](#set-up-jack)
1. [Starting the game](#starting-the-game)
1. [Troubleshooting](/guides/troubleshooting.md)

# Install necessary stuff

(I recommend `wine-staging` if your distro has it, but usual `wine` works as well.)

```
# disable readonly mode
sudo steamos-readonly disable

sudo pacman-key --init
sudo pacman-key --populate archlinux
sudo pacman-key --populate holo

# I also recommend to install: pipewire-jack-client
sudo pacman -S realtime-privileges wine-staging base-devel glibc linux-headers linux-api-headers libtool binutils lib32-glibc lib32-jack2 jack2 qjackctl
# the groups should already exist, but just in case
sudo groupadd audio
sudo groupadd realtime
sudo usermod -aG audio $USER
sudo usermod -aG realtime $USER
```

Please reboot.

<details><summary> How to check if this worked correctly</summary>

> For the packages, do `pacman -Q <package-name>` (You can do multiple packages at once). Should output the names and versions without errors.
>
> For the groups, run `groups`. This will give you a list, which should contain "audio" and "realtime".
</details>

# Create a clean prefix

Steam Play is now activated by default. The default one can be set by going to `Settings` ➞ `Compatibility`.
If you want to select a Proton version on a per-game basis, Right click Rocksmith and go to `Properties` ➞ `Compatibility`.

Delete or rename `$STEAMLIBRARY/steamapps/compatdata/221680`, then start Rocksmith and stop the game once it's running.

The rest will be set up later.

# wineasio

## Install

Installing `base-devel` is very useful for using the AUR and compiling in general. We installed these and additional needed packages in the first step.

<details><summary>Know already what's going on? Here are all commands in one piece without an explanation</summary>

> **If the commands in this collapsible section don't work for you, try the "longer" variant first before asking for help.**
>
> YOU NEED TO HAVE THE $PROTON AND $STEAMLIBRARY VARIABLE SET!! (or replaced with the correct path first)
>
> cd into the unpacked directory, then run this.
>
> ```
> rm -rf build32
> rm -rf build64
> make 32
> make 64
> sudo cp build32/wineasio32.dll /usr/lib32/wine/i386-windows/wineasio32.dll
> sudo cp build32/wineasio32.dll.so /usr/lib32/wine/i386-unix/wineasio32.dll.so
> sudo cp build64/wineasio64.dll /usr/lib/wine/x86_64-windows/wineasio64.dll
> sudo cp build64/wineasio64.dll.so /usr/lib/wine/x86_64-unix/wineasio64.dll.so
> cp build32/wineasio32.dll.so "$PROTON/lib/wine/i386-unix/wineasio.dll.so"
> cp build32/wineasio32.dll "$PROTON/lib/wine/i386-windows/wineasio.dll"
> cp build64/wineasio64.dll.so "$PROTON/lib64/wine/x86_64-unix/wineasio.dll.so"
> cp build64/wineasio64.dll "$PROTON/lib64/wine/x86_64-windows/wineasio.dll"
> env WINEPREFIX=$STEAMLIBRARY/steamapps/compatdata/221680/pfx ./wineasio-register
> ```
>
> And you're done, continue with [Installing RS_ASIO](#installing-rs_asio).
>
</details>

[Download](https://github.com/wineasio/wineasio/releases) the newest .tar.gz and unpack it. Open a terminal inside the newly created folder.



```
# build
rm -rf build32
rm -rf build64
make 32
make 64

# Install on normal wine
sudo cp build32/wineasio32.dll /usr/lib32/wine/i386-windows/wineasio32.dll
sudo cp build32/wineasio32.dll.so /usr/lib32/wine/i386-unix/wineasio32.dll.so
sudo cp build64/wineasio64.dll /usr/lib/wine/x86_64-windows/wineasio64.dll
sudo cp build64/wineasio64.dll.so /usr/lib/wine/x86_64-unix/wineasio64.dll.so
```



`wineasio` is now installed on your system.

<details><summary>How to check if it's installed correctly</summary>

> ```
> find /usr/lib32/ -name "wineasio*"
> find /usr/lib/ -name "wineasio*"
> ```
>
> This should output 4 paths (ignore the errors).
>
</details>

## Make use of

To make Proton use wineasio, we need to copy these files into the appropriate locations.

**STOP!** If you haven't set the environment variables yet, please follow [this part](/README.md#common-paths) of the prerequisites, then continue.

```
cp /usr/lib32/wine/i386-unix/wineasio32.dll.so "$PROTON/lib/wine/i386-unix/wineasio32.dll.so"
cp /usr/lib32/wine/i386-windows/wineasio32.dll "$PROTON/lib/wine/i386-windows/wineasio32.dll"
cp /usr/lib/wine/x86_64-unix/wineasio64.dll.so "$PROTON/lib64/wine/x86_64-unix/wineasio64.dll.so"
cp /usr/lib/wine/x86_64-windows/wineasio64.dll "$PROTON/lib64/wine/x86_64-windows/wineasio64.dll"
```

In theory, this should also work with Lutris runners (located in `$HOME/.local/share/lutris/runners/wine/`)

To register wineasio (so that it can be used in the prefix), run the `wineasio-register` script that comes in the wineasio zip and set the `WINEPREFIX` to Rocksmiths.

```
env WINEPREFIX=$STEAMLIBRARY/steamapps/compatdata/221680/pfx ./wineasio-register
```

Errors outputted by this command are expected. The important one is the message at the end saying "regsvr32: Successfully registered DLL ..." or "regsvr32: Failed to register ...".

<details><summary> How to check if this worked correctly</summary>

> Download this: [VBAsioTest_1013.zip](https://download.vb-audio.com/Download_MT128/VBAsioTest_1013.zip)
>
> Extract it somewhere and run a command like this (replace the last path with the correct path that you chose):
>
> ```
> WINEPREFIX=$STEAMLIBRARY/steamapps/compatdata/221680/pfx $PROTON/bin/wine /path/to/VBASIOTest32.exe
> ```
> !! The command above currently might not work. You can try instead: `LD_PRELOAD=/usr/lib32/libjack.so wine /path/to/VBASIOTest32.exe` !!
>
</details>

## Installing RS_ASIO

1. [Download](https://github.com/mdias/rs_asio/releases) the newest release, unpack everything to the root of your Rocksmith installation (`$STEAMLIBRARY/steamapps/common/Rocksmith2014/`)
1. Edit RS_ASIO.ini: fill in `wineasio-rsasio` where it says `Driver=`. Do this for every Output and Input section. You could also fill in `WineASIO` instead. Alternatively, you can [download mine](https://github.com/theNizo/linux_rocksmith/blob/main/RS_ASIO.ini).

And you're done with RS_ASIO. But in case you want to configure the inputs further (relevant for multiplayer), see [this](/guides/setup-rs-asio.md).

## Set up JACK

Open QjackCtl and go to "Setup". Select

1. Driver: alsa
1. Tick "Realtime"
1. Sample Rate: 48000
1. Frames/Period: 256 (my recommendation)
1. Periods/Buffer: 4 (my recommendation)
1. Select audio device as described below.

If you use the same device for input and output, you can set it in the "Parameters" 🡲 "Interface".

If you use one device for input and one for output (say, RealToneCable and internal soundcard for example), select them in the "Advanced" Tab, as "Output Device" and "Input Device". The number of channels below are usually selected automatically

Quick note on Frames and Buffer size: More frames equals better quality audio. More Buffer equals more stability. Higher numbers equal higher latency. 256/4 is decent audio with decent delay and works for most people.

Press apply, close setup and try to start JACK.

# Starting the game

![](/img/3-start-button.webp)

Delete the `Rocksmith.ini` inside your Rocksmith installation. It will auto-generate with the correct values. The only important part is the `LatencyBuffer=`, which has to match the Buffer Periods.

Steam and JACK need to be running.

If we start the game from the button that says "Play" in Steam, the game can't connect to wineasio (you won't have sound and will get an error message). This is an issue with Steam and pipewire-jack. There are two ways to go about this. You can apply both at the same time, they don't break each other.

[It has been reported](https://github.com/theNizo/linux_rocksmith/issues/63) that changing the volume crashes Rocksmith, due to a sound being played in KDE plasma. You can disable this by going to "System Settings" ➞ "Audio" ➞ "Configure Volume Controls" and untick "Play audio feedback for changes to".

<details><summary>1. LD_PRELOAD</summary>

* Advantages: Run from Steam directly
* Disadvantages: higher possibility of crashes, steps you might need to do every game-boot.

Add these launch options to Rocksmith:
```
LD_PRELOAD=/usr/lib32/libjack.so %command%
```

You can launch the game from Steam now. For the first few boot-ups, you have to remove window focus from Rocksmith (typically done with Alt+Tab) as soon as the window shows up. If it doesn't crash, continue with instructions.

If there is NO message saying "No output device found", RS_ASIO is working fine. If you can hear sound, everything works fine.

If you cannot hear sound, open QjackCtl and go to "Graph". We want to connect microphones to the inputs of Rocksmith and two outputs to our actual output device. Rocksmith will sometimes crash when messing with the patchbay. I recommend connecting everything before entering a profile.

---

</details>

<details><summary>2. Start script, shortcut in Steam</summary>

* Advantage: Reliable one time setup
* Disadvantages: Another Steam game entry, or having to launch from terminal entirely

### Get the start script

Please select the Proton Version you use (Rocksmith has been working fine since at least Proton 4 btw):

* [Proton 9 or higher](/guides/start-script/proton-9.md) (newer versions)
* [Proton 8 or lower](/guides/start-script/proton-8.md) (slightly easier)

We can start the game via this script now: `path/to/rocksmith-launcher.sh`



If you want the Steam overlay to work, you need to launch the script via Steam, see the next step.

### Making it nice via Steam entry (optional, but recommended)

With Proton's runtime, we can't start Rocksmith directly from the Steam Library just like that (excluding LD_PRELOAD). But we can use the Steam Library to start the script that starts the game in a way that Steam recognizes.

<details><summary>Fitting meme format</summary>

![](https://i.kym-cdn.com/photos/images/original/002/546/187/fb1.jpg)

</details>

Go into your Steam Library and select "Add a game" ➞ "Add a Non-Steam Game" on the bottom left.

Make sure you can see all files. Select the script we generated just now and add it. This will create a shortcut to the script, which will be refered to as "shortcut" from here on. For Proton versions 8 or lower, right click on the shortcut and select "Properties". Add these launch Options: `PIPEWIRE_LATENCY="256/48000" %command%`

You can now start the game from Steam. Use the shortcut, it will launch the actual game.

<details><summary>If launching the script from Steam doesn't work</summary>

You can try and add it to Lutris, then add a Lutris shortcut by right-clicking and selecting "Create Steam shortcut".

This works because of how Lutris behaves when games are launched from Steam. All the Steam shortcut does is to notify Lutris to start a game. This is finished when Lutris received the message (= Steam sees it as "stopped"). Lutris then launches the game.

Important Settings:

* Runner: Linux
* Working Directory: The folder where your script is.
* Disable Lutris Runtime: true

</details>

### Beautification (even more optional, but recommended)

Leaving the shortcut just like that is not pretty, so we're going to change that.

You can give the games in your Steam Library a custom look. A good Website for resources is the [SteamGridDB](https://www.steamgriddb.com/).

You can take artwork from [Rocksmith](https://www.steamgriddb.com/game/1841), [Rocksmith 2014](https://www.steamgriddb.com/game/2295), [Rocksmith+](https://www.steamgriddb.com/game/5359161) or anything else you want. I would recommend something that makes the shortcut look different than the game.

Right-Click on the shortcut and select "Properties".

**Name and icon:** Can be found in "General". I recommend something like "Rocksmith 2014 - Launcher".

**Everything else:** Can be found under "Customization"

---

</details>

# Troubleshooting

[Go to Troubleshooting](/guides/troubleshooting.md)
