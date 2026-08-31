# Initial installation

Use `archinstall`. May need to fix Windows EOL in generated `/etc/fstab`.

This note assumes you installed KDE on Wayland and use `systemd-boot`.

# Password lockout

By default user will be lockout for 10 minutes after 3 failed login attempts
in a 15 minute period. Edit `/etc/security/faillock.conf`, increase `deny`
and decrease `unlock_time` if the default is too restrictive (it is, for me).

# Fingerprint reader

Install `fprintd`; the Synaptics `06cb:00f0` reader is supported by the
repository version of `libfprint`.

Enroll fingerprints through Plasma System Settings or with:

```
fprintd-enroll
fprintd-verify
```

Plasma 6's lock screen supports fingerprint unlocking through the
package-provided `/usr/lib/pam.d/kde-fingerprint`; do not create or modify
`/etc/pam.d/kde` for ordinary lock-screen use.

SDDM login remains password-based. This also allows `pam_kwallet5` to unlock
KDE Wallet, which cannot be unlocked using only a fingerprint.

Do not add `pam_fprintd.so` to `system-auth` merely for Plasma lock-screen
support, because that enables fingerprint authentication for additional PAM
consumers such as `sudo`.

# Wayland-related issues

## Run apps in Wayland mode

### About `XDG_CONFIG_HOME`

`${XDG_CONFIG_HOME}` is directory for user-specific configurations, defaulted
to `~/.config`.

### QT apps

Qt 6 applications use Wayland automatically. No environment variable is
normally needed.

Qt 5 applications run through Xwayland by default. To allow them to run
natively on Wayland, install `qt5-wayland`. After installing it, Qt 5
applications should normally select the appropriate backend automatically.
Do not set `QT_QPA_PLATFORM` globally.

To test a particular application on native Wayland, run:

```
QT_QPA_PLATFORM=wayland application
```

If the application has Wayland compatibility problems, force it to use
Xwayland:

```
QT_QPA_PLATFORM=xcb application
```

## Wayland clipboard

Install `wl-clipboard` to use `wl-copy` and `wl-paste`.

# `systemd-boot` update

Enable `systemd-boot-update.service`. Note that you have to reboot twice to
actually *use* the new bootloader: the systemd package updates the bootloader
binary under `/usr/lib/systemd/boot/efi`; the service copies it to the ESP on
the next boot.

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

Install `firewalld`, enable and start `firewalld.service`.

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

On this laptop, the Qualcomm QCNFA765/WCN6855 is reported by ath11k as a
self-managed regulatory device. Although Linux sets the global domain to the
country specified in `wireless-regdom`, the firmware may rejects the request
and retains US rules. See [WCN6855_notes.md](WCN6855_notes.md) for more information.

## Sharing Internet via Wi-Fi with NetworkManager

Need `dnsmasq` to work. NetworkManager run its own instance of `dnsmasq` as
DHCP server.

There are some issue with access point feature in NetworkManager:

* Can't create WPA3 access point.
* The network confuse some device and cause connection failure (Nexus 6 running of 
LineageOS can connect by Pixel 5a running stock ROM failed to connect).

# Bluetooth

Enable and start `bluetooth.service`.

# Package management

## AUR

Install `git`.  Install `paru-bin` by cloning and `makepkg -si`.

Ideally, AUR packages should be built in a clean chroot. Install `devtools`,
then configure paru to build packages into a local repository.

Add the local repository to `/etc/pacman.conf`:

```
[aur]
SigLevel = PackageOptional DatabaseOptional
Server = file:///var/lib/repo/aur
```

Enable the local repository and clean-chroot build in `/etc/paru.conf`:

```
[options]
LocalRepo = aur
Chroot
```

Modern paru can technically build in a chroot without a local repository,
but the local repository remains the better-supported and more robust workflow
for interdependent AUR packages.

After this configuration, running plain `paru` builds AUR packages in the clean
chroot, adds them to the local `[aur]` repository, and installs them through
`pacman`.

### Parallel build

Install `pigz` and `pbzip2`.

Edit `/etc/makepkg.conf` to enable parallel `make` and compression:

```
MAKEFLAGS="-j8"
...
COMPRESSGZ=(pigz -p 8 -c -f -n)
COMPRESSBZ2=(pbzip2 -p8 -c -f)
COMPRESSXZ=(xz -c -z --threads=8 -)
COMPRESSZST=(zstd -c -q -T0 -)
```

### Rebuild AUR packages after dependency updates

AUR packages are not automatically rebuilt when an ABI dependency changes.
Install `rebuild-detector`; its pacman hook reports many packages that may need
rebuilding. Run a more complete check manually with:

```
checkrebuild -v
```

After a Python minor-version transition, find packages that still own files in
the previous interpreter's directory and force paru to rebuild them:

```
pacman -Qoq /usr/lib/python${PREV_VERSION}/ | paru -S --rebuild=yes -
```

Review the package list before using `--noconfirm` in `paru` command.
This directory-based check does not detect every possible ABI dependency,
so run `checkrebuild -v` afterward.

## Enhancements to `pacman`

Uncomment `Color` line in `/etc/pacman.conf` to enable color ouput.

Enable `NewsOnUpgrade` in `/etc/paru.conf` to display unread Arch news before
upgrades performed through paru.

Alternatively, install `informant` if pacman transactions should be blocked
until unread news has been acknowledged. Using both is generally redundant.

Install `pacman-contrib`. It provides:

* `checkupdates` command: check for updates without the need for root
privilege used to sync database.
* `paccache.timer`: enable and start this to discard unused cached packages
weekly.
* `paccache` command: remove cached packages manually.

Install `pacman-cleanup-hook` (AUR) to run `paccache` after each `pacman`
transaction.

Install `archlinux-contrib` to get `checkservices` command. It runs `pacdiff`
to merge `.pacnew` files then checks for processes running with outdated
mapped executable/library files and prompts the user if they want them to be restarted.

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

Install `gwenview`. Install `ffmpegthumbs` to generate video thumbnails.

## Chromium

Create `${XDG_CONFIG_HOME}/chromium-flags.conf` with the following content:

```
--enable-features=AcceleratedVideoEncoder
```

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

Install `fcitx5-unikey` and `fcitx5-im`. Add to `/etc/environment` the following
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

* Set `GTK_THEME` environment variable to use other theme (e.g. `Adwaita:dark`).
* Go to System Settings > Appearance > Application Style > Configure GNOME/GTK
Application Style... and select other theme for GTK apps. To have Adwaita
theme in the drop-down list you may have to install `gnome-themes-extra`.

# Misc

## Enable REISUB magic SysRq

Create file `/etc/sysctl.d/99-enable-sysrq.conf` with the following content:

```
kernel.sysrq=244
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
