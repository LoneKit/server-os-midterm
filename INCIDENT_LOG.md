# Incident Log Test

Group: Dune
Members: TODO (name + GitHub username, one per line)
Company / domain: novapeak.lab

Baseline before any fix: checker `TOTAL: 11.0 / 25`. Final: `TOTAL: 25.0 / 25`.

---

## Fault 1: Forward zone points at a file that does not exist

* **Symptom (what we observed):**

```
$ python3 /root/grading/check_lab.py --group lab01 --phase build
  [   1/2 ] C4   named-checkconf -z FAILED; named-checkzone clean
  [ 0.0/3 ] C2   ns1 expected 192.168.19.10 got nothing; www expected 192.168.19.20 got nothing; ...
```

* **Hypothesis (what we suspected and why):**
  DRAFT, rewrite in your own words: `named-checkzone` was clean, so the zone file itself is valid. `named-checkconf -z` fails, so the problem is more likely in how named.conf refers to the zone.

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

  Note: named was `active (running)` the whole time, but the company zone was not loaded. The `FORMERR resolving './NS/IN'` / `resolver priming query complete: failure` lines in the journal are named failing to reach the internet root servers (no outbound access in the playground) and are unrelated to this lab.

* **Fix (what we changed):**
  In `/etc/bind/named.conf.local`, changed the forward zone's `file` from `/etc/bind/db.MISSING` to `/etc/bind/db.novapeak.lab`:
  `sed -i 's#/etc/bind/db.MISSING#/etc/bind/db.novapeak.lab#' /etc/bind/named.conf.local`, then `systemctl restart named`.

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

* **Found/proved by:** TODO

---

## Fault 2: Firewall rule silently drops DNS (UDP/53)

* **Symptom (what we observed):**

```
$ python3 /root/grading/check_lab.py --group lab01 --phase build
  [   0/2 ] C1   client cannot reach named at 192.168.19.10 (timeout/no status). Check listen-on, allow-query, firewall, ...

$ ip netns exec client dig @192.168.19.10 novapeak.lab SOA +time=3 +tries=1
;; communications error to 192.168.19.10#53: timed out

$ dig @127.0.0.1 novapeak.lab SOA +short
;; communications error to 127.0.0.1#53: timed out
```

* **Hypothesis (what we suspected and why):**
  DRAFT, rewrite in your own words: a timeout (not REFUSED or SERVFAIL) means the query got no answer at all. Even loopback timed out, so allow-query would not explain it, because it would give REFUSED. We suspected a firewall rule dropping packets before named sees them. `ss` was used to rule out `listen-on`.

* **Proof (the command that confirmed the cause):**

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

  named is listening on lan0's address and on loopback (so `listen-on` is fine), but INPUT rule 1 drops UDP/53 and had already dropped 16 packets (our test queries).

```
$ dig @192.168.19.10 novapeak.lab SOA +tcp +short +time=3 +tries=1
ns1.novapeak.lab. admin.novapeak.lab. 2024010101 3600 1800 604800 3600
```

  The same query over TCP gets an answer while UDP times out, and the packet counter did not move (still 16 / 1358), so the rule is specific to UDP/53.

* **Fix (what we changed):**
  Removed only that rule: `iptables -D INPUT -p udp --dport 53 -j DROP`.

* **Confirmation (output showing it now works):**

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

* **Found/proved by:** TODO

---

## Fault 3: `api` A record points at the gateway instead of 192.168.19.40

* **Symptom (what we observed):**

```
  [2.25/3 ] C2   api expected 192.168.19.40 got ['192.168.19.1']

$ ip netns exec client dig @192.168.19.10 api.novapeak.lab A +short
192.168.19.1
```

* **Hypothesis (what we suspected and why):**
  DRAFT, rewrite in your own words: 192.168.19.1 is the DHCP gateway, so a host called `api` pointing at the router looks wrong. The reverse zone is a second source of truth, so we compared it with the forward zone.

* **Proof (the command that confirmed the cause):**

```
$ grep -n api /etc/bind/db.novapeak.lab
7:api IN A 192.168.19.1

$ grep -n -E '^(10|20|30|40) ' /etc/bind/db.novapeak.lab.rev
4:10 IN PTR ns1.novapeak.lab.
5:20 IN PTR www.novapeak.lab.
6:30 IN PTR mail.novapeak.lab.
7:40 IN PTR api.novapeak.lab.
```

  The reverse zone maps .40 to api, but the forward zone maps api to .1, so the two disagree. The scenario card (`python3 /root/grading/params.py lab01`) also says api is 192.168.19.40.

* **Fix (what we changed):**
  In `/etc/bind/db.novapeak.lab`, changed `api IN A 192.168.19.1` to `192.168.19.40`, bumped the SOA serial from 2024010101 to 2024010102, then reloaded the zone with `rndc reload novapeak.lab`.

* **Confirmation (output showing it now works):**

```
$ named-checkzone novapeak.lab /etc/bind/db.novapeak.lab
zone novapeak.lab/IN: loaded serial 2024010102
OK

$ ip netns exec client dig @192.168.19.10 api.novapeak.lab A +short
192.168.19.40

  [ 3.0/3 ] C2   all four A records correct
  TOTAL: 19.0 / 25
```

* **Found/proved by:** TODO

---

## Fault 4: DHCP hands out a public resolver (8.8.8.8) instead of the company DNS

* **Symptom (what we observed):**

```
  [   3/3 ] C6   client obtained a lease: 192.168.19.104
  [   0/3 ] C9   DHCP advertises 8.8.8.8, expected the company DNS 192.168.19.10 -- ...
  [   0/3 ] C11  using the leased resolver 8.8.8.8, www did not resolve to 192.168.19.20 (got nothing). ...
```

* **Hypothesis (what we suspected and why):**
  DRAFT, rewrite in your own words: the lease itself works, but a client using only what DHCP tells it would ask 8.8.8.8 about novapeak.lab, which cannot answer. So `domain-name-servers` in dhcpd.conf is probably wrong. The server config was syntactically valid (C5 clean), so only the client's view could reveal it.

* **Proof (the command that confirmed the cause):**

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

* **Fix (what we changed):**
  In `/etc/dhcp/dhcpd.conf` line 8, changed `option domain-name-servers 8.8.8.8;` to `192.168.19.10;`, validated with `dhcpd -t`, and restarted with `systemctl restart isc-dhcp-server`.

* **Confirmation (output showing it now works):**

```
$ dhcpd -t -cf /etc/dhcp/dhcpd.conf; echo "exit code: $?"
exit code: 0
$ systemctl is-active isc-dhcp-server
active

  [   3/3 ] C9   DHCP advertises the company DNS (192.168.19.10)
  [   3/3 ] C11  client resolves www via the leased resolver
  TOTAL: 25.0 / 25
```

* **Found/proved by:** TODO

---

## Lessons learned (optional)

* A service reporting "active (running)" does not mean the zone it should serve is loaded.
* A timeout, even on loopback, points to a firewall drop rather than allow-query (which would return REFUSED).
* Forward and reverse zones are independent checks on each other.
* The DHCP integration fault was only visible from the client's side, not from a syntax check.
