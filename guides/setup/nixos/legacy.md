(This is the original version, with very minor tweaks.)

# JACK to ASIO with pipewire on NixOS

Thanks to [TimP4w](https://github.com/TimP4w) for writing this.

## Table of contents

1. [NixOS Configuration](#configuration)
1. [Method 1 - Manual](#method-1---manual)
    1. [Enable Steam Play for Windows Games](#1-enable-steam-play-for-windows-games)
    1. [Download Rocksmith 2014](#2-download-rocksmith-2014)
    1. [Install Wineasio](#3-install-wineasio)
    1. [Install RS_ASIO](#4-install-rs_asio)
    1. [Reboot PC](#5-reboot-your-pc)

1. [Method 2 - Script](#method-2---automated-script)
1. [Final Steps](#final-steps-for-both-manual-and-script)
1. [Known Issues](#known-issues)

# Tested and working with
```
NixOS 25.11.20250910.ab0f360 (Xantusia) [64-bit]
Wineasio: wineasio-1.2.0
Pipewire Jack: pipewire-1.4.7-jack
RS_ASIO 0.7.4
Proton: Proton - Experimental
```

# NixOS Configuration

## Assumptions
- You have steam (`programs.steam.enable = true;`) installed
  - This is very important, because we need steam that comes with its own `FHS` environment AND `steam-run` to be able to execute commands in this environment.
- You use the pipewire service from nixpkgs (`services.pipewire.enable = true;`)

After applying the configuration, reboot your PC.


## Minimal Configuration
```nix
  ### Audio
  sound.enable = true;

  services.pipewire = {
    enable = true;
    jack.enable = true; 
  };


  ### Audio Extra 
  security.rtkit.enable = true; # Enables rtkit (https://directory.fsf.org/wiki/RealtimeKit)
  
  #
  # domain = "@audio": This specifies that the limits apply to users in the @audio group.
  # item = "memlock": Controls the amount of memory that can be locked into RAM.
  # value (`unlimited`) allows members of the @audio group to lock as much memory as needed. This is crucial for audio processing to avoid swapping and ensure low latency.
  #
  # item = "rtprio": Controls the real-time priority that can be assigned to processes.
  # value (`99`) is the highest real-time priority level. This setting allows audio applications to run with real-time scheduling, reducing latency and ensuring smoother performance.
  #
  security.pam.loginLimits = [
    { domain = "@audio"; item = "memlock"; type = "-"; value = "unlimited"; }
    { domain = "@audio"; item = "rtprio"; type = "-"; value = "99"; }
  ];

  # Add user to `audio` and `rtkit` groups.
  users.users.<username>.extraGroups = [ "audio" "rtkit" ];

  environment.systemPackages = with pkgs; [
    qpwgraph # Lets you view pipewire graph and connect IOs
    pavucontrol # Lets you disable inputs/outputs, can help if game auto-connects to bad IOs
    unzip # Used by patch-nixos.sh
    rtaudio 
  ];

  ### Steam (https://nixos.wiki/wiki/Steam)
  programs.steam = {
    enable = true;
    package = pkgs.steam.override {
      extraLibraries = pkgs': with pkgs'; [ pkgsi686Linux.pipewire.jack ]; # Adds pipewire jack (32-bit)
      extraPkgs = pkgs': with pkgs'; [ wineasio ]; # Adds wineasio
    };
  };

```

### Explanation
We of course want mainly two things: audio and steam. 
These are pretty much self-explanatory, but there are some more settings that we need:

#### Audio
We use pipewire (`services.pipewire`) and pipewire-jack. The goal is to connect jack to wine via wineasio.

I noticed that for audio we need some extra things:
- rtkit: without this the game just crashes
- PAM loginLimits: we also need to set some limits here for the audio group to access real-time scheduling with a higher priority.
- user groups: we add our user to the `audio` and `rtkit` groups to enable these limits for us
- `qjackctl` to control our audio pipeline (there are also other alternatives here, such as `helvum` or `qpwgraph`)


#### Steam
Here we only want to add two things:
- Wineasio, to connect JACK with wine
- The 32-bit libraries of pipewire JACK since Rocksmith is a 32-bit game.

NixOS is not using the [FHS](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard) convention however the official steam program is packaged with its own FHS environment. So what we are doing is to add an extra library (`pkgsi686Linux.pipewire.jack`) to the default environment that comes from nixpkgs. For `wineasio` we only need some `.dll` and `.so` files that we usually need to compile. However if we install it via nixpkgs, this is done automatically and we can later copy the generated files (see below).





# Method 1 - Manual
You must apply the configuration and rebuild your system _BEFORE_ continuing.


## 1. Enable Steam Play for Windows Games

Open Steam and activate Proton (`Settings` 🡲 `Compatibility` 🡲 `Enable Steam play for all other titles`).

Restart steam.


## 2. Download Rocksmith 2014
Simply install it from steam and **launch it once**, until you need to choose your user. Then close it.

<details><summary> How to check if this worked correctly</summary>

> You can check if everything worked correctly if the directory `/home/$(whoami)/.steam/steam/steamapps/compatdata/221680/pfx` exists and has some files inside.
</details>



## BEFORE PROCEDING FURTHER ENTER THE STEAM FHS ENVIRONMENT WITH `steam-run bash`
All commands below assume that you are in this environment, they won't work otherwise.

## 3. Install Wineasio
We now need to copy the wineasio library to where it's needed.

First thing, let's set some environment variables to make the next commands shorter
```sh
PROTONVER="<your proton version>" # (e.g. "Proton - Experimental", "Proton 8.0". Needs to be the same as its directory name. You can check it under ~/.steam/steam/steamapps/common)

# Attention! Only execute one of the following
PROTONPATH="/home/$(whoami)/.steam/steam/steamapps/common/$PROTONVER/files" # For proton 9.0 and Experimental
PROTONPATH="/home/$(whoami)/.steam/steam/steamapps/common/$PROTONVER/dist" # For proton 8.0

WINEPREFIX="/home/.steam/steam/steamapps/compatdata/221680/pfx"
```

Then let's copy the unix files into Proton.


```sh
cp "/lib/wine/i386-unix/wineasio32.dll.so" "$PROTONPATH/lib/wine/i386-unix/wineasio32.dll.so"
cp "/lib/wine/x86_64-unix/wineasio64.dll.so" "$PROTONPATH/lib/wine/x86_64-unix/wineasio64.dll.so"
```

<details><summary> How to check if this worked correctly</summary>

> First check that `/lib/wine/i386-unix/wineasio32.dll.so` and `/lib/wine/x86_64-unix/wineasio64.dll.so` exist. The copy should give you an error if something went wrong.
</details>


<br>
<hr>
<br>

Now add the dlls to Rocksmith's prefix

```sh
cp "/lib/wine/i386-windows/wineasio32.dll" "$WINEPREFIX/drive_c/windows/syswow64/wineasio32.dll"
cp "/lib/wine/x86_64-windows/wineasio64.dll" "$WINEPREFIX/drive_c/windows/system32/wineasio64.dll"
```

Finally register the dlls
It's important that we use the wine binary that come with proton.

```sh
cd "$PROTONPATH/bin"
./wine regsvr32 /lib/wine/i386-windows/wineasio32.dll
./wine64 regsvr32 /lib/wine/x86_64-windows/wineasio64.dll
```

<details><summary> How to check if this worked correctly</summary>

> After ever `regsvr32` command you should get an alert from wine stating that the dll was registered correctly.
</details>


<details><summary> VBAsioTest</summary>

> You can further test if wineasio is really correctly installed, if this program runs and produces a sound that you can hear.
> Download this: [VBAsioTest_1013.zip](https://download.vb-audio.com/Download_MT128/VBAsioTest_1013.zip)
>
> Extract it somewhere and run a command like this (replace the last path with the correct path that you chose):
> ```
> WINEPREFIX="/home/.steam/steam/steamapps/compatdata/221680/pfx" $PROTONPATH/bin/wine /path/to/VBASIOTest32.exe
> ```
>
</details>

## 4. Install RS_ASIO
Download the latest release from its Github repository https://github.com/mdias/rs_asio/releases, unzip the folder and copy the content in the Rocksmith installation (should be in `~/.steam/steam/steamapps/common/Rocksmith2014/`).

Now open the `RS_ASIO.ini` file that you just copied and set `Driver=wineasio-rsasio` for all your inputs and outputs.

## 5. Reboot your PC
Simply reboot your PC.


## Method 2 - Automated Script
You must apply the configuration and rebuild your system _BEFORE_ continuing.

A script that does almost everything automatically is provided in this repo.
Simply download `scripts/patch-nixos.sh` and execute it.

```sh
chmod +x patch-nixos.sh
steam-run ./patch-nixos.sh
```
First it will ask you if you want to override its default values, you can press enter to accept the default.

- Proton Version: the proton version you have installed. By default it assumes `Proton 10.0`. You can select another version by writing the corresponding number and pressing enter.

- RS_ASIO: The RS_ASIO you want to install. Currently it downloads the version `0.7.4`.

Reboot your PC after the script finishes.


## Final Steps (for both manual and script)
## Add Launch Options to Rocksmith

Copy the following command and paste it as Launch Options (Steam > Rocksmith 2014 > Right Click > Properties... > General > Launch Options)
``` 
LD_PRELOAD=/usr/lib32/libjack.so PIPEWIRE_LATENCY=256/48000 %command%
```

### Test and Enjoy

Open `qjackctl` to see if Rocksmith is showing up.

You may need to disable some devices if Rocksmith connects to the wrong ones.



# Known Issues
- Proton updates may require a re-patching (however system updates should work fine).
- We (mostly) can't change the inputs of Rocksmith when it's running, otherwise it will crash. Therefore disabling the devices we don't want to automatically connect is often required.

## Rocksmith freezes after starting and crashes
This means that the patch either wasn't properly applied OR was resetted, try to re-apply the patch.
