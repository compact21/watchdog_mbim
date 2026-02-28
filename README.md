# Watchdog LTE MBIM/QMI for OpenWrt

Daemon to monitor LTE connections (MBIM/QMI) on OpenWrt.
Automatically restarts the WAN interface if the connection is lost.
Tested on **OpenWrt LTE5398-M904** (MBIM).

Credits:
**yosh781** – [Daemon-modem-watchdog_qmi-for-Openwrt](https://github.com/yosh781/Daemon-modem-watchdog_qmi-for-Openwrt),
**@antonk** – [OpenWrt forum: sample procd init script](https://forum.openwrt.org/t/create-a-sample-procd-init-script/230977).  

---

Preferences:
Store scripts in `/root/bin/`. Preserve files on upgrade by adding `/root` and `/etc/init.d/watchdog_mbim` to `/etc/sysupgrade.conf`.
**Start daemon ~10 min after boot** to avoid conflicts with first WAN initialization.  

---

How it works: checks LTE status via:

```
# MBIM
uqmi -m -d /dev/cdc-wdm0 -t 20000 --get-data-status
# QMI
uqmi -d /dev/cdc-wdm0 -t 20000 --get-data-status
```

Detects unresponsive modems. On failure, daemon safely restarts WAN. Example:

```
time uqmi -m -d /dev/cdc-wdm0 -t 20000 --get-data-status
"connected"
real    0m0.25s
user    0m0.00s
sys     0m0.00s
```

Variables:

```
ltestatus=$(uqmi -m -d /dev/cdc-wdm0 -t 20000 --get-data-status 2>/dev/null)  # "connected"
lteerror=$?                                                                   # 0
ltefind=$(echo "$ltestatus" | grep -c "\"connected\"")                        # 1
```

Delay start (~10 min) to avoid: system date corrections or WAN first connection issues.
References:
```
https://forum.openwrt.org/t/openwrt-out-of-sync-even-though-ntp-is-enabled/234493/6
https://forum.openwrt.org/t/retroactivelly-change-logs-timestamp/206872/5
https://forum.openwrt.org/t/kernel-timestamps-are-6-days-behind-on-reboot/139872
https://forum.openwrt.org/t/lte-o2-and-t-d1-work-but-vodafone-d2-does-not/43160
https://forum.openwrt.org/t/trouble-registering-private-apn-on-teltonika-rut955-with-openwrt/170960
```

Required packages in addition to those mentioned in this guide https://openwrt.org/docs/guide-user/network/wan/wwan/ltedongle:

```
- timeout       Command timeout handling
- at            Scheduling delayed tasks (atd)
- uqmi          MBIM/QMI modem management
- umbim         MBIM modem management
``` 

Install via:
```
opkg update
opkg install coreutils-timeout atd kmod-usb-net-qmi-wwan uqmi luci-proto-qmi kmod-usb-serial-option picocom kmod-usb-net-cdc-mbim umbim luci-proto-mbim
```

Setup: add delay start in /etc/rc.local:
```
# Put your custom commands here that should be executed once
# the system init finished. By default this file does nothing.

# Connectivity check function with delay
check_ping_with_delay() {
    target="$1"
    ping -c 5 -W 1 "$target" > /dev/null 2>&1
    status=$?

    if [ "$status" -eq "0" ]; then
        logger "ping $target successful"
        return 0
    else
        logger "ping $target failed, sleep 60"
        sleep 60
        return 1
    fi
}

# NTP update
check_ping_with_delay 9.9.9.9
status1=$?
if [ "$status1" -eq 0 ]; then
    logger "WAN is up, starting one-shot NTP synchronization"
    ntpd -nq -p pool.ntp.org > /dev/null 2>&1
    status2=$?
    if [ "$status2" -eq "0" ]; then
        logger "NTP sync successful"
        sleep 2
        /etc/init.d/sysntpd restart
        sleep 2
    else
        logger "NTP did not adjust system time"
    fi
else
    logger "WAN not ready, skipping NTP synchronization"
fi


# atd and watchdog_mbim scheduling
check_ping_with_delay www.google.it
status3=$?
if [ -x /etc/init.d/atd ] && [ "$status3" -eq "0" ] && [ -x /etc/init.d/watchdog_mbim ]; then
        /etc/init.d/atd reload
        logger "run echo \"/etc/init.d/watchdog_mbim start\" | at now+10minutes"
        echo "/etc/init.d/watchdog_mbim start" | at now+10minutes
else
        logger "Ping check failed, waiting 300s before starting watchdog"
        sleep 300
        /etc/init.d/watchdog_mbim start
fi

exit 0
```

Download & install scripts:

```
cd /tmp
wget --no-hsts https://raw.githubusercontent.com/compact21/watchdog_mbim/main/watchdog_mbim
wget --no-hsts https://raw.githubusercontent.com/compact21/watchdog_mbim/main/watchdog_mbim_script
wget --no-hsts https://raw.githubusercontent.com/compact21/watchdog_mbim/main/watchdog_mbim.conf
mv watchdog_mbim /etc/init.d/watchdog_mbim
mv watchdog_mbim.conf /etc/
mkdir -p /root/bin/
mv watchdog_mbim_script /root/bin/watchdog_mbim
chmod +x /etc/init.d/watchdog_mbim
chmod +x /root/bin/watchdog_mbim
echo "/root/" >> /etc/sysupgrade.conf
echo "/etc/init.d/watchdog_mbim" >> /etc/sysupgrade.conf
echo "/etc/watchdog_mbim.conf" >> /etc/sysupgrade.conf
service watchdog_mbim start
```
