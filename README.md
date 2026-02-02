# Ubuntu iptables + GeoIP Blocking (HTTP/HTTPS) – Step-by-Step Considered Guide
This guide shows how to enable iptables, create GeoIP-based access control for HTTP/HTTPS using ipset, and lock down SSH safely.

### 1. Install Required Packages
```shell
apt update
apt install -y iptables ipset netfilter-persistent wget
```
### Verify:
iptables --version
ipset --version

### 2. Create GeoIP Directory
```shell
mkdir -p /usr/local/geoip
cd /usr/local/geoip
```
### 3. Download Country IP List (CIDR Format)
    Using ipdeny.com (free & reliable) Example: Sri Lanka (LK)
    
```shell
   wget https://www.ipdeny.com/ipblocks/data/countries/lk.zone
```
Verify:
```shell
wc -l lk.zone
head lk.zone
```
### 4. Create ipset for Geo Blocking
```shell
ipset create allowed_https hash:net
```
```shell
while read ip; do
    ipset add allowed_https $ip
done < lk.zone
```
Verify:
```shell
ipset list allowed_https | grep "Number of entries"
```
### 5. Flush Existing iptables Rules (Safe Start)
5.1 Loopback

```shell
    iptables -I INPUT 1 -i lo -j ACCEPT
```
5.2 Established Connections

```shell
    iptables -I INPUT 1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

### 6. Lock Down SSH (Allow Only Trusted IPs)

```shell
iptables -I INPUT 1 -p tcp --dport 22 -s x.x.x.x -j ACCEPT
iptables -I INPUT 1 -p tcp --dport 22 -s x.x.x.x -j ACCEPT
```
6.1 (Optional) Drop all other SSH:

```shell
    iptables -A INPUT -p tcp --dport 22 -j DROP
```
### 7. Allow HTTP/HTTPS Only From GeoIP (Country)

```shell
iptables -A INPUT -p tcp --dport 80  -m set --match-set allowed_https src -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -m set --match-set allowed_https src -j ACCEPT
```
7.1 Drop others:

```shell
iptables -A INPUT -p tcp --dport 80  -j DROP
iptables -A INPUT -p tcp --dport 443 -j DROP
```
### 8. Set Default Policy to DROP (Final Lock)

```shell
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
```
### 9. Verify Firewall Rules
```shell
iptables -L INPUT -n --line-numbers
```
Expected:

* SSH allowed only from trusted IPs
* HTTP/HTTPS allowed only from GeoIP
* Default INPUT = DROP

### 10. Save Rules (Persistence After Reboot)
```shell
iptables-save > /etc/iptables/rules.v4
ipset save > /etc/ipset.conf
systemctl enable netfilter-persistent
```
Test reload:

```shell
systemctl restart netfilter-persistent
```
### 11. Auto-Update GeoIP List (Recommended)
Create script

```shell
nano /usr/local/bin/update-geoip.sh
```
Paste:

```shell
#!/bin/bash
cd /usr/local/geoip || exit 1

wget -q -O lk.zone https://www.ipdeny.com/ipblocks/data/countries/lk.zone

ipset flush allowed_https
while read ip; do
    ipset add allowed_https $ip
done < lk.zone
```
Make executable:

```shell
chmod +x /usr/local/bin/update-geoip.sh
```
Cron job (weekly)

```shell
crontab -e
```
Add:

```shell
0 3 * * 0 /usr/local/bin/update-geoip.sh
```
### 12. Rollback (Emergency)
```shell
iptables -P INPUT ACCEPT
iptables -F
```


