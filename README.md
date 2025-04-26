# Modem and network restart daemon for connecting with proto MBIM for Openwrt.

I test on my Openwrt LTE5398-M904

Thanks to the work of "yosh781" on "https://github.com/yosh781"

Thanks "@antonk" to the support from "https://forum.openwrt.org/t/create-a-sample-procd-init-script/230977"

If anyone wants to contribute, they know how to contact me... (see Openwrt forum)

I prefer to put all the scripts in root as I don't need to add anything else
to /etc/sysupgrade.conf so that they are preserved by sysupgrade

What to do:

Move script "watchdog_mbim" in /etc/init.d/watchdog_mbim

chmod +x /etc/init.d/watchdog_mbim

Move script "watchdog_mbim_script" in /root

chmod +x /root/modem_watchdog_mbim

Add "/root/" and "/etc/init.d/watchdog_mbim" to /etc/sysupgrade.conf preserved sysupgrade

echo "/root/" >> /etc/sysupgrade.conf ; echo "/etc/init.d/watchdog_mbim" >> /etc/sysupgrade.conf

Enabe and start service

service watchdog_mbim enable; service watchdog_mbim start
