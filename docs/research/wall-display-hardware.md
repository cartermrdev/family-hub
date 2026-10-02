# Wall Display hardware (13" or larger)

Research for issue #6. Checked 2026-10-01. Prices are approximate US list prices from the cited page on that date. They are moving fast in 2026, especially Raspberry Pi prices because of the DRAM shortage.

## Answer

**Recommendation (the human decides in "Which Wall Display hardware and orientation?"):** a **Raspberry Pi 5 (2–4 GB) driving a 13–24" USB touch monitor, running Chromium in `--kiosk` mode**. It is the cheapest option at 15"+, it has no battery to wear out on always-on power, and screen blanking and display on/off are documented and scriptable. The trade-off is more setup.

The **13" iPad Air (LCD)** is the low-effort runner-up. Guided Access gives a good-enough kiosk in minutes, and the 80% charge limit reduces battery wear. Its weak points are price (~$949), a sealed battery on 24/7 power, and getting kiosk mode to survive a restart, which needs a supervised device. Avoid the **13" iPad Pro** for this job: it costs more and has an OLED panel. Android panels work well with Fully Kiosk Browser, but commercial all-in-ones (Elo) are expensive and ship old Android.

## Comparison

| | 13" iPad Air (M4) | 13" iPad Pro (M5) | Raspberry Pi 5 + touch monitor | Android tablet (e.g. Galaxy Tab S10 FE+ 13.1") | Commercial Android all-in-one (Elo I-Series 4, 15.6"/21.5") |
|---|---|---|---|---|---|
| Approx. cost | from $949 [A3] | from $1,499 [A5] | Pi 5 2GB $77.50 [R3] + monitor: 13.3" Waveshare $173 [W1] or 24" Dell P2424HT $425 [D1], plus PSU, case and storage. About $300–550 total | from $649.99 [S1] + Fully Kiosk PLUS licence (price not captured) | Price not listed on manufacturer page [E1] (flag) |
| Screen | 12.9" LCD (IPS), 600 nits [A4] | 13" tandem OLED, 1000 nits SDR [A6] | 13.3" IPS 1080p [W1] / 23.8" IPS 1080p 300 nits [D1] | 13.1" 2880×1800 [S1] | 15.6"/21.5" LCD 1080p, ~215–258 nits with touch [E1][E2] |
| Kiosk mode | Guided Access (built in, quick) [A1]. App Lock / Single App Mode needs a supervised device (Apple Configurator or MDM) [A2][A7] | Same as Air | Chromium `--kiosk` started from labwc autostart [R5]. The OS is yours to lock down | Fully Kiosk Browser. Full lock (Lock Task Mode) needs Device Owner provisioning [F1][G1] | Android + Fully Kiosk / Lock Task. Vendor tooling (EloView / OS 360) [E1] |
| Burn-in risk | Low (LCD) | Higher (OLED). Apple does not document it (flag) | Low (IPS LCD) | Low (LCD, assumed; flag) | Low (LCD) |
| Screen on/off scheduling | Auto-Lock setting during Guided Access [A1]. No built-in time schedule (flag) | Same | `wlr-randr` display on/off plus cron, and labwc blanking timeout [R4] | Fully Kiosk PLUS: scheduled wake/sleep, motion wake, brightness [F1] | Via Fully Kiosk / vendor tooling |
| Always-on power | Sealed battery. 80% Limit supported on Air M2+ / Pro M4+ [A8] | Same | No battery. 5V/5A USB-C PSU [R1]. Monitor runs on mains | Sealed battery (flag: battery-protect options not verified) | Mains or PoE via adapter [E1] |
| Mounting | Third-party wall mount (no VESA) | Same | VESA 100×100 on the Dell [D1]. Pi mounts behind it | Third-party mount | VESA 75/100 [E1] |
| Runs the web app as | Safari, or a Home Screen "Open as Web App" [A9] | Same | Chromium (full desktop browser) | Fully Kiosk's Android System WebView | Android WebView. Ships Android 10, with upgrades through a paid subscription [E1] |

## Details

### iPad (13" Air or Pro)

- **Running the web app.** In Safari, Share → Add to Home Screen, then turn on "Open as Web App" so it runs without Safari's interface [A9].
- **Guided Access.** Triple-click the top button to start it, and triple-click plus passcode to leave. You can set "Display Auto-Lock" for the session and an optional time limit [A1]. It is quick and free, and needs no Mac or MDM.
  - **Restart risk (flag).** Apple's docs do not say whether Guided Access survives a restart or an OS update. Assume it does not. After a power cut, someone has to re-enter it.
- **Single App Mode / App Lock.** Apple Configurator's Single App Mode can disable touch, buttons, Sleep/Wake and Auto-Lock, and requires a supervised device [A2]. The MDM "App Lock" restriction also requires supervision [A7].
  - Supervision means wiping the iPad and enrolling it through Apple Configurator (Mac) or MDM. That is a lot of overhead for one household device.
  - Apple's pages do not state the restart behaviour (flag).
- **Always-on power.** Apple says it is safe to leave an iPad connected. "80% Limit" stops charging at about 80% and resumes at 75% on iPad Pro (M4)+, iPad Air (M2)+, iPad mini (A17 Pro) and iPad (A16) [A8]. Long-term swelling risk on a permanently charged sealed battery is not addressed by Apple (flag).
- **Burn-in.** The 13" iPad Pro uses tandem OLED [A6]. A static dashboard is the classic burn-in risk for OLED, and Apple publishes no guidance (flag). The Air's IPS LCD [A4] avoids that worry.
- **Scheduling.** There is no documented built-in "screen off at 23:00" schedule (flag). The app can dim itself (dark theme at night), and the screen stays on per the Auto-Lock setting.
- **Cost.** 13" Air from $949 [A3]. 13" Pro from $1,499 [A5].

### Raspberry Pi 5 + touchscreen

- **No official screen is big enough.** Raspberry Pi's own Touch Display 2 comes only in 5", 7" and 10" ($40/$60/$80) [R2]. You need a third-party HDMI display with USB touch:
  - Waveshare 13.3" IPS 1080p, 10-point capacitive over USB, 12V supply, $172.99 [W1]
  - Dell P2424HT 23.8" IPS 1080p, 300 nits, touch over USB-C, VESA 100, about 18W typical, $424.99 [D1]
- **Kiosk.** The official tutorial uses Raspberry Pi OS (64-bit) and Chromium with `--kiosk --noerrdialogs --disable-infobars --no-first-run`, launched from `~/.config/labwc/autostart` [R5].
  - Because it is a full Linux box, auto-login, a systemd watchdog to restart Chromium, and remote SSH are all available. Those are standard Linux practice; the official tutorial does not cover auto-login (flag).
- **Blanking and scheduling.** Screen blanking can be toggled in raspi-config, and its timeout is set in the labwc autostart. `wlr-randr` turns a display on and off under Wayland [R4], so a cron job can switch the screen off overnight.
- **Power.** The Pi 5 needs 5V/5A (or 5V/3A with peripheral limits) over USB-C [R1]. With no battery, it suits 24/7 use. An SD card can be corrupted by power cuts; that is general practice rather than documented here (flag), so prefer a read-only root or NVMe.
- **Cost volatility.** Raspberry Pi raised Pi 5 prices in Feb, Apr and Oct 2026 because of LPDDR4 costs [R3][R6][R7].
  - The 2GB model is $77.50 as of 2026-10-01 [R3], and the 16GB model is $305 [R8].
  - A kiosk only needs 2–4 GB. The current 4GB price was not stated on an official page (flag).

### Android tablets and all-in-one panels

- **Android tablet.** The Galaxy Tab S10 FE+ (13.1", 2880×1800, from $649.99) [S1] is an example. It runs a browser kiosk app such as Fully Kiosk Browser [F1], which offers:
  - PIN-protected kiosk lockdown
  - scheduled screen wake/sleep and brightness
  - motion-detection wake
  - auto-reload
  - remote admin on port 2323
  - Most of these are in the paid PLUS licence; the price was not captured (flag).
- **Hard lock.** Android's Lock Task Mode hides the status bar, Home and Overview and blocks other apps. It needs a Device Policy Controller (device owner) [G1]. Fully Kiosk supports Device Owner provisioning [F1], which typically requires a factory reset.
- **Commercial all-in-ones.** The Elo I-Series 4 for Android (15.6" or 21.5") [E1][E2] offers:
  - 1080p screen, 10-touch PCAP
  - VESA 75/100, PoE through an adapter, Ethernet
  - Android 10, with upgrades to 12/14 through the OS 360 subscription
  - It is commercial-grade and built for wall mounting, but the price is not listed (quote-based; flag), and Android 10 is old for a modern web app.

## Risks

- **Prices.** Pi and RAM prices changed three times in 2026. Re-check before buying.
- **iPad kiosk after a power cut.** Recovery is undocumented. Without supervision, plan for manual re-entry into Guided Access.
- **OLED burn-in** on the iPad Pro is undocumented by Apple, and a static dashboard is a high-risk use for OLED.
- **Sealed batteries on 24/7 power.** This applies to the iPad and Android tablet options. Long-term behaviour beyond the 80% limit is not documented.
- **Portrait orientation.** Whether the Dell stand pivots, and whether the Waveshare touch mapping supports rotation, was not verified. This matters for the later orientation decision (flag).
- **Brightness.** The touch-layer panels (Elo ~215–258 nits [E2], Dell 300 nits [D1]) are dimmer than an iPad (600–1000 nits). That is fine indoors, but check it against a bright room.

## Sources

- [A1] Apple, Use Guided Access with iPhone/iPad. https://support.apple.com/en-us/111795
- [A2] Apple Configurator for Mac, Start, stop or restart devices (Single App Mode options, supervised). https://support.apple.com/guide/apple-configurator-mac/start-stop-or-restart-devices-cadb1d640325/mac
- [A3] Apple Store, Buy iPad Air (13" from $949). https://www.apple.com/shop/buy-ipad/ipad-air
- [A4] Apple, iPad Air tech specs. https://www.apple.com/ipad-air/specs/
- [A5] Apple Store, Buy iPad Pro (13" from $1,499). https://www.apple.com/shop/buy-ipad/ipad-pro
- [A6] Apple, iPad Pro tech specs. https://www.apple.com/ipad-pro/specs/
- [A7] Apple Platform Deployment, restrictions (App Lock requires supervision). https://support.apple.com/guide/deployment/app-lock-payload-settings-dep0f7dd3d8/web
- [A8] Apple, About charging and maintaining your iPad battery. https://support.apple.com/en-us/118418
- [A9] Apple, Bookmark a website in Safari on iPad (Add to Home Screen, Open as Web App). https://support.apple.com/guide/ipad/bookmark-a-website-ipadc602b75b/ipados
- [R1] Raspberry Pi documentation, Raspberry Pi hardware (power supply). https://www.raspberrypi.com/documentation/computers/raspberry-pi.html
- [R2] Raspberry Pi, Touch Display 2 product page. https://www.raspberrypi.com/products/touch-display-2/
- [R3] Raspberry Pi news, Price increases for 2GB Raspberry Pi 4 and Raspberry Pi 5 (2026-10-01). https://www.raspberrypi.com/news/price-increases-for-2gb-raspberry-pi-4-and-raspberry-pi-5/
- [R4] Raspberry Pi documentation, Configuration (screen blanking, wlr-randr). https://www.raspberrypi.com/documentation/computers/configuration.html
- [R5] Raspberry Pi tutorial, How to use a Raspberry Pi in kiosk mode. https://www.raspberrypi.com/tutorials/how-to-use-a-raspberry-pi-in-kiosk-mode/
- [R6] Raspberry Pi news, More memory-driven price rises (2026-02-02). https://www.raspberrypi.com/news/more-memory-driven-price-rises/
- [R7] Raspberry Pi news, A new 3GB Raspberry Pi 4 ... and more memory-driven price increases (2026-04-01). https://www.raspberrypi.com/news/a-new-3gb-raspberry-pi-4-for-83-75-and-more-memory-driven-price-increases/
- [R8] Raspberry Pi 5 product page (16GB $305). https://www.raspberrypi.com/products/raspberry-pi-5/
- [W1] Waveshare, 13.3inch HDMI LCD (H) with case. https://www.waveshare.com/13.3inch-hdmi-lcd-h-with-case.htm
- [D1] Dell, Dell Pro 24 Plus Touch USB-C Hub Monitor P2424HT. https://www.dell.com/en-us/shop/dell-pro-24-plus-touch-usb-c-hub-monitor-p2424ht/apd/210-bhsf/monitors-monitor-accessories
- [S1] Samsung, Galaxy Tab S10 FE+ buy page. https://www.samsung.com/us/tablets/galaxy-tab-s10-fe/buy/galaxy-tab-s10-fe-plus-blue-128gb-sku-sm-x620nlbaxar/
- [F1] Fully Kiosk Browser. https://www.fully-kiosk.com/en/
- [G1] Android Developers, Lock task mode. https://developer.android.com/work/dpc/dedicated-devices/lock-task-mode
- [E1] Elo, 15-inch Android I-Series 4. https://www.elotouch.com/touchscreen-computers-aaio4-15.html
- [E2] Elo, I-Series 4 for Android datasheet. https://docs.elotouch.com/Elo_I-Series-4_Android_EU.pdf
