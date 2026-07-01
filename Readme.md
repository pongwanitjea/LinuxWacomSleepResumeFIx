On all Linux distros, some touchscreen with Wacom pen laptop when sleep and then resume. Both touchscreen and stylus input will not working and require restart.
Until Claude Sonnet finally cracked this problem on Fujitu U9310X.

Change 0003:056A:51DD to be your own by ls in /sys/bus/hid/drivers/wacom/

## systemd-sleep Hook: The Reliable Long-Term Fix

Since every sysfs rebind attempt you've tried fails because the firmware is stuck, the correct approach is to **cut power to the device before suspend** so the firmware gets a proper reset on resume. This is the approach that worked for `bjgroom` on a similar Thinkpad Yoga with the same error.

### Pre-suspend: Unbind the xHCI driver (force USB power cut)

```bash
sudo nano /etc/systemd/system-sleep/wacom-reset.sh
```

```bash
#!/bin/bash
WACOM_HID_PRE=""  # will be populated dynamically

pre_suspend() {
    # Unbind both wacom HID interfaces
    for hid in /sys/bus/hid/drivers/wacom/0003:056A:51DD.*; do
        [ -e "$hid" ] && echo -n "$(basename $hid)" > /sys/bus/hid/drivers/wacom/unbind
    done
    # Remove wacom module entirely to prevent race on rebind
    modprobe -r wacom
}

post_resume() {
    # Give USB firmware time to settle (critical: this is the fix for -110)
    sleep 3
    modprobe wacom
}

case $1 in
    pre)  pre_suspend ;;
    post) post_resume ;;
esac
```

```bash
sudo chmod +x /etc/systemd/system-sleep/wacom-reset.sh
```

The 3-second delay in `post_resume` is the key difference from your earlier failed `modprobe -r wacom; sleep 5; modprobe wacom` attempt — previously you likely ran that with the device already in a zombie state mid-session, not as a clean pre/post suspend hook.

***
