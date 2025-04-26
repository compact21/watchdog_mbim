# watchdog_mbim

Modem and network restart daemon for connecting with proto MBIM for Openwrt.

Soon I will test on my Openwrt LTE5398-M904

Thanks to the work of "yosh781" on "https://github.com/yosh781"

Move script "watchdog_mbim" in /etc/init.d/watchdog_mbim

chmod +x /etc/init.d/watchdog_mbim

Move script "watchdog_mbim_script" in /usr/bin/watchdog_mbim

chmod +x /usr/bin/modem_watchdog_mbim

Enabe and start service

service watchdog_mbim enable; service watchdog_mbim start

