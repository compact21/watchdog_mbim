# Watchdog LTE MBIM/QMI for OpenWrt

Daemon to monitor LTE connections (MBIM/QMI) on OpenWrt.<br>
Automatically restarts the WAN interface if the connection is lost.<br>
Tested on **OpenWrt LTE5398-M904** (MBIM).

Credits:
**yosh781** – [Daemon-modem-watchdog_qmi-for-Openwrt](https://github.com/yosh781/Daemon-modem-watchdog_qmi-for-Openwrt),
**@antonk** – [OpenWrt forum: sample procd init script](https://forum.openwrt.org/t/create-a-sample-procd-init-script/230977).

---

## How it works

Checks LTE status via `uqmi`:

```sh
# MBIM
uqmi -m -d /dev/cdc-wdm0 --get-data-status
# QMI
uqmi -d /dev/cdc-wdm0 --get-data-status
```

Detects both disconnected and unresponsive MBIM/QMI modems. On failure, the daemon safely restarts the WAN interface. Example:

```
time uqmi -m -d /dev/cdc-wdm0 --get-data-status
"connected"
real    0m 0.19s
user    0m 0.00s
sys     0m 0.00s
```

Variables set by the check:

```sh
lte_status=$(uqmi -m -d /dev/cdc-wdm0 --get-data-status 2>&1)         # "connected"
lte_rc=$?                                                              # 0
lte_connected=$(echo "$lte_status" | grep -c "\"connected\"")          # 1
```

> **Note:** stderr is redirected to stdout (`2>&1`) so that uqmi error messages
> are captured in `lte_status` and logged to syslog on failure.

---

## Connectivity monitoring

The daemon uses two independent checks:

1. LTE session status via uqmi
2. End-to-end connectivity via ICMP ping

A modem may report `"connected"` while Internet connectivity is lost.

In this case `ping_logger()` detects the failure and triggers the second-stage
recovery path (forced MBIM disconnect and escalation).

> **Note:** `ping_logger()` creates the flag file `/tmp/watchdog/ping_failed`. The main loop checks for its presence and triggers the second-stage recovery path.

```sh
# Forced MBIM disconnect if ping timeout detected
if [ -f "$ping_failed_flag" ]; then
```

---

## Recovery escalation

When connectivity is lost, the daemon escalates through the following steps:

1. **`safe_ifup`** — WAN down/up via ubus

2. **`umbim disconnect` + `safe_ifup`** — forced MBIM disconnect followed by a second
   WAN restart (triggered after `ping_timeout_sec` without response). After the second
   `safe_ifup`, `ping_verify()` polls connectivity for up to `ping_verify_timeout`
   seconds. Escalation to step 3 occurs only if `ping_verify()` times out.

3. **`atcmd -C`** — full modem restart via AT+CFUN=1,1 (requires `/root/bin/atcmd`).

   > **Note:** `atcmd` and `atcmd.usage` are available at
   > [compact21/openwrt-script](https://github.com/compact21/openwrt-script).
   >
   > The daily invocation limit is enforced by `atcmd` itself to preserve modem flash
   > lifespan. When the limit is reached, `atcmd -C` exits with code `11` and the
   > daemon logs `atcmd -C daily limit reached, skipping CFUN escalation`, then
   > proceeds directly to step 4.

4. **`usbreset`** — USB hardware reset (requires `/usr/bin/usbreset`).

   > **Note:** `usbreset` is provided by the `usbutils` package.

Each level is attempted only once per disconnect event. All flags reset when WAN comes back online.


---

## Boot grace period

The daemon must not interfere with the first WAN initialization after boot (system date
corrections, modem registration, first LTE connection). This is handled entirely by the
daemon itself via `boot_grace_period` in `watchdog_mbim.conf` (default: 600 seconds).

The grace period uses **monotonic uptime** (`/proc/uptime`) instead of wall clock
(`date +%s`), making it immune to NTP time jumps at boot.

**Grace period is active only at boot.** The daemon checks its start uptime against
`boot_grace_period`:

- If `uptime_at_start < boot_grace_period` → daemon started near boot → grace period active
- If `uptime_at_start >= boot_grace_period` → manual or procd restart → grace period skipped, monitoring starts immediately

This means a `service watchdog_mbim restart` on a running system takes effect right away,
without waiting 10 minutes.

#### Example log at boot (uptime ~30s):

```
# always logged
init: grace period active, ends at uptime=630s (boot_grace_period=600s)

# only with debug enabled (/tmp/watchdog/mbim_debug present)
main: boot grace period active (uptime=45s < end=630s)
```

#### Example log on manual restart (uptime ~34000s):

```
# always logged
init: uptime=34641s >= boot_grace_period=600s, grace period skipped
```

#### References for the NTP/date issue at boot:
```
https://forum.openwrt.org/t/openwrt-out-of-sync-even-though-ntp-is-enabled/234493/6
https://forum.openwrt.org/t/retroactivelly-change-logs-timestamp/206872/5
https://forum.openwrt.org/t/kernel-timestamps-are-6-days-behind-on-reboot/139872
https://forum.openwrt.org/t/lte-o2-and-t-d1-work-but-vodafone-d2-does-not/43160
https://forum.openwrt.org/t/trouble-registering-private-apn-on-teltonika-rut955-with-openwrt/170960
```

---

## Required packages

In addition to those mentioned in the [OpenWrt LTE dongle guide](https://openwrt.org/docs/guide-user/network/wan/wwan/ltedongle):

```
- coreutils-timeout   Command timeout handling
- uqmi                MBIM/QMI modem management
- umbim               MBIM modem management
- jsonfilter          JSON parsing utility (normally included in standard OpenWrt installations)
```

> **Note:** `atd` is no longer required. The boot delay is now handled internally
> by the daemon via `boot_grace_period`.

> **Recommended:** rather than installing packages manually after flashing, use one of
> the following approaches to build a firmware image that already includes all required
> packages. This avoids post-flash setup steps and survives sysupgrade cleanly.
>
> - **[owut](https://openwrt.org/docs/guide-user/installation/sysupgrade.owut)** (OpenWrt
>   Upgrade Tool, pre-installed on OpenWrt 25.12+): upgrade your running system while
>   adding the required packages in one step:
>   ```sh
>   owut upgrade --add "jsonfilter usbutils coreutils-timeout uqmi luci-proto-qmi kmod-usb-net-qmi-wwan kmod-usb-serial-option picocom kmod-usb-net-cdc-mbim umbim luci-proto-mbim"
>   ```
>   On subsequent upgrades, owut remembers the installed packages automatically — no
>   need to specify them again.
>
> - **[OpenWrt Firmware Selector](https://firmware-selector.openwrt.org/)**: build a
>   custom image online by adding the required packages to the list before downloading.

If you need to install packages manually on a running system (APK, OpenWrt 25.12+):

```sh
apk update
apk add jsonfilter usbutils coreutils-timeout uqmi luci-proto-qmi kmod-usb-net-qmi-wwan \
        kmod-usb-serial-option picocom kmod-usb-net-cdc-mbim umbim luci-proto-mbim
```

---

## Preferences

Store scripts in `/root/bin/`.

Preserve `/root`, `/etc/init.d/watchdog_mbim` and `/etc/watchdog_mbim.conf`
across sysupgrade by adding them to `/etc/sysupgrade.conf`.

---

## Download & install

```sh
mkdir -p /root/bin/
cd /tmp
wget --no-hsts https://raw.githubusercontent.com/compact21/watchdog_mbim/main/watchdog_mbim_script
wget --no-hsts https://raw.githubusercontent.com/compact21/watchdog_mbim/main/watchdog_mbim
wget --no-hsts https://raw.githubusercontent.com/compact21/watchdog_mbim/main/watchdog_mbim.conf
mv watchdog_mbim /etc/init.d/watchdog_mbim
mv watchdog_mbim.conf /etc/
mv watchdog_mbim_script /root/bin/watchdog_mbim
chmod +x /etc/init.d/watchdog_mbim
chmod +x /root/bin/watchdog_mbim
echo "/root/" >> /etc/sysupgrade.conf
echo "/etc/init.d/watchdog_mbim" >> /etc/sysupgrade.conf
echo "/etc/watchdog_mbim.conf" >> /etc/sysupgrade.conf
service watchdog_mbim enable
service watchdog_mbim start
```

---

## Upgrading from a previous version

If your `/etc/rc.local` still contains the old `atd`-based scheduling block for
`watchdog_mbim`, remove it. With `service watchdog_mbim enable` the daemon is now
started automatically by procd at boot — the `rc.local` entry is redundant and causes
an unnecessary restart approximately 10 minutes after boot.

Block to remove from `rc.local`:

```sh
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
```

---

## LTE modem firmware upgrade

Before flashing new firmware onto the LTE modem module, **stop the watchdog and bring
the WAN interface down**. The daemon actively monitors the modem via `uqmi`/`umbim` and
may interfere with the flashing process or trigger a WAN restart while the modem is in
firmware upgrade mode.

```sh
service watchdog_mbim stop
ifdown wan
```

After the modem firmware upgrade is complete, restart normally:

```sh
ifup wan
service watchdog_mbim start
```

---

## Debug mode

Debug logging writes verbose output to syslog and is controlled by a flag file:

```sh
# Enable
touch /tmp/watchdog/mbim_debug

# Disable
rm -f /tmp/watchdog/mbim_debug

# Check status
service watchdog_mbim help
```

Debug is **not persistent** across reboots: the flag file lives in `/tmp` and is cleared on every boot.

To enable it automatically at every startup, uncomment the two lines in
`start_service()` in `/etc/init.d/watchdog_mbim`:

```sh
#touch "$WATCHDOG_DEBUG_FILE"
#logger -t "$LOGHEADER" "watchdog_mbim debug enabled ($WATCHDOG_DEBUG_FILE)"
```

---

## Runtime control

```sh
service watchdog_mbim start      # Start the daemon
service watchdog_mbim stop       # Stop the daemon
service watchdog_mbim restart    # Restart (grace period skipped if system already up)
service watchdog_mbim enable     # Enable autostart at boot
service watchdog_mbim disable    # Disable autostart at boot
service watchdog_mbim help       # Show status and debug info
```

### Temporary disable without stopping

To pause watchdog monitoring without stopping the process (e.g. during manual modem
maintenance):

```sh
touch /tmp/watchdog/watchdog.block    # pause
rm -f /tmp/watchdog/watchdog.block    # resume
```

### Simulated ping failure (escalation test)

To test the full recovery escalation ladder without physically disconnecting the modem,
create the force-fail flag file:

```sh
touch /tmp/watchdog/ping_once_force_fail
```

While this file is present, `ping_once()` always returns failure regardless of actual
connectivity. The daemon will progress through the escalation steps exactly as it would
during a real outage. The flag is **not** removed automatically by the daemon during
recovery — remove it manually when the test is complete:

```sh
rm -f /tmp/watchdog/ping_once_force_fail
```

> **Note:** the force-fail flag affects only `ping_once()`. The LTE session check via
> `uqmi` is not affected, so `safe_ifup` and `umbim disconnect` are still triggered
> based on actual modem state.

---

## Configuration file: `/etc/watchdog_mbim.conf`

The original config is copied to `/tmp/watchdog/mbim.conf` at startup. This allows runtime modifications without touching the original file.

Key parameters:

| Parameter | Default | Description |
|---|---|---|
| `iface` | `wan` | WAN interface name |
| `boot_grace_period` | `600` | Seconds to skip monitoring after boot (0 to disable) |
| `delay` | `20` | Seconds between loop iterations |
| `delay_reconnect` | `120` | Seconds to wait after WAN restart before resuming checks |
| `ping_targets` | `8.8.8.8 9.9.9.9 1.1.1.1` | Space-separated list of hosts for connectivity checks |
| `ping_timeout_sec` | `300` | Max seconds for background ping test before marking failure |
| `ping_verify_timeout` | `60` | Max seconds `ping_verify()` waits after forced MBIM disconnect before escalating |
| `modem_device` | `/dev/cdc-wdm0` | MBIM/QMI control device |
| `usbreset_id` | `2c7c:0512` | USB VID:PID for usbreset last-resort reset |
| `atcmd_block_file` | `/tmp/watchdog/atcmd_block` | Block file set during modem recovery |
| `ping_lock` | `/tmp/watchdog/ping_lock` | Lock directory for ping_logger |
| `watchdog_block_file` | `/tmp/watchdog/watchdog.block` | Presence pauses monitoring |
| `watchdog_predown_file` | `/tmp/watchdog/predown.sh` | Optional script run before WAN down |
| `watchdog_postup_file` | `/tmp/watchdog/postup.sh` | Optional script run after WAN up |
