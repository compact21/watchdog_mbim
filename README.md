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
I suggest you install the "at" package ...

### Add "at" package with the commands:
   
```
opkg update
opkg install at
```

or create your own custom firmware with the "at" command already inserted in the image:
<br/>
https://firmware-selector.openwrt.org/?version=24.10.1&target=ramips%2Fmt7621&id=zyxel_lte5398-m904

a example from forum how to do it:
<br/>
https://forum.openwrt.org/t/backup-full-firmware/172133


### Edit the /etc/rc.local file which will look something like this:

cat /etc/rc.local
```
# Put your custom commands here that should be executed once
# the system init finished. By default this file does nothing.

echo "service watchdog_mbim start" | at now+10minutes

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

----------------------------------------------------------------------------------------------------------------------------------------------

<b>
I keep the documentation (see below) for a daemon that starts at boot and runs all the time,
but I don't recommend it,
as it would be better if the first connection was made without any interference (see above)
</b>

<br/>
<br/>

<s>
## Watchdog service always running ...

Service always active/enabled and always run (a real script daemon)

### What to do:

```
cd /tmp
wget --no-hsts https://raw.githubusercontent.com/compact21/watchdog_mbim/refs/heads/main/watchdog_mbim_service
wget --no-hsts https://raw.githubusercontent.com/compact21/watchdog_mbim/refs/heads/main/watchdog_mbim_script
mv watchdog_mbim_service /etc/init.d/watchdog_mbim
mv watchdog_mbim_script /root/watchdog_mbim
chmod +x /etc/init.d/watchdog_mbim
chmod +x /root/modem_watchdog_mbim
echo "/root/" >> /etc/sysupgrade.conf
echo "/etc/init.d/watchdog_mbim" >> /etc/sysupgrade.conf
service watchdog_mbim enable
service watchdog_mbim start
```

</s>
