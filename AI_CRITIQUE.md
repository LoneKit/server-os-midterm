# AI Usage and Critique

Group: TODO (group name / number)
Members: TODO (name + GitHub username, one per line)

---

## Part 3a: Declaration (required)

| AI tool | What we used it for |
| --- | --- |
| Claude (claude.ai), main conversation | We uploaded the midterm project zip and asked it to review it. In its first reply it listed the four planted faults and several design issues in the checker and handout, so it saw the "answer key" before we diagnosed anything. It then coached us step by step through the lab: it told us which commands to run, interpreted the output we pasted back, and suggested each fix. It also helped with the Git/PowerShell steps to submit, drafted INCIDENT_LOG.md from our real session output (including the draft hypothesis lines we then rewrote), and drafted the text of this AI_CRITIQUE.md, which we then read, checked against our session.log and edited. We ran every command ourselves in the playground; Claude never had access to our system. |
| Claude (claude.ai), new fresh conversation | Part 3c test only. We pasted the symptom-only prompt below into a brand-new conversation with no earlier context. |
| ChatGPT | Part 3c test only. Same symptom-only prompt, in a new chat. |

No other AI tools were used.

---

## Part 3b: Confirm you diagnosed first

Honest statement: our diagnosis was **not fully independent**. In the main Claude conversation, the AI named all four planted faults in its very first review of the project files, before we had run anything. After that, we did the work ourselves, step by step, but under its guidance:

* We ran the checker and read each symptom (baseline `TOTAL: 11.0 / 25`).
* We ran the diagnostic commands ourselves and pasted the real output: `named-checkconf -z`, `ls /etc/bind`, `iptables -L INPUT -n -v --line-numbers`, `dig` from the client namespace, `dhcpcd -U` on the client, and `grep` on the config files.
* We applied each fix ourselves and re-ran the checker after each one until it reached `TOTAL: 25.0 / 25`.

The four faults we found and proved, in our words:

1. The forward zone in `named.conf.local` pointed at `/etc/bind/db.MISSING`, so the zone never loaded.
2. A firewall rule (`iptables ... DROP udp dpt:53`) silently dropped DNS over UDP, so even `dig @127.0.0.1` timed out.
3. The `api` A record pointed at the gateway (192.168.19.1) instead of 192.168.19.40; the reverse zone already said .40.
4. DHCP advertised `8.8.8.8` as the DNS server instead of 192.168.19.10.

Our record of the diagnostic path is `INCIDENT_LOG.md` and `session.log`. Because the AI told us what to look for, the comparison in 3c below was done with fresh conversations that saw only the symptoms.

---

## Part 3c: Putting the AIs to the test

We gave the following to two AI tools, in fresh conversations, once each, with no config files and no hints:

> I run a BIND9 + isc-dhcp-server lab for the company novapeak.lab on Ubuntu (server 192.168.19.10, DHCP clients get leases from it). A grader runs from a client network namespace and reports: C4 named-checkconf -z FAILED; C1 client cannot reach named at 192.168.19.10 (timeout); C2/C3 got nothing for ns1/www/mail/api and the reverse lookup; C9 DHCP advertises 8.8.8.8, expected the company DNS 192.168.19.10; C11 www did not resolve via the leased resolver. `dig @127.0.0.1` also times out on the server. Diagnose the root causes and give me the exact fixes. You cannot see my system, so tell me which commands to run to confirm each cause before changing anything.

