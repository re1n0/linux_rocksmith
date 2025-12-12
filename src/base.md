000-title-000

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

000-install-necessary-000
# the groups should already exist, but just in case
sudo groupadd audio
sudo groupadd realtime
sudo usermod -aG audio $USER
sudo usermod -aG realtime $USER
```

000-install-necessary-post-000

<details><summary> How to check if this worked correctly</summary>

> For the packages, do `000-list-inst-000 <package-name>` (You can do multiple packages at once). Should output the names and versions without errors.
>
> For the groups, run `groups`. This will give you a list, which should contain "audio" and "realtime".
</details>

# Create a clean prefix

Steam Play is now activated by default. The default one can be set by going to `Settings` ➞ `Compatibility`.
If you want to select a Proton version on a per-game basis, Right click Rocksmith and go to `Properties` ➞ `Compatibility`.

Delete or rename `$STEAMLIBRARY/steamapps/compatdata/221680`, then start Rocksmith and stop the game once it's running.

The rest will be set up later.

# wineasio

000-wineasio-install-000

To make Proton use wineasio, we need to copy these files into the appropriate locations.

**STOP!** If you haven't set the environment variables yet, please follow [this part](/README.md#common-paths) of the prerequisites, then continue.

```
cp 000-lib32unix-000/wineasio32.dll.so "$PROTON/lib/wine/i386-unix/wineasio32.dll.so"
cp 000-lib32windows-000/wineasio32.dll "$PROTON/lib/wine/i386-windows/wineasio32.dll"
cp 000-lib64unix-000/wineasio64.dll.so "$PROTON/lib64/wine/x86_64-unix/wineasio64.dll.so"
cp 000-lib64windows-000/wineasio64.dll "$PROTON/lib64/wine/x86_64-windows/wineasio64.dll"
```

In theory, this should also work with Lutris runners (located in `$HOME/.local/share/lutris/runners/wine/`)

To register wineasio (so that it can be used in the prefix), run the `wineasio-register` script that comes in the wineasio zip and set the `WINEPREFIX` to Rocksmiths.

```
env WINEPREFIX=$STEAMLIBRARY/steamapps/compatdata/221680/pfx 000-wineasio-register-000
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
> !! The command above currently might not work. You can try instead: `LD_PRELOAD=000-libjack-path-000/libjack.so wine /path/to/VBASIOTest32.exe` !!
>
</details>

## Installing RS_ASIO

1. [Download](https://github.com/mdias/rs_asio/releases) the newest release, unpack everything to the root of your Rocksmith installation (`$STEAMLIBRARY/steamapps/common/Rocksmith2014/`)
1. Edit RS_ASIO.ini: fill in `wineasio-rsasio` where it says `Driver=`. Do this for every Output and Input section. You could also fill in `WineASIO` instead. Alternatively, you can [download mine](https://github.com/theNizo/linux_rocksmith/blob/main/RS_ASIO.ini).

And you're done with RS_ASIO. But in case you want to configure the inputs further (relevant for multiplayer), see [this](/guides/setup-rs-asio.md).

## Set up JACK

000-set-up-jack-000

# Starting the game

![](/img/3-start-button.webp)

Delete the `Rocksmith.ini` inside your Rocksmith installation. It will auto-generate with the correct values. The only important part is the `LatencyBuffer=`, which has to match the Buffer Periods.

000-steam-running-000

If we start the game from the button that says "Play" in Steam, the game can't connect to wineasio (you won't have sound and will get an error message). This is an issue with Steam and pipewire-jack. There are two ways to go about this. You can apply both at the same time, they don't break each other.

000-deck-note-000

<details><summary>1. LD_PRELOAD</summary>

* Advantages: Run from Steam directly
* Disadvantages: higher possibility of crashes, steps you might need to do every game-boot.

Add these launch options to Rocksmith:
```
000-ldpreload-command-000
```

You can launch the game from Steam now. For the first few boot-ups, you have to remove window focus from Rocksmith (typically done with Alt+Tab) as soon as the window shows up. If it doesn't crash, continue with instructions.

If there is NO message saying "No output device found", RS_ASIO is working fine. If you can hear sound, everything works fine.

000-connect-sound-000

---

</details>

<details><summary>2. Start script, shortcut in Steam</summary>

* Advantage: Reliable one time setup
* Disadvantages: Another Steam game entry, or having to launch from terminal entirely

### Get the start script

Please select the Proton Version you use (Rocksmith has been working fine since at least Proton 4 btw):

* [Proton 9 or higher](/guides/start-script/proton-9.md) (newer versions)
* [Proton 8 or lower](/guides/start-script/proton-8.md) (slightly easier)

We can start the game via this script now: `000-start-via-script-000`

000-start-script-pipewire-note-000

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
000-lutris-env-000
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
