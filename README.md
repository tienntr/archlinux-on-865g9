# Initial installation

Use `archinstall`. You may need to fix Windows EOL in generated `/etc/fstab`.

This note assume you installed KDE on Wayland and use `systemd-boot`.

# Network

## Firewall

Install `iptables-nft` (this removes `iptables`) to use newer nftable backend.
Install `firewalld`, enable and start `firewalld.service`.

## DNS caching

Install `dnsmasq`, create `/etc/NetworkManager/conf.d/dns.conf` with the
following content:

```
[main]
dns=dnsmasq
```

Then run `nmcli general reload` as root. NetworkManager runs its own instance
of `dnsmasq` that listen on `127.0.0.1:53`.

## Wireless

Install `wireless-regdb` and uncomment correct country in
`/etc/conf.d/wireless-regdom`.

## Sharing Internet via Wi-Fi with NetworkManager

Need `dnsmasq` to work. NetworkManager run its own instance of `dnsmasq` as
DHCP server.

# AUR

Install `git`, `pigz` and `pbzip2 `.

Edit `/etc/makepkg.conf` to enable parallel `make` and compression:

```
MAKEFLAGS="-j8"
...
COMPRESSGZ=(pigz -p 8 -c -f -n)
COMPRESSBZ2=(pbzip2 -p8 -c -f)
COMPRESSXZ=(xz -c -z --threads=8 -)
COMPRESSZST=(zstd -c -z -q --threads=8 -)
```

Install `yay-bin` by cloning and `makepkg -si`.

# Fonts

## Install MS fonts

```
yay -S ttf-ms-win11-auto
```

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

# Misc

## Wayland clipboard

Install `wl-clipboard` to use `wl-copy` and `wl-paste`.
