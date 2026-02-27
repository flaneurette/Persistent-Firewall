# Persistent Linux Firewall

In this document we are going to build a triple-layered defense:

- a persistent firewall(initial boot)
- a systemd service (post-service restore)
- a cron canary (continuous monitoring every 5 min)
- ipset support

In this way, we do not have to rely on packages such as `UFW`, `netfilter-persistent` nor `nftables`. Our method is rather safe, because there are few surprises (no mysterious flushing of tables). Even if you are locked out, a `crontab` will restore the tables properly.

### Why?

Some services, like netfilter, tailscale, fail2ban and perhaps others, can flush or overwrite iptables rules on startup, reinstallment or reconfiguration which leads to empy iptables. Quite risky! On our system, 
Tailscale clears iptables during its initialization before it reads its own `nf=off` preference - there is no way to prevent this.

The solution is a custom systemd program that runs after boot, and makes sure that the iptables rules are restored, regardless of the programs running before it.

- No blind trust in init systems

- No faith-based persistence

- Recovery > prevention

- No mysterious flushes

```
new boot order
 ├─ netfilter-persistent (ignored, as it might flush if nftables > iptables)
 ├─ tailscaled (can flush)
 ├─ fail2ban (can flush)
 ├─ iptables-restore-onboot.service (restores)
 └─ cron canary (keeps healing)
```

However, sometimes `netfilter-persistent save` doesn't always work properly especially with `nftables` (not recommended), and might flush your tables!  (because mixed nft/legacy setups can silently flush or desync rulesets and often it is not clear which wrapper runs on netfilter!)

A lot of things can and will go wrong. So we designed the following `firewall inplementation` like how `HA systems` think:

- assume mutation

- detect drift

- reconcile continuously

### Proceed with implementation

```
sudo apt install mailutils fail2ban

# Disable nftables
sudo systemctl stop nftables
sudo systemctl disable nftables

# Switch to REAL iptables (not the nftables wrapper)
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy

# Flush nftables completely
sudo nft flush ruleset

# Reload your iptables rules
sudo iptables-restore < /etc/iptables/rules.v4

iptables --version
# Should say "legacy" now

# Disables netfilter-persistent, which can flush your iptables!
sudo systemctl disable netfilter-persistent
sudo systemctl mask netfilter-persistent
sudo apt remove netfilter-persistent iptables-persistent

# Be sure to remove these too:
sudo apt remove ubuntu-standard bpfcc-tools bpftrace bpfmon bpfcc-lua
sudo apt remove ubuntu-kernel-accessories
apt autoremove
```

Then quickly:

```
echo "flush ruleset" >> /etc/nftables.conf
sudo apt-mark hold nftables

# Finally:
apt purge nftables
```

To prevent Ubuntu from pulling it in again. (happened to me, and nftables flushed iptables yet again.)

### Edit fail2ban

Fail2ban uses nftables by default. We don't want that anymore.

```
nano /etc/fail2ban/jail.local
```

Add below `[DEFAULT]`

```
[DEFAULT]
# start using ipset also, which is much better.
banaction = iptables-ipset[type=multiport]
banaction_allports = iptables-ipset[type=allports]
```

Then:

```
systemctl restart fail2ban
```

IMPORTANT: If you have a custom `ipset` tables, you need to be very careful. You must add this at the top of in any script that restores iptables rules. 
If an ipset does not exist, and iptables loads it, iptables might fail or flush. Hence, we check:

```
ipset create YOUR_IP_SET_TABLE hash:ip -exist
```

To find your custom ipsets: `ipset list -t`

### Custom firewall script

Now start using regular `iptables` again.

Create:

`sudo nano /usr/local/bin/firewall`

Paste:

```
#!/bin/bash
set -e

# Check ipset table. Only uncomment if you use custom ipsets.
# ipset create YOUR_IP_SET_TABLE hash:ip -exist 2>/dev/null || true

# Only uncomment if you use custom ipsets or use fail2ban:
# ipset save > /etc/iptables/ipsets.conf

iptables-save > /etc/iptables/rules.v4.bak.firewall
ip6tables-save > /etc/iptables/rules.v6.bak.firewall

iptables-save > /etc/iptables/rules.v4
ip6tables-save > /etc/iptables/rules.v6

echo "Saved all iptables!"
```

Then:

`sudo chmod +x /usr/local/bin/firewall`

