# Watchdog for restart connecting with proto MBIM for Openwrt.

I test on my Openwrt LTE5398-M904

Thanks to the work of "yosh781" from "https://github.com/yosh781/Daemon-modem-watchdog_qmi-for-Openwrt"

Thanks "@antonk" to the support from "https://forum.openwrt.org/t/create-a-sample-procd-init-script/230977"

----------------------------------------------------------------------------------------------------------------------------------------------

I prefer to put all the scripts in "/root" directory

Add "/root" + "/etc/init.d/watchdog_mbim" into /etc/sysupgrade.conf so that they are preserved by sysupgrade

----------------------------------------------------------------------------------------------------------------------------------------------

<b>
I advice starting the daemon 10 minutes after the router boots,

so that there is no interference during the first connection
</b>

see this document:

## Watchdog disable service and start this after a certain time ...

If you want, as in my case, to have the daemon start after a certain time from the router startup,
since the system date is assumed to be the date of the most recent file found on the partition
show this document: "https://forum.openwrt.org/t/openwrt-out-of-sync-even-though-ntp-is-enabled/234493/6"
you can opt for a "long sleep"

I suggest you edit /etc/rc.local like this:

### Edit the /etc/rc.local file which will look something like this:

cat /etc/rc.local
```
# Put your custom commands here that should be executed once
# the system init finished. By default this file does nothing.

sleep 600
service watchdog_mbim start

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
