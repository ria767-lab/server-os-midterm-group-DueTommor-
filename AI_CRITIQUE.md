# AI Usage and Critique

Group: DueTomorr-

Members:

- Ria — 6705142022
- Eaint Kyar Phyu Linn — 6705142010
- Chit Myat Noe — 6705142017
- Hnin Ei Ei Win — 6705142036
- Aung Thet Khine — 6705142032

Company/domain: NovaPeak / novapeak.lab

## Part 3a: Declaration (required)
| AI tool | What we used it for |
| --- | --- |
| OpenAI Codex | Interpreted assignment instructions and terminal outputs, suggested diagnostic and repair commands, helped collect submission files, and drafted documentation. |
| Google Gemini | Evaluated a description containing the four established faults and proposed explanations, repairs, and verification commands. |

## Part 3b: Confirm you diagnosed first

We completed the repairs before the Gemini comparison. However, we used OpenAI Codex during diagnosis and repair, and prepared INCIDENT_LOG.md afterward using the recorded session. Therefore, we cannot claim that we diagnosed the faults independently before using AI.

The four faults confirmed through commands we executed were:

1. The forward-zone declaration referenced `/etc/bind/db.MISSING` instead of `/etc/bind/db.novapeak.lab`.
2. An INPUT firewall rule dropped UDP traffic to DNS port 53.
3. The API A record contained `192.168.19.1` instead of `192.168.19.40`.
4. DHCP gave `8.8.8.8` instead of the company DNS server, `192.168.19.10`.

We verified the repairs using the client-side checker, which reached 25/25. Our evidence is recorded in `session.log` and explained in `INCIDENT_LOG.md`. The Gemini comparison was performed afterward using the established evidence.

## Part 3c: Putting the AIs to the test
We gave Codex our terminal outputs while investigating the lab. After fixing the problems, we gave Gemini a description of the four faults and their evidence. The tools received different inputs at different times, so this was not an equal test of independent diagnosis.


| Fault (in our words) | We found it? | AI #1 (OpenAI Codex) | AI #2 (Google Gemini) | Was each AI actually right? |
| --- | --- | --- | --- | --- |
| DNS pointed to the missing file `db.MISSING`. | We confirmed it using commands suggested by Codex. | Change the path to `db.novapeak.lab`, validate, and restart DNS. | Change the same path and reload BIND. | Both gave the correct file change. After our repair, C4 passed and the score increased from 11 to 12. We did not test Gemini’s exact reload command. |
| A firewall rule blocked UDP port 53. | We checked the listeners and firewall rule with Codex’s help. | Remove the specific DROP rule. | Remove the rule, or add an ACCEPT/UFW rule. | Removing the rule worked. Client DNS became reachable, and the score reached 18.25. We did not test Gemini’s alternatives. |
| The API record had the wrong IP address. | We confirmed it by checking the zone file and checker output. | Change the address to `192.168.19.40`, increase the serial, and restart DNS. | Make the same correction and reload the zone. | Both suggested the correct address. C2 passed and the score reached 19. Gemini’s reload command was not tested. |
| DHCP gave clients the wrong DNS server. | We confirmed it in `dhcpd.conf` and the client checker results. | Replace `8.8.8.8` with `192.168.19.10`, validate, and restart DHCP. | Make the same change, then renew the client lease using `dhclient` on `lan0`. | The server change worked. C9 and C11 passed, giving 25/25. Gemini’s lease-renewal command needed more checking because the client is in a separate namespace. |

### Suggestions that were wrong, misleading, or unverified

**Gemini’s client-interface assumption**

Gemini suggested:

`dhclient -r lan0 && dhclient lan0`

However, `lan0` is the server-side interface in our lab. The client is in a separate namespace. We would need to check the client’s interface and installed DHCP tool first. We did not run this command.

**Gemini’s DNS test**

Gemini suggested:

`dig @192.168.19.10 novapeak.lab +short`

Our zone file did not contain an A record for `novapeak.lab` itself. Therefore, an empty answer would not automatically mean DNS was broken. Testing a known host, such as `api.novapeak.lab`, would give clearer evidence.

**Gemini’s syntax-check explanation**

Gemini correctly said that valid syntax does not prove the system works. However, our `named-checkconf -z` command did detect the missing zone file. Its explanation needed to distinguish the different validation options.

**Untested Gemini commands**

We did not test Gemini’s UFW or alternative reload commands. We cannot call them successful repairs, but being untested does not automatically make them wrong.

**Limitations in Codex’s help**

Codex first gave us general document templates. Later, the handout showed that we had to use the official templates, so we needed to change the format.

Codex also helped prepare the Gemini prompt, which already contained the confirmed faults. This means Gemini was reviewing supplied evidence rather than discovering the faults itself. The prompt also did not clearly explain the client namespace.

### How we verified the repairs

Our recorded checker scores were:

- Before repairs: 11/25
- After fixing the zone path: 12/25
- After removing the firewall rule: 18.25/25
- After correcting the API record: 19/25
- After correcting the DHCP DNS option: 25/25

The final checker confirmed that the DNS records were correct, DHCP advertised the company DNS server, and the client resolved `www` using its leased resolver.

These results are recorded in `session.log` and `checker_result.txt`. They prove the repairs we performed, not every alternative command suggested by the AIs.

## Part 3d: Verdict

- Where our human diagnosis beat the AIs: We cannot say our diagnosis was independent because Codex helped us throughout. Our main contribution was running the commands, collecting real outputs, and checking whether each repair worked. We also used the lab details to notice that Gemini’s client-interface assumption needed correction.

- Where an AI genuinely helped: Codex helped us understand the errors and investigate one problem at a time. Gemini explained the same causes and why correct syntax does not always mean correct behavior. However, Gemini already had the fault evidence in its prompt.

- What we learned about trusting AI to diagnose a live system it cannot see: AI can give useful explanations, but it may assume the wrong interface, tool, or environment. We should understand commands before running them and verify the results from the client. We should also separate advice that worked from advice we never tested.
