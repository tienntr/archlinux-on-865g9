# Initial installation

Use `archinstall`. May need to fix Windows EOL in generated `/etc/fstab`.

This note assumes you installed KDE on Wayland and use `systemd-boot`.

# Password lockout

By default user will be lockout for 10 minutes after 3 failed login attempts
in a 15 minute period. Edit `/etc/security/faillock.conf`, increase `deny`
and decrease `unlock_time` if the default is too restrictive (it is, for me).

# Wayland-related issues

## Run apps in Wayland mode

### About `XDG_CONFIG_HOME`

`${XDG_CONFIG_HOME}` is direcdtory for user-specific configurations, defaulted
to `~/.config`.

### Set environment variables for Wayland session

Environment variable that should be set only for Wayland session can be set in
`${XDG_CONFIG_HOME}/environment.d/envvars.conf`

### QT apps

Set enviroment variable `QT_QPA_PLATFORM=wayland`. For Qt 6 apps, also install
`qt6-wayland`.

### Electron apps

Create `${XDG_CONFIG_HOME}/electron-flags.conf` with the following content:

```
--ozone-platform-hint=auto
```

Add `ELECTRON_OZONE_PLATFORM_HINT=auto` to `/etc/environment`.

## Wayland clipboard

Install `wl-clipboard` to use `wl-copy` and `wl-paste`.

# `systemd-boot` update

Enable `systemd-boot-update.service`. Note that you have to reboot twice to
actually *use* the new bootloader: `pacman` update the installer and during the
first reboot (using old bootloader) this service triggers installation of new
bootloader.

# Bash enhancement

Install `bash-completion`.

For "command-not-found" equivalent, install `pkgfile`, enable and start
`pkgfile-update.timer` to allow periodic database update, and add

```bash
if [[ -r /usr/share/doc/pkgfile/command-not-found.bash ]]; then
  . /usr/share/doc/pkgfile/command-not-found.bash
fi
```

to `/etc/bash.bashrc`.

# Network

## Firewall

Install `iptables-nft` (this removes `iptables`) to use nftables and prevent
conflicts with `iptables`. Install `firewalld`, enable and start
`firewalld.service`.

## DNS caching

Install `dnsmasq`, create `/etc/NetworkManager/conf.d/dns.conf` with the
following content:

```
[main]
dns=dnsmasq
```

Then run `nmcli general reload` as root. NetworkManager runs its own instance
of `dnsmasq` that listens on `127.0.0.1:53`.

## Wi-Fi

Install `wireless-regdb` and uncomment correct country in
`/etc/conf.d/wireless-regdom`.

It seems that the firmware on Qualcomm card self-manages regdomain, so the
regdomain can't be changed.

## Sharing Internet via Wi-Fi with NetworkManager

Need `dnsmasq` to work. NetworkManager run its own instance of `dnsmasq` as
DHCP server.

There are some issue with access point feature in NetworkManager:

* Can't create WPA3 access point.
* The network confuse some device and cause connection failure (Nexus 6 running of 
LineageOS can connect by Pixel 5a running stock ROM failed to connect).
* Need to figure out how to make KDE connect work (firewall-related issue?)

# Bluetooth

Enable and start `bluetooth.service`.

# Package management

## AUR

Install `git`.  Install `paru-bin` by cloning and `makepkg -si`.

Ideally AUR packages should be built in clean chroot, otherwise build and run issues
(albeit rare in my experience) may arise. To use this feature, you have to setup
paru` local repo first. Edit `/etc/pacman.conf`, uncomment the line

```
CacheDir = /var/cache/pacman/pkg/
```

and add the following excerpt to the end:

```
[options]
CacheDir = /var/lib/repo/aur

