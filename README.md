# Modem and network restart daemon for connecting with proto MBIM for Openwrt.

# I test on my Openwrt LTE5398-M904

# Thanks to the work of "yosh781" on "https://github.com/yosh781"
# Thanks "@antonk" to the support from "https://forum.openwrt.org/t/create-a-sample-procd-init-script/230977"

# Move script "watchdog_mbim" in /etc/init.d/watchdog_mbim

chmod +x /etc/init.d/watchdog_mbim

# Move script "watchdog_mbim_script" in /root

# I prefer to put all the scripts in root as I don't need to add anything else to /etc/sysupgrade.conf
echo "/root/" >> /etc/sysupgrade.conf
# so that they are preserved by sysupgrade

chmod +x /root/modem_watchdog_mbim

# Enabe and start service

service watchdog_mbim enable
service watchdog_mbim start