Use It:

```
# When you change rules, save with:
sudo firewall
```

## Saving rules

Whenever you change your iptables rules, save them:

`sudo firewall`

This writes your current rules to `/etc/iptables/rules.v4` and `/etc/iptables/rules.v6`.

## The restore script

Example for `tailscale`. Replace your `service` if you want to check another one.

`nano /usr/local/sbin/iptables-restore-onboot.sh`

Edit ALERT_EMAIL and then paste:

```bash
#!/bin/bash

ALERT_EMAIL="your@email.com"
HOSTNAME=$(hostname)
ERRORS=""

# Do you use ipsets or fail2ban? 1 for true, 0 for false.
# You MUST be certain ipsets exists. Otherwise: failure.

IPSET_USE=0

# Wait for tailscaled to start, timeout after 75 seconds
for i in $(seq 1 15); do
    if systemctl is-active --quiet tailscaled; then
        sleep 5  # Give it a moment to finish flushing
        break
    fi
    sleep 5
done

if [ "$IPSET_USE" == 1 ]; then
    # Restore ipsets first - iptables rules may depend on these sets existing
    if [ -f /etc/iptables/ipsets.conf ]; then
        ipset restore -exist < /etc/iptables/ipsets.conf 2>&1
        if [ $? -ne 0 ]; then
            ERRORS="${ERRORS}\n[FAILED] ipset restore from /etc/iptables/ipsets.conf"
        fi
    else
        ERRORS="${ERRORS}\n[WARNING] /etc/iptables/ipsets.conf not found - ipsets not restored"
    fi
fi

# Restore iptables rules
iptables-restore < /etc/iptables/rules.v4 2>&1
if [ $? -ne 0 ]; then
    ERRORS="${ERRORS}\n[FAILED] iptables-restore from /etc/iptables/rules.v4"
fi

# Restore ip6tables rules
ip6tables-restore < /etc/iptables/rules.v6 2>&1
if [ $? -ne 0 ]; then
    ERRORS="${ERRORS}\n[FAILED] ip6tables-restore from /etc/iptables/rules.v6"
fi

# Send alert email if any errors occurred
if [ -n "$ERRORS" ]; then
    echo -e "Subject: [ALERT] ${HOSTNAME} - iptables restore failed on boot\n\nThe following errors occurred during iptables restore on ${HOSTNAME}:\n${ERRORS}\n\nPlease check your firewall rules immediately." \
        | sendmail "$ALERT_EMAIL"
fi

# Check if fail2ban is running and restart it to recreate its chains
if systemctl is-active --quiet fail2ban; then
    systemctl restart fail2ban
    if [ $? -ne 0 ]; then
        echo -e "Subject: [ALERT] ${HOSTNAME} - fail2ban restart failed after iptables restore\n\nfail2ban failed to restart on ${HOSTNAME} after iptables restore." \
            | sendmail "$ALERT_EMAIL"
    fi
fi

# Post-fail2ban ipset saving.
if [ "$IPSET_USE" == 1 ]; then
    ipset save > /etc/iptables/ipsets.conf
fi
```

> NOTE: $(seq 1 60); = 5 minutes. Shorter: 15 instead of 60. ~= 75 seconds. Still, tailscale can be very slow. 5 minutes is extra safety.

`chmod +x /usr/local/sbin/iptables-restore-onboot.sh`

## The systemd service

`nano /etc/systemd/system/iptables-restore-onboot.service`

```ini
[Unit]
Description=Restore iptables rules after boot
After=network-online.target tailscaled.service
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/iptables-restore-onboot.sh
TimeoutStartSec=120
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

The `After=network-online.target tailscaled.service` is what guarantees this runs last.

Drawback is, it might wait for 30-60 seconds for full reboot. You might have to tweak the TimeoutStartSec, to see whether tailscale boots fast or not. Tailscale can be slow to boot. So 60-120 seconds is safe.

## Enable it

```bash
sudo systemctl daemon-reload
sudo systemctl enable iptables-restore-onboot.service
sudo systemctl start iptables-restore-onboot.service (might be slow!)
```


# Self-healing crontab

A self-healing crontab is very useful, this gives an extra layer of protection. Although rare, systemd can fail. You might be locked out, or something else causes an issue, like kernel ordering of executing is mixed up or deferred.

To mitigate this we can create a crontab that runs every 5 minutes, regardless of what is going on, and heals the firewall when it detects it was flushed.

🐤 Run this canary rule first:

```
sudo iptables -I INPUT 2 -s 203.0.113.99 -m comment --comment "CANARY-ADMIN" -j DROP
```

The above adds a “dummy rule” as a canary to check whether your iptables have been wiped or not.

> Note: 203.0.113.0/24 and 2001:db8::/32 are TEST-NET ranges - they're reserved and will never be routed on the internet, so they're perfect for canaries.

```
sudo iptables-save > /etc/iptables/rules.v4
sudo ip6tables-save > /etc/iptables/rules.v6
```

Then:

`nano /usr/local/sbin/check-iptables.sh`

Paste:

```
#!/bin/bash
# Check if the dummy rule exists + check if systemd service is running.

