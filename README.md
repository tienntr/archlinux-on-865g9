# Fonts

## Install MS fonts

```
yay -S ttf-ms-win11-auto
```

## Fix jagged Calibri and Cambria fonts

Disable embedded bitmaps for these fonts by creating `/etc/fonts/local.conf` with the following content:

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
