# AI Usage and Critique

Group: TODO
Members: TODO (name + GitHub username, one per line)

---

## Part 3a: Declaration (required)

| AI tool | What we used it for |
| --- | --- |
| Claude (claude.ai) | Uploaded the midterm project zip and asked it to review it. In its first reply it listed the four planted faults and some design issues in the checker and handout. It then coached us step by step through the lab: told us which commands to run, interpreted the output we pasted, and suggested fixes (all four faults, and the git/PowerShell steps to submit). It also drafted INCIDENT_LOG.md from our session output, and the draft hypothesis lines in it. We ran every command ourselves in the playground; Claude had no access to the system. |
| TODO: second AI tool (name) | TODO: what it was used for (the 3c test below) |
| TODO: third AI tool, if any | TODO |

Because Claude named the faults before we had diagnosed them, the diagnosis in 3b below was **not** fully independent. TODO: add anything else any member used (autocomplete, search summaries, translation, etc.).

---

## Part 3b: Confirm you diagnosed first

TODO: write this in your own words and keep it truthful. Suggested honest structure:

* What we did ourselves: ran the checker, read each symptom, ran the diagnostic commands (`named-checkconf -z`, `iptables -L`, `dig` from the client namespace, `dhcpcd -U` on the client), applied each fix, and re-ran the checker until 25/25.
* What the AI did: named the four faults in advance and guided the order of diagnosis (see 3a).
* Our own record of the diagnostic path is INCIDENT_LOG.md and session.log.

---

## Part 3c: Putting the AIs to the test

We gave the following to at least two AI tools: TODO: describe exactly what you pasted (for example: "the first checker output and the symptom commands, no config files, no hints"). Each test was run in a **fresh conversation** with no earlier context. Raw answers are saved in: TODO (e.g. `ai-answers/`, optional).

| Fault (in our words) | We found it? | AI #1 (TODO name) | AI #2 (TODO name) | Was each AI actually right? |
| --- | --- | --- | --- | --- |
| Forward zone `file` path points at `/etc/bind/db.MISSING` (zone never loads) | TODO | TODO | TODO | TODO |
| Firewall rule `DROP udp dpt:53` in INPUT (timeouts, even on loopback) | TODO | TODO | TODO | TODO |
| `api` A record is 192.168.19.1 instead of 192.168.19.40 (forward and reverse zones disagree) | TODO | TODO | TODO | TODO |
| DHCP advertises 8.8.8.8 instead of 192.168.19.10 | TODO | TODO | TODO | TODO |

For each AI suggestion that was WRONG or misleading, describe it and how you verified it was wrong:

* TODO (example shape: "AI #2 told us to change `listen-on`. We checked with `ss -lunp` that named already listens on 192.168.19.10, so this was not the cause.")

---

## Part 3d: Verdict

**Where our human diagnosis beat the AIs:**

TODO

**Where an AI genuinely helped:**

TODO

**What we learned about trusting AI to diagnose a live system it cannot see:**

TODO