| Fault (in our words) | We found it? | AI #1 (Claude, fresh chat) | AI #2 (ChatGPT) | Was each AI actually right? |
| --- | --- | --- | --- | --- |
| Zone `file` path points at `/etc/bind/db.MISSING`, so the zone never loads | Yes, with `named-checkconf -z` ("file not found") | Told us to run `named-checkconf -z` and listed "file not found / wrong path in zone stanza" as one of six possible causes | Told us to run `named-checkconf -z` and listed "wrong file path" among six possible causes, and said to use the reported filename and line number | **Partly right.** Both pointed at the right command and listed the right cause, but neither committed to it; it was one item in a menu. Verified: our `named-checkconf -z` output named `db.MISSING` directly. |
| Firewall `DROP udp dpt:53` in INPUT | Yes, with `iptables -L INPUT -n -v --line-numbers` | **Right cause, partly wrong fix.** Reasoned that a *timeout* on loopback (rather than "connection refused") means a DROP rule, and called the firewall its "top suspect". But the fix inserted ACCEPT rules above the bad rule instead of removing it, and also pushed `ufw` and rewriting `listen-on`/`allow-query` | **Missed in practice.** Told us to look at the firewall only after `dig @127.0.0.1` already worked, and blamed the loopback timeout on BIND being down. Its firewall checks were `ufw status` and `nft list ruleset`, never `iptables -L`, and its fix was `ufw allow` | Claude: right diagnosis, but its fix would have left the DROP rule in place. ChatGPT: wrong order of investigation for this fault. Verified: in our `session.log` the loopback `dig` timed out while `iptables -L` showed the rule with 16 dropped packets, and `dig +tcp` still answered, so the cause was the firewall, not named. We did not test whether `nft list ruleset` would have displayed this rule. |
| `api` A record is 192.168.19.1 instead of 192.168.19.40 | Yes, by comparing forward and reverse zones | Did not mention it | Did not mention it. It told us to `dig` each name and compare to "the IP addresses specified in your forward zone", which is circular because the wrong value *is* in the forward zone | **Both missed it.** Verified: before the fix the checker said `C4 ... named-checkzone clean` and, separately, `C2 ... api expected 192.168.19.40 got ['192.168.19.1']`. A zone with a wrong IP is still syntactically valid, so syntax checks cannot catch it; it took comparing two data sources (forward vs. PTR). |
| DHCP advertises `8.8.8.8` instead of `192.168.19.10` | Yes, from the client's lease (`dhcpcd -U`) and `dhcpd.conf` | Pointed to `domain-name-servers` in `dhcpd.conf`, gave the right grep and fix, and noted scope overrides | Same diagnosis and fix, with good detail on scope overrides and lease renewal | **Both right.** Verified: our lease showed `domain_name_servers=8.8.8.8` and `dhcpd.conf` line 8 matched. Minor mismatch: both told us to renew with `dhclient`, but our lab client uses `dhcpcd`. |

For each AI suggestion that was WRONG or misleading, how we verified it was wrong:

* **"Check/change `listen-on` and `allow-query` in `named.conf.options`" (both AIs).** Wrong for this lab. Verified: `ss -lunp | grep :53` in `session.log` shows named already listening on `192.168.19.10:53` and `127.0.0.1:53`, and our submitted `named.conf.options` still has the original `listen-on { any; }; allow-query { any; };`. We never changed that file and the checker still reached 25/25.
* **"Add ACCEPT rules for port 53" (Claude).** Misleading: it would make DNS work while leaving the real fault (the DROP rule) in place. Verified: we removed only that rule with `iptables -D INPUT -p udp --dport 53 -j DROP`, and `iptables -L INPUT` then showed an empty chain and the client `dig` answered.
* **"Only investigate the firewall after loopback works" (ChatGPT).** Wrong ordering. Verified: loopback only started working after the firewall rule was removed, as `session.log` shows (timeouts first, then a SOA answer).
* **`ufw` commands (both).** We did not run `ufw status` on the playground, so we cannot say whether it was active; our firewall problem was an iptables rule, so `ufw allow` would not have removed it.
* **"Three independent problems" (Claude).** There were four faults; it grouped the zone-path and `api` problems together as "config or zone errors", which hid the `api` fault.

---

## Part 3d: Verdict

**Where our human diagnosis beat the AIs:**
The `api` record. Both AIs missed it, and its checker line sat next to a "named-checkzone clean" line, so syntax-focused advice cannot find it. We found it by comparing the forward zone with the reverse zone and with the scenario card, which means checking two sources of truth against each other. The firewall fault is the other case: reading `iptables -L` directly and seeing a packet counter increase on a DROP rule was stronger evidence than either AI's reasoning, and it showed that ChatGPT's order of investigation (fix BIND first, firewall later) would have sent us the wrong way.

**Where an AI genuinely helped:**
The fresh Claude noticed the key clue in the symptoms alone: a *timeout* on loopback, rather than a refusal, points at a DROP rule, not a dead named. Both AIs gave correct, well-explained DHCP fixes (including the scope-override warning), and both asked us to confirm each cause with a command before changing anything. ChatGPT's `tcpdump` suggestion (queries arriving but no replies leaving) is a good way to prove a firewall drop. In the main conversation, the AI's step-by-step coaching also saved time on commands and Git.

**What we learned about trusting AI to diagnose a live system it cannot see:**
An AI that cannot see the system gives a menu of plausible causes, and the confident-sounding ones are not necessarily the right ones: both told us to edit `listen-on` and `allow-query`, which were fine. The useful ones asked for evidence first, and the only real test was running the command ourselves and reading the output. A fix can also work while being wrong: adding an ACCEPT rule above a DROP rule would have made the checker pass for the wrong reason. We also learned to be honest about our own use of AI: the main conversation told us the faults up front, so it was not a fair test, and the blind test only means something when the AI sees just the symptoms. In the end, the trustworthy parts of our submission are the ones backed by command output in `session.log`.
