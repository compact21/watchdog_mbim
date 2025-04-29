# Modem and network restart daemon for connecting with proto MBIM for Openwrt.

I test on my Openwrt LTE5398-M904

Thanks to the work of "yosh781" on "https://github.com/yosh781"

Thanks "@antonk" to the support from "https://forum.openwrt.org/t/create-a-sample-procd-init-script/230977"

If anyone wants to contribute, they know how to contact me... (see Openwrt forum)

I prefer to put all the scripts in root as I don't need to add anything else
to /etc/sysupgrade.conf so that they are preserved by sysupgrade

What to do:


<code>
cd /tmp
wget --no-hsts https://raw.githubusercontent.com/compact21/watchdog_mbim/refs/heads/main/watchdog_mbim
wget --no-hsts https://raw.githubusercontent.com/compact21/watchdog_mbim/refs/heads/main/watchdog_mbim_script
mv watchdog_mbim /etc/init.d/watchdog_mbim
mv watchdog_mbim_script /root/watchdog_mbim
chmod +x /etc/init.d/watchdog_mbim
chmod +x /root/modem_watchdog_mbim
echo "/root/" >> /etc/sysupgrade.conf
echo "/etc/init.d/watchdog_mbim" >> /etc/sysupgrade.conf
service watchdog_mbim enable
service watchdog_mbim start
</code>
