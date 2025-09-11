# JACK to ASIO with pipewire on NixOS

### Notice!

Flakes - while being widely adopted and useful - are still an experimental feature in Nix. If you don't want to use them in your NixOS configuration, either refer to the [legacy guide](legacy.md) or proceed on by using `default.nix` in the flake's [repo](https://github.com/re1n0/nixos-rocksmith). Using flake-compat is neither covered in this manual nor tested. You're on your own. 

## Table of contents

1. [NixOS Configuration](#configuration)
1. [Automated Script](#automated-script)
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
# flake.nix
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";

    # A flake providing necessary module `programs.steam.rocksmithPatch`
    nixos-rocksmith = {
      url = "github:re1n0/nixos-rocksmith";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = {self, nixpkgs, ...}@inputs: {
    nixosConfigurations.default = nixpkgs.lib.nixosSystem {
      specialArgs = {inherit inputs;};
      modules = [
        ./configuration.nix
      ];
    };
  };
}
```

```nix
# configuration.nix
{
  ### Audio
  services.pipewire.enable = true;

  # Add user to `audio` and `rtkit` groups.
  users.users.<username>.extraGroups = [ "audio" "rtkit" ];

  environment.systemPackages = with pkgs; [
    helvum # Lets you view pipewire graph and connect IOs
    rtaudio 
  ];

  ### Steam (https://nixos.wiki/wiki/Steam)
  programs.steam = {
    enable = true;
    rocksmithPatch.enable = true;
  };
}
```

## Automated Script

You must apply the configuration and rebuild your system _BEFORE_ continuing.

A script that does almost everything automatically is provided by the flake.

```sh
steam-run patch-rocksmith
```

First it will ask you if you want to override its default values, you can press enter to accept the default.

- Proton Version: the proton version you have installed. By default it assumes `Proton - Experimental`. You can select another version by writing the corresponding number and pressing enter.

- RS_ASIO: The RS_ASIO you want to install. Currently it downloads the version `0.7.4`.

Reboot your PC after the script finishes.


## Final Steps (for both manual and script)

### Add Launch Options to Rocksmith

Copy the following command and paste it as Launch Options (Steam > Rocksmith 2014 > Right Click > Properties... > General > Launch Options)

``` 
LD_PRELOAD=/usr/lib32/libjack.so PIPEWIRE_LATENCY=256/48000 %command%
```

Optionally, if you want to use [rs-linux-autoconnect](https://github.com/KczBen/rs-linux-autoconnect), you should provide it in Launch Options

```
LD_PRELOAD=/usr/lib32/librsshim.so:/usr/lib32/libjack.so PIPEWIRE_LATENCY=256/48000 %command%
```

### Test and Enjoy

Open `helvum` to see if Rocksmith is showing up.

# Known Issues

- Proton updates may require a re-patching (however system updates should work fine).

## Rocksmith freezes after starting and crashes

This means that the patch either wasn't properly applied OR was resetted, try to re-apply the patch.