[aur]
SigLevel = PackageOptional DatabaseOptional
Server = file:///var/lib/repo/aur
```

The first `CacheDir` option specifies cache directory for official repos and the
second one specifies cache directory for the local `aur` repo.

Package can be built in clean chroot and installed with `-S --chroot` options.

### Parallel build

Install `pigz` and `pbzip2`.

Edit `/etc/makepkg.conf` to enable parallel `make` and compression:

```
MAKEFLAGS="-j8"
...
COMPRESSGZ=(pigz -p 8 -c -f -n)
COMPRESSBZ2=(pbzip2 -p8 -c -f)
COMPRESSXZ=(xz -c -z --threads=8 -)
COMPRESSZST=(zstd -c -z -q --threads=8 -)
```

### Rebuild AUR packages after dependencies update

If dependencies of an AUR package are updated, the packages may need to be rebuilt.
(Most?) AUR helpers don't do this automatically.

Installed `rebuild-detector`, and run `checkrebuild -v` to know which packages
should be rebuilt. `rebuild-dectector` also install pacman hook that run the check
automatically with smaller scan graph. It should be note that the tool may have
false negative.

To rebuild manually after a known dependenies update, search for the dependents and
rebuild them. For example, to rebuild all AUR packages that depend on Python after a
Python update, run:

```
pacman -Qoq /usr/lib/python${PREV_VERSION}/ | paru -S --rebuild --no-confirm -
```

or

```
pacman -Qoq /usr/lib/python${PREV_VERSION} | paru -S --answerclean All -
```

You can also rebuild all AUR packages periodically.

## Enhancements to `pacman`

Uncomment `Color` line in `/etc/pacman.conf` to enable color ouput.

Install `informant` and add your user to `informant` group so `pacman` will
prevent you from installing new packages without reading all the news.

Install `pacman-contrib`. It provides:

* `checkupdates` command: check for updates without the need for root
priviledge used to sync database.
* `paccache.timer`: enable and start this to discard unused cached packages
weekly.
* `paccache` command: remove cached packages manually.

Install `pacman-cleanup-hook` (AUR) to run `paccache` after each `pacman`
transaction.

Install `archlinux-contrib` to get `checkservices` command. It runs `pacdiff`
to merge `.pacnew` files then checks for processes running with outdated
libraries and prompts the user if they want them to be restarted.

Add `Server = https://archive.archlinux.org/packages/.all` to the end of
enabled mirrors. This allows using Arch Linux Archive to get old packages
and avoid 404 error when you install packages after a long time from the last
database synchronization.

## Contribute package statistics

Install `pkgstats`.

# Fonts

## Install MS fonts

Install `ttf-ms-win11-auto` AUR package.

## Fix jagged Calibri and Cambria fonts

Disable embedded bitmaps for these fonts by creating `/etc/fonts/local.conf`
with the following content:

```xml
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "urn:fontconfig:fonts.dtd">
<fontconfig>
  <!-- Disable embedded bitmap in some MS fonts which make text
  pixelated at some sizes -->
  <match target="font">
    <test name="family" compare="contains">
      <string>Calibri</string>
    </test>
    <edit name="embeddedbitmap" mode="assign">
      <bool>false</bool>
    </edit>
  </match>
  <match target="font">
    <test name="family" compare="contains">
      <string>Cambria</string>
    </test>
    <edit name="embeddedbitmap" mode="assign">
      <bool>false</bool>
    </edit>
  </match>
</fontconfig>
```

# Connecting with Android devices

Install `android-tools` and `android-udev`.

# Multimedia

# Image viewer

Install `gwenview`. Gwenview video playback also work if `phonon-qt5-vlc` is
installed.

## Hardware-accelerated video decoding

Install `mesa-vdpau`. Add `VDPAU_DRIVER=radeonsi` to `/etc/enviroment`.

### Firefox

Run Firefox in Wayland mode. Go to `about:config` and set
`media.ffmpeg.vaapi.enabled` to `true`.

**Note**: it seems that using VA-API causes time-out error in `amdgpu` driver
and session crashes. In that case switch to other virtual TTY and issue a
reboot commmand, or use REISUB magic SysRq.

# Power management

## Improve S0 power consumption

By default the laptop in sleep mode drain all the battery in less than 24h.
Edit boot entry on ESP to add `acpi.ec_no_wakeup=1` to kernel command line to
remedy this issue. The power consumption after applying this workaround is
<1% battery level per hour. Note that this disable waking up by opening the lid
or by pressing any key on the keyboard.

Disable waking from touchpad (by disabling correspondin I2C device) doesn't
improve power consumption.

# Vietnamese input method

Install `fcitx5-unikey` and `fcitx5-im`. Add to `/etc/enviroment` the following
lines:

```
XMODIFIERS=@im=fcitx
```

# Qt and GTK theming

## GTK warning when using Breeze theme

This warning may appear when launching GTK application:

```
Gtk-WARNING **: <time_stamp>: Theme parsing error: gtk.css:1649:16: '-gtk-icon-size' is not a valid property name
```

It's pretty benign but may cause distraction in CLI. The reason seems to be
changes in GTK that made the CSS property in Breeze theme for GTK no longer
valid.

Several ways to fix this:

* Set `GTK_THEME` enviroment variable to use other theme (e.g. `Adwaita:dark`).
* Go to System Settings > Appearance > Application Style > Configure GNOME/GTK
Application Style... and select other theme for GTK apps. To have Adwaita
theme in the drop-down list you may have to install `gnome-themes-extra`.

# Misc

## Enable REISUB magic SysRq

Create file `/etc/sysctl.d/99-enable-sysrq.conf` with the following content:

```
kernel.sysrq=224
```

PrtScr key and Fn + S can be used as SysRq key.

## Special keys

This is defined by keyboard/system firmware:

* Fn + R: Break
* Fn + S: SysRq
* Fn + C: ScrollLock
* Fn + W: Pause
* Fn + E: Insert

## Man pages

Install `man-db` and `man-pages`.
