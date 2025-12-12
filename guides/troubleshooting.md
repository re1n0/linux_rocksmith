# Troubleshooting

If some commands don't work, make sure you've set the environment variables as described [here](/README.md#common-paths)

## "I see the following window when starting the game"

![](https://user-images.githubusercontent.com/46491912/161832608-b333a1ae-80f7-4f0b-82d0-d7e87b9f4e9f.png)

Follow [this](/guides/troubleshoot-no-sound.md) to figure out what your problem is.

## Game crashes (on start)

Can happen sometimes when you use a different application, then focus Rocksmith again. Other than that:

### Try first
* If the game crashes at the start, try two more times. Sometimes it was just random.
* **Focus away:** If you use pipewire and the game crashes right after the window shows up, you could try taking the focus to another window as quick as possible. It helps sometimes, but isn't reliable.
* **Start from terminal:** This gives you more info on what's going on. Launch the script from the terminal or

### Audio devices misbehave
* Keep Pavucontrol (or whatever you used) open while starting/playing.
* Look at what happens in the patchbay. Does Rocksmith appear at all? Is something not connected? Do some entries jump around?
* (Pipewire) **Plug in less audio devices:** RS_ASIO can't handle audio devices changing in the patchbay and will crash. You can disable them of course, but it's easier to just not have them plugged in in the first place. My RealTone Cable didn't cause any issues yet btw.
* **Patch bay:** (Meaning: Changes with something like qpwgraph or Catia.) The game doesn't like these changes too much. You might get away with 1-2, but this is a bit luck-based.
* (Pipewire) Sometimes it can help to add `WINEASIO_NUMBER_INPUTS=2` to the launch command. This would limit the number of inputs to the given amount.

### Other
* **Disable Big Picture:** I think this was an issue for me at one point. I would do it just to be sure.
* **Try the old approach:** This is not meant to be used for playing anymore, but it's a reliable way to get the game running: `PIPEWIRE_LATENCY=256/48000 WINEPREFIX=$STEAMLIBRARY/steamapps/compatdata/221680/pfx $PROTON/bin/wine $STEAMLIBRARY/steamapps/common/Rocksmith2014/Rocksmith2014.exe`

## WineASIO

[VBASIO](https://forum.vb-audio.com/viewtopic.php?t=1204) is a handy debugging tool (that I've also [used in the past](https://github.com/theNizo/linux_rocksmith/issues/22#issuecomment-1276457128)). 

You can get verbose output of wineasio by using `/usr/bin/pw-jack -v -s 48000 -p 256 %command%`. -v stands for "verbose" and will give you additional information in the terminal.

## DLC

* In the past, we had to set the working directory to the root of the game's folder. This would be done in the script, in the properties of the shortcut, or in the terminal via `cd`.
* About those DLCs Ubisoft doesn't want you to know about: Add the .dll-installer in Steam ("Add non-Steam game"), set it to use Proton (right click on entry 🡲 game Properties 🡲 Compatibility), and then launch it from there.

## Other

* (Pipewire) If you use an audio interface, make sure the default sample rate is set to 48kHz. You can do this by running `pw-metadata -n settings 0 clock.rate 48000 && pw-metadata -n settings 0 clock.force-rate 48000`
* (Multiplayer) Calibration once didn't work for me. But entering 2nd players profile, setting up the sound there, then going back to first player and start Multiplayer worked apparently...

## Still doesn't help?

You can [create an issue](https://github.com/theNizo/linux_rocksmith/issues/new?assignees=&labels=help+wanted&projects=&template=help-needed.md&title=) and ask for help.
