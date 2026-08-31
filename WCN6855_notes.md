# QCNFA765/WCN6855 regulatory domain issues

The safest approach is to treat this QCNFA765 as unsuitable for unattended AP use until its firmware accepts `VN`. Client use on correctly configured Vietnamese networks is much lower risk.

This is technical risk reduction, not legal certification. For authoritative compliance, use Vietnamese regulations and certified equipment. Vietnam’s current plan permits WLAN/RLAN in 5150–5350, 5470–5725, 5725–5850 and 5925–6425 MHz. The 6 GHz rules include specific indoor/outdoor EIRP and PSD limits. [Vietnam spectrum plan](https://thuvienphapluat.vn/van-ban/Cong-nghe-thong-tin/Quyet-dinh-37-2025-QD-TTg-Quy-hoach-pho-tan-so-vo-tuyen-dien-quoc-gia-675645.aspx), [6 GHz amendment](https://thuvienphapluat.vn/van-ban/Cong-nghe-thong-tin/Thong-tu-01-2025-TT-BKHCN-sua-doi-noi-dung-tai-Phu-luc-kem-theo-Thong-tu-08-2021-TT-BTTTT-650256.aspx)

## 1. Keep Linux configured for Vietnam

Retain:

```bash
WIRELESS_REGDOM="VN"
```

in `/etc/conf.d/wireless-regdom`.

After boot, check:

```bash
iw reg get
```

You should continue to see:

```text
global
country VN
```

The separate self-managed US domain will remain until the firmware problem is fixed. Do not configure Linux as `US` merely to make the two outputs agree.

## 2. Use conservative common channels

These choices stay inside both the currently reported VN and US ranges.

| Band | Conservative choice | Notes |
|---|---:|---|
| 2.4 GHz | Channels 1, 6 or 11 | Use 20 MHz width |
| 5 GHz | Channels 36, 40, 44 or 48 | Indoor use; channel 36 is the simplest AP choice |
| 5 GHz DFS | Avoid for your laptop AP | Requires working radar detection and country handling |
| Upper 5 GHz | Avoid unless necessary | More opportunity for domain-specific differences |
| 6 GHz | Do not use this laptop as an AP | Firmware exposes US-only frequencies above 6425 MHz |

For your home router, a safe straightforward arrangement is:

- 2.4 GHz: channel 1, 6 or 11; 20 MHz
- 5 GHz: channel 36–48
- 6 GHz: use only a router certified/configured for Vietnam

Your current client connection on channel 44 at 20 dBm is within these conservative parameters.

## 3. Client-mode precautions

For networks you control:

1. Configure the router’s country/region as Vietnam.
2. Disable automatic region selection if the router offers it.
3. Select one of the conservative channels above.
4. Keep router firmware current.
5. Confirm the connection afterward:

```bash
iw dev wlan0 link
```

Check the `freq` value:

```text
2412 MHz  → channel 1
2437 MHz  → channel 6
2462 MHz  → channel 11
5180 MHz  → channel 36
5200 MHz  → channel 40
5220 MHz  → channel 44
5240 MHz  → channel 48
```

For a fixed home network, you can lock the NetworkManager profile to its band and BSSID:

```bash
nmcli connection modify "PROFILE" \
    802-11-wireless.band a \
    802-11-wireless.bssid AA:BB:CC:DD:EE:FF
```

This reduces the chance of roaming to another band or AP. It is not a regulatory enforcement mechanism, and locking a BSSID disables normal roaming.

On public networks, ordinary client operation is relatively low risk because the AP selects the channel and usually advertises power constraints. Nevertheless, avoid connecting to 6 GHz networks operating above 6425 MHz.

## 4. Safest AP recommendation

For regular or unattended hotspot use, use different hardware:

- A Vietnam-market access point/router
- A Wi-Fi adapter whose driver reports `country VN`
- Internet sharing over Ethernet to a compliant external AP

This is the only high-confidence solution because software cannot presently guarantee that the QCNFA765 obeys VN rules.

If the laptop hotspot is only occasional, use the constrained procedure below.

## 5. Create a constrained NetworkManager hotspot

Do not use the one-click hotspot without inspecting its saved channel. Automatic selection may choose from the incorrect US channel list.

Create a 5 GHz, channel 36, 20 MHz profile:

```bash
nmcli connection add \
    type wifi \
    ifname wlan0 \
    con-name vn-hotspot \
    ssid YOUR_SSID
```

Then constrain it:

```bash
nmcli connection modify vn-hotspot \
    802-11-wireless.mode ap \
    802-11-wireless.band a \
    802-11-wireless.channel 36 \
    802-11-wireless.channel-width 20mhz \
    ipv4.method shared \
    ipv6.method disabled
```

Configure WPA2/RSN with CCMP:

```bash
nmcli connection modify vn-hotspot \
    802-11-wireless-security.key-mgmt wpa-psk \
    802-11-wireless-security.proto rsn \
    802-11-wireless-security.pairwise ccmp \
    802-11-wireless-security.group ccmp \
    802-11-wireless-security.psk 'YOUR_STRONG_PASSWORD'
```

Entering the password this way may put it in shell history. Prefer NetworkManager’s GUI or an interactive secret mechanism if that matters.

NetworkManager documents that `band` and `channel` lock an AP profile to the specified band/channel, while `channel-width` controls AP width. [NetworkManager settings](https://www.networkmanager.dev/docs/api/latest/nm-settings-nmcli.html)

Activate it:

```bash
nmcli connection up vn-hotspot
```

## 6. Apply a conservative transmit-power ceiling

After activating the hotspot:

```bash
sudo iw dev wlan0 set txpower limit 2000
```

`2000` mBm means 20 dBm. The `iw` interface takes power in mBm, with 100 mBm equal to 1 dBm. [Linux wireless documentation](https://wireless.docs.kernel.org/en/latest/en/users/documentation/iw.html)

Use `limit`, not `fixed`: a limit permits the driver to use less power when appropriate.

Check the reported result:

```bash
iw dev wlan0 info
```

Look for something like:

```text
type AP
channel 36 (5180 MHz), width: 20 MHz
txpower 20.00 dBm
```

Important limitations:

- NetworkManager or the driver may reset the power setting when the connection is restarted.
- Firmware may clamp or interpret the request internally.
- Reported power is not a calibrated RF measurement.
- Legal limits are EIRP and can involve antenna gain and spectral-density constraints.

Reapply and verify the limit every time the AP is activated. If the driver continues reporting more than 20 dBm, stop the hotspot.

## 7. Verify every hotspot activation

Use this checklist:

```bash
iw reg get
iw dev wlan0 info
journalctl -k --since "2 minutes ago" |
    grep -iE 'ath11k|regulatory|country|radar'
```

Confirm:

- Global domain is `VN`.
- Interface type is `AP`.
- Frequency is exactly 5180 MHz/channel 36.
- Width is 20 MHz.
- Reported power is no more than 20 dBm.
- No channel-switch event moved the AP elsewhere.
- No regulatory or radar errors appeared.

NetworkManager profile confirmation:

```bash
nmcli -f \
connection.id,802-11-wireless.mode,802-11-wireless.band,802-11-wireless.channel,802-11-wireless.channel-width \
connection show vn-hotspot
```

Never leave `channel=0`, automatic channel selection, or ACS enabled with this card.

## 8. A 2.4 GHz hotspot alternative

For maximum client compatibility:

```bash
nmcli connection modify vn-hotspot \
    802-11-wireless.band bg \
    802-11-wireless.channel 6 \
    802-11-wireless.channel-width 20mhz
```

Then activate and reapply the 20 dBm ceiling:

```bash
nmcli connection up vn-hotspot
sudo iw dev wlan0 set txpower limit 2000
```

Channels 1, 6 and 11 are available under both domains. Do not use automatic channel selection, and do not expect channels 12 or 13 to work because the firmware disables them.

## 9. Avoid DFS channels for AP mode

Do not configure this laptop’s hotspot on channels 52–64 or 100–144.

Those ranges require DFS/radar detection. Although ath11k advertises radar detection, the incorrect country negotiation creates uncertainty about:

- Detection parameters
- Channel-availability checks
- Non-occupancy periods
- Country-specific power limits

There is no benefit worth that additional uncertainty for a temporary hotspot.

## 10. Avoid 6 GHz AP mode entirely

Do not use:

```text
802-11-wireless.band 6GHz
```

with this card.

The firmware exposes channels through 7125 MHz, but Vietnam’s current unlicensed WLAN allocation ends at 6425 MHz. Vietnam also imposes:

- Indoor/outdoor distinctions
- EIRP limits
- PSD limits
- A guard segment at 5925–5945 MHz
- Restrictions involving drones/UAS

A simple `iw set txpower` command does not enforce PSD, indoor operation or the upper frequency boundary. Even manually choosing a nominally valid 6 GHz channel would leave too much dependent on the incorrect firmware domain.

## 11. Do not rely on these as safeguards

The following are not sufficient:

- `iw reg set VN`: the firmware currently rejects it.
- `WIRELESS_REGDOM=VN` alone: it fixes the global domain, not the PHY.
- The AP’s country information element alone.
- Automatic channel selection.
- A reported `txpower` value without checking the selected channel.
- Selecting another country that “looks similar.”
- Editing Qualcomm’s `regdb.bin`.

## 12. Recheck after updates

After updates to any of these:

- `linux`
- `linux-firmware-atheros`
- `wireless-regdb`
- BIOS/UEFI

reboot and run:

```bash
iw reg get
journalctl -b -k |
    grep -iE 'ath11k|regulatory|country'
```

The issue can be considered fixed only when the per-PHY result becomes compatible with VN and the rejection disappears, for example:

```text
phy#0 (self-managed)
country VN
```

Until then, the operational rule is simple: client mode on a correctly configured router is acceptable with verification; for AP mode, fix channel 36 or a 1/6/11 channel, use 20 MHz and a 20 dBm ceiling, verify after every activation, and prefer separate Vietnam-compliant AP hardware whenever possible.