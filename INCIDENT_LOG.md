# Incident Log 

## Group – Dune

| Name | GitHub Username | Student ID |
|---|---|---|
| Swam Pyae Paing | [@LoneKit](https://github.com/LoneKit) | 6705142028 |
| Thet Htoo Aung | [@thethtoo724-design](https://github.com/thethtoo724-design) | 6705142042 |
| Minn Khant Si Thu | [@minnkhantsithu](https://github.com/minnkhantsithu) | 6705142014 |
| Soe Moe Hein | [@6705142013-SoeMoe](https://github.com/6705142013-SoeMoe) | 6705142013 |

- **Company / domain you were assigned:** novapeak.lab
- **Group id used with the checker:** lab01

Baseline before any fix: checker `TOTAL: 11.0 / 25`. Final: `TOTAL: 25.0 / 25`.

---

## Fault 1: Forward zone points at a file that does not exist

* **Symptom (what we observed):**

The lab checker showed that named-checkconf -z failed and the DNS records for ns1 and www could not be found.
```
$ python3 /root/grading/check_lab.py --group lab01 --phase build
  [   1/2 ] C4   named-checkconf -z FAILED; named-checkzone clean
  [ 0.0/3 ] C2   ns1 expected 192.168.19.10 got nothing; www expected 192.168.19.20 got nothing; ...
```

* **Hypothesis (what we suspected and why):**

  We suspected that the forward zone configuration was pointing to the wrong file. Since named-checkzone was clean, the zone file itself appeared to be valid. However, named-checkconf -z failed, suggesting that BIND could not load the zone because of a configuration problem.

* **Proof (the command that confirmed the cause):**

```
$ named-checkconf -z
zone novapeak.lab/IN: loading from master file /etc/bind/db.MISSING failed: file not found
zone novapeak.lab/IN: not loaded due to errors.
_default/novapeak.lab/IN: file not found
zone 19.168.192.in-addr.arpa/IN: loaded serial 2024010101

$ ls -l /etc/bind/
-rw-r--r-- 1 bind bind  212 Oct 13 20:04 db.novapeak.lab
-rw-r--r-- 1 bind bind  233 Oct 13 20:04 db.novapeak.lab.rev

$ grep -n file /etc/bind/named.conf.local
3:      file "/etc/bind/db.MISSING";
8:      file "/etc/bind/db.novapeak.lab.rev";
```

  These results confirmed that the forward zone pointed to /etc/bind/db.MISSING, which did not exist. The correct file was /etc/bind/db.novapeak.lab.

* **Fix (what we changed):**

  We corrected the file path in /etc/bind/named.conf.local using this command:
  `sed -i 's#/etc/bind/db.MISSING#/etc/bind/db.novapeak.lab#' /etc/bind/named.conf.local`, Then, we restarted the BIND service:
   `systemctl restart named`.

* **Confirmation (output showing it now works):**

```
$ named-checkconf -z; echo "exit code: $?"
zone novapeak.lab/IN: loaded serial 2024010101
zone 19.168.192.in-addr.arpa/IN: loaded serial 2024010101
...
exit code: 0

  [   2/2 ] C4   named-checkconf -z clean; named-checkzone clean
  TOTAL: 12.0 / 25
```

* **Found/proved by:** Thet Htoo Aung/ Running named-checkconf -z, checking the available zone files, and inspecting /etc/bind/named.conf.local.

---

## Fault 2: Firewall rule silently drops DNS (UDP/53)

* **Symptom :**

```
$ python3 /root/grading/check_lab.py --group lab01 --phase build
  [   0/2 ] C1   client cannot reach named at 192.168.19.10 (timeout/no status). Check listen-on, allow-query, firewall, ...

$ ip netns exec client dig @192.168.19.10 novapeak.lab SOA +time=3 +tries=1
;; communications error to 192.168.19.10#53: timed out

$ dig @127.0.0.1 novapeak.lab SOA +short
;; communications error to 127.0.0.1#53: timed out
```

Hypothesis : The DNS queries timed out without receiving a response. Since loopback queries also timed out, allow-query was unlikely to be the cause. We suspected that a firewall rule was dropping UDP/53 packets. The ss command helped rule out the listen-on setting.

* **Proof :**

```
$ ss -lunp | grep ':53'
UNCONN 0 0   192.168.19.10:53   0.0.0.0:*   users:(("named",pid=3037,fd=25))
UNCONN 0 0   127.0.0.1:53       0.0.0.0:*   users:(("named",pid=3037,fd=18))
...

$ iptables -L INPUT -n -v --line-numbers
Chain INPUT (policy ACCEPT 1427 packets, 141K bytes)
num   pkts bytes target     prot opt in     out     source               destination
1       16  1358 DROP       17   --  *      *       0.0.0.0/0            0.0.0.0/0            udp dpt:53
```

named was listening on both the LAN address and loopback, so the listen-on setting was correct. However, INPUT rule 1 was dropping UDP/53 packets and had already blocked 16 packets from our test queries.

```
$ dig @192.168.19.10 novapeak.lab SOA +tcp +short +time=3 +tries=1
ns1.novapeak.lab. admin.novapeak.lab. 2024010101 3600 1800 604800 3600
```

The same query worked over TCP but timed out over UDP. The packet counter remained unchanged, confirming that the firewall rule specifically blocked UDP/53 traffic.

Fix : We removed only the rule blocking UDP port 53 using iptables -D INPUT -p udp --dport 53 -j DROP.

Confirmation :

```
$ iptables -L INPUT -n -v --line-numbers
Chain INPUT (policy ACCEPT 1500 packets, 146K bytes)
num   pkts bytes target     prot opt in     out     source               destination

$ ip netns exec client dig @192.168.19.10 novapeak.lab SOA +short +time=3 +tries=1
ns1.novapeak.lab. admin.novapeak.lab. 2024010101 3600 1800 604800 3600

  [   2/2 ] C1   named is authoritative for novapeak.lab and reachable from the client.
  [   2/2 ] C3   reverse lookup of 192.168.19.10 returns ns1.
  TOTAL: 18.25 / 25
```

* **Found/proved by:** MINN KHANT SI THU

---

## Fault 3: `api` A record points at the gateway instead of 192.168.19.40

* **Symptom :**

```
  [2.25/3 ] C2   api expected 192.168.19.40 got ['192.168.19.1']

$ ip netns exec client dig @192.168.19.10 api.novapeak.lab A +short
192.168.19.1
```

* **Hypothesis :**
  The DNS lookup returned 192.168.19.1, which is the DHCP gateway, instead of the expected IP address 192.168.19.40. This suggested that the A record for api.novapeak.lab was incorrectly configured. To investigate further, we compared the forward and reverse DNS zones to identify any inconsistencies between the hostname and its assigned IP address.

* **Proof :**

```
$ grep -n api /etc/bind/db.novapeak.lab
7:api IN A 192.168.19.1

$ grep -n -E '^(10|20|30|40) ' /etc/bind/db.novapeak.lab.rev
4:10 IN PTR ns1.novapeak.lab.
5:20 IN PTR www.novapeak.lab.
6:30 IN PTR mail.novapeak.lab.
7:40 IN PTR api.novapeak.lab.
```

  The reverse DNS zone associated IP address 192.168.19.40 with api.novapeak.lab, while the forward DNS zone incorrectly mapped the hostname to 192.168.19.1. This inconsistency confirmed that the forward A record was misconfigured. The scenario configuration, verified using python3 /root/grading/params.py lab01, also confirmed that the correct IP address for api was 192.168.19.40.

* **Fix :**
   We corrected the A record in /etc/bind/db.novapeak.lab by changing the IP address for api from 192.168.19.1 to 192.168.19.40. We then incremented the SOA serial number from 2024010101 to 2024010102 to reflect the update and reloaded the DNS zone using rndc reload novapeak.lab to apply the changes.

* **Confirmation :**

```
$ named-checkzone novapeak.lab /etc/bind/db.novapeak.lab
zone novapeak.lab/IN: loaded serial 2024010102
OK

$ ip netns exec client dig @192.168.19.10 api.novapeak.lab A +short
192.168.19.40

  [ 3.0/3 ] C2   all four A records correct
  TOTAL: 19.0 / 25
```

* **Found/proved by:** Swam Pyae Paing

---

## Fault 4: DHCP hands out a public resolver (8.8.8.8) instead of the company DNS

**Symptom (what we saw):**

The client got a DHCP lease just fine, but DHCP was advertising the public DNS `8.8.8.8` instead of the company DNS `192.168.19.10`. So the client couldn't resolve `www.novapeak.lab` to the expected IP `192.168.19.20`.

```
[   3/3 ] C6   client obtained a lease: 192.168.19.104
[   0/3 ] C9   DHCP advertises 8.8.8.8, expected the company DNS 192.168.19.10
[   0/3 ] C11  using the leased resolver 8.8.8.8, www did not resolve to 192.168.19.20
```

**Hypothesis (what we thought and why):**

We figured the DHCP server was handing out the wrong DNS server to clients. The lease process itself was working—client got an IP no problem. But it was pointed at `8.8.8.8`, which can't resolve the internal domain `novapeak.lab`. That pointed to the `option domain-name-servers` line in `/etc/dhcp/dhcpd.conf` being wrong. Even though the config passed the syntax check (C5), we still had to look at the lease from the client's side to catch it.

**Proof (command that confirmed it):**

```
$ ip netns exec client dhcpcd -U cli0 | grep -E 'ip_address|routers|domain_name'
ip_address=192.168.19.104
routers=192.168.19.1
domain_name_servers=8.8.8.8
domain_name=novapeak.lab

$ grep -n -E 'domain-name|router|range' /etc/dhcp/dhcpd.conf
1:option domain-name "novapeak.lab";
6:      range 192.168.19.104 192.168.19.144;
7:      option routers 192.168.19.1;
8:      option domain-name-servers 8.8.8.8;

$ ip netns exec client dig @8.8.8.8 www.novapeak.lab +time=2 +tries=1
;; communications error to 8.8.8.8#53: timed out
```

That confirmed it. DHCP was advertising `8.8.8.8` instead of the internal DNS `192.168.19.10`. The DNS query to the wrong resolver timed out, which is why the client couldn't resolve `www.novapeak.lab`.

**Fix (what we changed):**

We fixed the DNS server option in `/etc/dhcp/dhcpd.conf` by changing this line:

`option domain-name-servers 8.8.8.8;`

to:

`option domain-name-servers 192.168.19.10;`

Then we checked the config and restarted the DHCP service:

`dhcpd -t -cf /etc/dhcp/dhcpd.conf`

`systemctl restart isc-dhcp-server`

**Confirmation (output showing it works now):**

```
$ dhcpd -t -cf /etc/dhcp/dhcpd.conf; echo "exit code: $?"
exit code: 0

$ systemctl is-active isc-dhcp-server
active

[   3/3 ] C9   DHCP advertises the company DNS (192.168.19.10)
[   3/3 ] C11  client resolves www via the leased resolver
TOTAL: 25.0 / 25
```

Config was valid, service was running, and the lab checker confirmed DHCP was handing out the right DNS and the client could resolve `www.novapeak.lab`.

**Found/proved by:** [Soe Moe Hien] / Ran `dhcpcd -U cli0` to check the client's lease, looked at `/etc/dhcp/dhcpd.conf` for the advertised DNS, and used `dig` to test resolution.



## Lessons learned (optional)

* A service reporting "active (running)" does not mean the zone it should serve is loaded.
* A timeout, even on loopback, points to a firewall drop rather than allow-query (which would return REFUSED).
* Forward and reverse zones are independent checks on each other.
* The DHCP integration fault was only visible from the client's side, not from a syntax check.
