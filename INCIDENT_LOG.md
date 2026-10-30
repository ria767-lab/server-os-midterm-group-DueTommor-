
# Incident Log

**Group:** [DueTomorrow]  
**Members:** [ Ria-6705142022 ], [Eaint Kyar Phyu Linn-6705142010], [Chit Myat Noe-6705142017], [Hnin Ei Ei Win-6705142036], [Aung Thet Khine-6705142032]  
**Company / domain:** NovaPeak / novapeak.lab

---

## Fault 1 — Incorrect DNS Zone File Path

- **Symptom (what we observed):**

  The DNS server could not load the company domain because the configured zone file was missing. This prevented the DNS server from resolving the company's hostnames correctly.

  ```bash
  named-checkconf -z
  ```

  Output:
  ```text
  zone novapeak.lab/IN: loading from master file /etc/bind/db.MISSING failed: file not found
  zone novapeak.lab/IN: not loaded due to errors.
  ```

- **Hypothesis (what we suspected and why):**

  We suspected that the DNS configuration was pointing to an incorrect zone file. The server was trying to load `db.MISSING` instead of the actual zone file, `db.novapeak.lab`.

- **Proof (the command that confirmed the cause):**

  ```bash
  cat -n /etc/bind/named.conf.local
  ```

  Output:
  ```text
  1  zone "novapeak.lab" {
  2      type master;
  3      file "/etc/bind/db.MISSING";
  4  };
  ```

  This confirmed that the DNS configuration contained an incorrect file path.

- **Fix (what we changed):**

  We corrected the zone file path in `named.conf.local` using the following command:

  ```bash
  sed -i.bak 's|db.MISSING|db.novapeak.lab|' /etc/bind/named.conf.local
  ```

  We then validated the configuration and restarted the DNS service.

  ```bash
  named-checkconf -z
  systemctl restart named
  ```

- **Confirmation (output showing it now works):**

  ```bash
  named-checkconf -z
  ```

  Output:
  ```text
  zone novapeak.lab/IN: loaded serial 2024010101
  zone 19.168.192.in-addr.arpa/IN: loaded serial 2024010101
  ```

  The DNS zone files loaded successfully without errors. This confirmed that the incorrect file path was fixed.

- **Found/proved by:** [Aung Thet Khine - 6705142032]

- **Evidence files:**
  - `evidence/named.conf.local.before`
  - `evidence/named.conf.local.after`
  - `evidence/checker_after_fix1.txt`


---

## Fault 2 — Firewall Blocking DNS Traffic on UDP Port 53

- **Symptom (what we observed):**

  After correcting the DNS zone file path, the client still could not communicate with the DNS server. DNS requests timed out even though the DNS service was running.

  ```bash
  ip netns exec client dig @192.168.19.10 www.novapeak.lab +time=2 +tries=1
  ```

  Output:
  ```text
  ;; communications error to 192.168.19.10#53: timed out
  ;; no servers could be reached
  ```

- **Hypothesis (what we suspected and why):**

  We suspected that the firewall was blocking DNS traffic. DNS normally uses UDP port 53, so a firewall rule blocking this port could prevent clients from receiving DNS responses.

- **Proof (the command that confirmed the cause):**

  ```bash
  iptables -S
  ```

  Relevant output:
  ```text
  -P INPUT ACCEPT
  -P FORWARD ACCEPT
  -P OUTPUT ACCEPT
  -A INPUT -p udp -m udp --dport 53 -j DROP
  ```

  This showed that the firewall contained a rule dropping incoming UDP traffic on port 53.

- **Fix (what we changed):**

  We removed the firewall rule that was blocking DNS traffic using the following command:

  ```bash
  iptables -D INPUT -p udp -m udp --dport 53 -j DROP
  ```

  This allowed the DNS server to receive DNS queries without disabling the entire firewall.

- **Confirmation (output showing it now works):**

  We tested DNS resolution again from the client.

  ```bash
  ip netns exec client dig @192.168.19.10 www.novapeak.lab +time=2 +tries=1
  ```

  Relevant output:
  ```text
  ;; ->>HEADER<<- opcode: QUERY, status: NOERROR
  ;; ANSWER SECTION:
  www.novapeak.lab. 3600 IN A 192.168.19.20
  ;; Query time: 0 msec
  ;; SERVER: 192.168.19.10#53(192.168.19.10) (UDP)
  ```

  The client successfully received the correct IP address for `www.novapeak.lab`. The grading checker also improved to 18.25/25 after this repair, confirming that DNS communication was working.

- **Found/proved by:** [Aung Thet Khine - 6705142032]

- **Evidence files:**
  - `evidence/iptables_before.txt`
  - `evidence/iptables_after.txt`
  - `evidence/dns_client_before.txt`
  - `evidence/dns_client_after.txt`
