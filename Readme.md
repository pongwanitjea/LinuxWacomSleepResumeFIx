On all Linux distros, some touchscreen with Wacom pen laptop when sleep and then resume. Both touchscreen and stylus input will not working and require restart.
Until Claude Sonnet finally cracked this problem on Fujitu U9310X.

Change 1-9.1 to be your own by exploring /sys/bus/usb/devices/
Change 0003:056A:51DD to be your own by ls in /sys/bus/hid/drivers/wacom/

## systemd-sleep Hook: The Reliable Long-Term Fix

Since every sysfs rebind attempt you've tried fails because the firmware is stuck, the correct approach is to **cut power to the device before suspend** so the firmware gets a proper reset on resume. This is the approach that worked for `bjgroom` on a similar Thinkpad Yoga with the same error.

### Pre-suspend: Unbind the xHCI driver (force USB power cut)

```bash
sudo nano /etc/systemd/system-sleep/wacom-reset
```

```bash
#!/bin/bash
WACOM_USB_PATH="/sys/bus/usb/devices/1-9.1"

pre_suspend() {
    logger -t wacom-reset "SUSPEND starting"
    for hid in /sys/bus/hid/drivers/wacom/0003:056A:51DD.*; do
        [ -e "$hid" ] && echo -n "$(basename "$hid")" > /sys/bus/hid/drivers/wacom/unbind
    done
    modprobe -r wacom
}

post_resume() {
    logger -t wacom-reset "RESUME starting, waiting for device..."
    for i in $(seq 1 40); do
        if [ -e "$WACOM_USB_PATH" ] && [ -e "$WACOM_USB_PATH/idVendor" ]; then
            logger -t wacom-reset "device found after $i checks"
            sleep 1
            break
        fi
        sleep 0.5
    done
    modprobe wacom
    logger -t wacom-reset "RESUME finished, wacom module loaded"
}

case $1 in
    pre)  pre_suspend ;;
    post) post_resume ;;
esac
```

```bash
sudo chmod +x /etc/systemd/system-sleep/wacom-reset
```

The 3-second delay in `post_resume` is the key difference from your earlier failed `modprobe -r wacom; sleep 5; modprobe wacom` attempt — previously you likely ran that with the device already in a zombie state mid-session, not as a clean pre/post suspend hook.

***



### You may need to allow script execution for SELinux or script need to be in `/usr/lib/systemd/system-sleep/wacom-reset`

sudo cp /etc/systemd/system-sleep/wacom-reset /usr/lib/systemd/system-sleep/wacom-reset
sudo chmod +x /usr/lib/systemd/system-sleep/wacom-reset
sudo restorecon -v /usr/lib/systemd/system-sleep/wacom-reset