LOG="/var/log/iptables-check.log"
EMAIL="info@example.com"
FROM="root@example.com"
SERVICE="iptables-restore-onboot.service"
CANARY_IP="203.0.113.99"
CANARY_COMMENT="CANARY-ADMIN"

# Do you use ipsets or fail2ban? 1 for true, 0 for false.
# You MUST be certain ipsets exists. Otherwise: failure.

IPSET_USE=0

touch "$LOG"
chmod 600 "$LOG"

restore_needed=0

# Check for top canary (catches early flush)
if ! iptables -C INPUT -s "$CANARY_IP" -m comment --comment "$CANARY_COMMENT" -j DROP &>/dev/null; then
    echo "$(date): IPv4 top canary missing" >> "$LOG"
    restore_needed=1
fi

if [ $restore_needed -eq 1 ]; then
    echo "$(date): Canary missing - restoring iptables..." >> "$LOG"
   
    if [ "$IPSET_USE" == 1 ]; then
        ipset restore < /etc/iptables/ipsets.conf 2>&1
    fi
    
    [ -f /etc/iptables/rules.v4 ] && iptables-restore < /etc/iptables/rules.v4
    [ -f /etc/iptables/rules.v6 ] && ip6tables-restore < /etc/iptables/rules.v6
    
    echo "$(date): Rules restored successfully" >> "$LOG"
    
    # Check if fail2ban is running and restart it to recreate its chains
    if systemctl is-active --quiet fail2ban; then
       systemctl restart fail2ban
       if ! iptables -C INPUT -s "$CANARY_IP" -m comment --comment "$CANARY_COMMENT" -j DROP &>/dev/null; then
           echo "$(date): Fail2ban flushed the iptables?!" >> "$LOG"
           mail -s "Fail2ban flushed the iptables?!" -r "$FROM" "$EMAIL"
       fi
    fi
    
    # Post-fail2ban ipset saving.
    if [ "$IPSET_USE" == 1 ]; then
        ipset save > /etc/iptables/ipsets.conf
    fi
    
fi

# Check if the service is enabled on boot
if systemctl is-enabled --quiet "$SERVICE"; then
    echo "$(date) - $SERVICE is enabled" >> "$LOG"
else
    echo "$(date) - $SERVICE is NOT enabled, enabling" >> "$LOG"
    systemctl enable "$SERVICE"
fi  

# Check if the service is running
if systemctl is-active --quiet "$SERVICE"; then
    echo "$(date) - $SERVICE is running" >> "$LOG"
else
    echo "$(date) - $SERVICE is NOT running, starting it." >> "$LOG"
    systemctl start "$SERVICE"
fi

```

Then:

`sudo chmod +x /usr/local/sbin/check-iptables.sh`

Then:

`sudo crontab -e`

Then add:

`*/5 * * * * /usr/local/sbin/check-iptables.sh`

---

### Quick Verification

After setup, verify each layer:

### Check saved rules exist
`ls -lh /etc/iptables/rules.v4`

### Check systemd service
`sudo systemctl status iptables-restore-onboot.service`

### Check cron is scheduled
`sudo crontab -l | grep check-iptables`

### Force a test restore
`sudo /usr/local/sbin/check-iptables.sh`

Test it on boot:

```
sudo reboot
```

Rebooting may take some time, please be patient.

Then check:

```
iptables -L -n -v --line-numbers
```

Run this a few times after one another, to see `systemd` or `cron` effect.

---

End.

Additional useful IP reassignment script:

https://github.com/flaneurette/Server-Scripts/blob/main/Scripts/reassign-ip.sh