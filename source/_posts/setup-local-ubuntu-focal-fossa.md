---
title: Setup Ubuntu Focal Fossa Desktop
top: false
cover: false
toc: true
mathjax: false
date: 2020-09-10 13:53:51
password:
summary:
tags:
  - Environments
categories: Linux
---



### mount bitlocker locked partition

* tool: [dislocker](https://github.com/Aorimn/dislocker)

* references: [ref1](https://www.linuxuprising.com/2019/04/how-to-mount-bitlocker-encrypted.html), [ref2](https://askubuntu.com/questions/617950/use-windows-bitlocker-encrypted-drive-on-ubuntu-14-04-lts)


1. Install `dislocker`

```bash
sudo apt install dislocker
```

2. Create two folders for decrypting and mounting partition

e.g.
```bash
sudo mkdir -p /media/bitlocker
sudo mkdir -p /media/bitmount
```

3. Identify partition

```bash
sudo fdisk -l
```

4. Decrypt

```bash
sudo dislocker -r -V /dev/sdaX -p1536987-000000-000000-000000-000000-000000-000000-000000 -- /media/bitlocker
```

* `-r` means read-only. This option avoids many potential problems.
* `/dev/sdaX` shall be replaced by your partition
* `1536987-000000-000000-000000-000000-000000-000000-000000` shall be replaced by your recovery key.

5. Mount

```bash
sudo mount -r -o loop /media/bitlocker/dislocker-file /media/bitmount
```

6. (Optional) Make a script

For convenience...

```bash
#!/bin/sh

sudo dislocker -r -V /dev/sdaX -p1536987-000000-000000-000000-000000-000000-000000-000000 -- /media/bitlocker

sudo mount -r -o loop /media/bitlocker/dislocker-file /media/bitmount

```

### enable three-finger swipe to switch workplace

* tool: [libinput-gestures](https://github.com/bulletmark/libinput-gestures)

* references: [ref1](https://unix.stackexchange.com/questions/24330/how-can-i-turn-off-middle-mouse-button-paste-functionality-in-all-programs?answertab=votes#tab-top)


1. Install `libinput-gestures`

Follow the instructions in `github`.

2. Configuration

Enable animation when switching workplace in *~/.config/libinput-gestures.conf*

```conf
# gesture swipe up	_internal ws_up
gesture swipe up	xdotool key super+Page_Down

# gesture swipe down	_internal ws_down
gesture swipe down	xdotool key super+Page_Up
```

note that `_internal ws_up` switches workplaces without animation. For more details please read the instructions in `github` or in the comments of the *.conf* file.

3. Disable unwanted middle-mouse-click-copy behavior

```bash
sudo apt install xbindkeys xsel xdotool

echo -e '"echo -n | xsel -n -i; pkill xbindkeys; xdotool click 2; xbindkeys"\nb:2 + Release' >> ~/.xbindkeysrc

# reload
xbindkeys -p

# start
xbindkeys

# stop
pkill xbindkeys

```

See [ref1](https://unix.stackexchange.com/questions/24330/how-can-i-turn-off-middle-mouse-button-paste-functionality-in-all-programs?answertab=votes#tab-top) for details.


### enable shortcut for deepwine applications

* tool: `xdottool`
* reference: [ref1](https://zhuanlan.zhihu.com/p/144286142)


1. Install `xdottool`

```bash
sudo apt install --no-install-recommends xdotool
```

2. Create script

Example in reference named as *open_wechat.sh*

```bash
#!/bin/sh

xdotool key --window $(xdotool search --limit 1 --all --pid $(pgrep WeChat.exe)) "ctrl+alt+W"
```

make the script executable by `chmod +x open_wechat.sh`

3. Set the shortcut `Settings` -> `Keyboard Shortcuts`


### onedrive

* tool: [OneDrive Client for Linux](https://github.com/abraunegg/onedrive)
* references: [installation](https://github.com/abraunegg/onedrive/blob/master/docs/INSTALL.md), [configuration](https://github.com/abraunegg/onedrive/blob/master/docs/USAGE.md)

It is recommended to run `onedrive` as a service and enable it at boot.
