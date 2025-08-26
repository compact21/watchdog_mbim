# Watchdog for restart connecting with proto MBIM or QMI for Openwrt.

I test on my Openwrt LTE5398-M904 with the LTE module in MBIM

Thanks to the work of "yosh781" from "https://github.com/yosh781/Daemon-modem-watchdog_qmi-for-Openwrt"

Thanks "@antonk" to the support from "https://forum.openwrt.org/t/create-a-sample-procd-init-script/230977"

----------------------------------------------------------------------------------------------------------------------------------------------

### Some of my preferences

I prefer to put all the scripts in "/root" directory

Add "/root" + "/etc/init.d/watchdog_mbim" into /etc/sysupgrade.conf so that they are preserved by sysupgrade

<b>
I advice starting the daemon 10 minutes after the router boots,

so that there is no interference during the first connection
</b>

If you notice any inconsistencies in the documentation, please let me know.

----------------------------------------------------------------------------------------------------------------------------------------------

### What is it based on?

It is based on returning the connection status of this command:

uqmi -m -d /dev/cdc-wdm0 -t 20000 --get-data-status (for MBIM)
</br>
uqmi -d /dev/cdc-wdm0 -t 20000 --get-data-status (for QMI)

A 20 second timeout has been added so that if it doesn't respond within that time,
an error is detected and internet connectivity is restarted.

Theoretically the "uqmi -m -d /dev/cdc-wdm0 -t 20000 --get-data-status" command should respond in:
```
time uqmi -m -d /dev/cdc-wdm0 -t 20000 --get-data-status
"connected"
real    0m 0.25s
user    0m 0.00s
sys     0m 0.00s
```

the normal state of important script variables:
```
ltestatus=$(uqmi -m -d /dev/cdc-wdm0 -t 20000 --get-data-status 2> /dev/null); # return "connected"
lteerror=$?; # return 0
ltefind=$(echo "$ltestatus" | grep -c "\"connected\""); # return "1"
```

----------------------------------------------------------------------------------------------------------------------------------------------

### Various scripts

Scripts have been added that can run commands before restarting the interface and after the interface has been restarted.

```
watchdog_predown_file="/tmp/watchdog/predown_file"
https://github.com/compact21/watchdog_mbim/blob/main/varius-scripts/predown_file

watchdog_postdown_file="/tmp/watchdog/postdown_file"
https://github.com/compact21/watchdog_mbim/blob/main/varius-scripts/postdown_file

watchdog_ping_test="/tmp/watchdog/ping_test"
https://github.com/compact21/watchdog_mbim/blob/main/varius-scripts/ping_test
```
These are not necessary for the daemon to function but can help detect reconnection time,
or in case you want to run specific commands when a loss of connectivity is detected.

----------------------------------------------------------------------------------------------------------------------------------------------

### Activates the watchdog service after 10 minutes of router startup

I preferred to activate the service after 10 minutes to avoid the following issues:

1. Correction of the system date, as the date is set with
   the most recent file found in the "/etc" directory.
   </br>
   https://forum.openwrt.org/t/openwrt-out-of-sync-even-though-ntp-is-enabled/234493/6
   </br>
   https://forum.openwrt.org/t/retroactivelly-change-logs-timestamp/206872/5
   </br>
   https://forum.openwrt.org/t/kernel-timestamps-are-6-days-behind-on-reboot/139872

3. The first connection may not align with the correct parameters defined by your "ISP."
   </br>
   https://forum.openwrt.org/t/lte-o2-and-t-d1-work-but-vodafone-d2-does-not/43160
   </br>
   https://forum.openwrt.org/t/trouble-registering-private-apn-on-teltonika-rut955-with-openwrt/170960

----------------------------------------------------------------------------------------------------------------------------------------------

## If you like how the service has been implemented here are the things that need to be done

### Edit the /etc/rc.local file which will look something like this:

cat /etc/rc.local
```
# Put your custom commands here that should be executed once
# the system init finished. By default this file does nothing.

logger "exec: sleep 60 for activated wan uplink"
sleep 60

ping -c 5 www.google.it
if [ $? -ne "0" ]; then
logger "exec: ping fail, sleep 120"
sleep 120
fi

ntpd -dnq -p pool.ntp.org > /tmp/boot/boot_ntpd 2>&1
if [ $? -eq "0" ]; then
sleep 5
/etc/init.d/sysntpd restart
sleep 120
    if [ -x /etc/init.d/watchdog_mbim ]; then
    sleep 300
    /etc/init.d/watchdog_mbim start
    fi
else
logger "exec: ntpd fail update date"
fi

exit 0
```

### What to do after:

```
cd /tmp
wget --no-hsts https://raw.githubusercontent.com/compact21/watchdog_mbim/refs/heads/main/watchdog_mbim
wget --no-hsts https://raw.githubusercontent.com/compact21/watchdog_mbim/refs/heads/main/watchdog_mbim_script
mv watchdog_mbim /etc/init.d/watchdog_mbim
mv watchdog_mbim_script /root/watchdog_mbim
chmod +x /etc/init.d/watchdog_mbim
chmod +x /root/modem_watchdog_mbim
echo "/root/" >> /etc/sysupgrade.conf
echo "/etc/init.d/watchdog_mbim" >> /etc/sysupgrade.conf
service watchdog_mbim start
```
