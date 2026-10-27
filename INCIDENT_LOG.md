
# Incident Log

**Group:** DueTommor

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

---

## Fault 3 — Incorrect API DNS Address

- **Symptom (what we observed):**

  The API DNS record had the wrong IP address. It was pointing to `192.168.19.1`, which is the gateway address, instead of the required API server address `192.168.19.40`.

  We checked the DNS zone file using:

  `cat /etc/bind/db.novapeak.lab`

  Relevant output:

  ```
  ns1 IN A 192.168.19.10
  www IN A 192.168.19.20
  mail IN A 192.168.19.30
  api IN A 192.168.19.1
  ```

- **Hypothesis (what we suspected and why):**

  We suspected that the API A record was configured with the wrong IP address. The API hostname should point to `192.168.19.40`, but the zone file showed `192.168.19.1`.

- **Proof (the command that confirmed the cause):**

  `cat /etc/bind/db.novapeak.lab`

  Relevant output:

  ```
  api IN A 192.168.19.1
  ```

  This confirmed that the API DNS record was pointing to the wrong IP address.

- **Fix (what we changed):**

  We changed the API A record from `192.168.19.1` to `192.168.19.40` and increased the zone serial from `2024010101` to `2024010102` using the recorded command:

  ```
  sed -i 's/api IN A 192.168.19.1$/api IN A 192.168.19.40/; s/2024010101/2024010102/' /etc/bind/db.novapeak.lab
  ```

  Then we validated the forward zone and reloaded the DNS service.

  ```
  named-checkzone novapeak.lab /etc/bind/db.novapeak.lab
  systemctl reload named
  ```

- **Confirmation (output showing it now works):**

  We checked the DNS zone file again.

  `cat /etc/bind/db.novapeak.lab`

  Relevant output:

  ```
  api IN A 192.168.19.40
  ```

  After the firewall issue was also resolved, we tested the API DNS record using:

  `ip netns exec client dig @192.168.19.10 api.novapeak.lab +noall +answer`

  Output:

  ```
  api.novapeak.lab. 3600 IN A 192.168.19.40
  ```

  The API hostname successfully resolved to the correct IP address. The final grading checker also confirmed that all four A records were correct.

- **Found/proved by:** [Hnin Ei Ei Win - 6705142036]

---

## Fault 4 — Incorrect DHCP DNS Server

- **Symptom (what we observed):**

  The DHCP server was advertising the wrong DNS server address. Instead of using the company's DNS server `192.168.19.10`, it was giving clients the public DNS server `8.8.8.8`.

  The initial grading checker showed:

  ```
  [ 0/3 ] C9 DHCP advertises 8.8.8.8,
  expected the company DNS 192.168.19.10
  ```

- **Hypothesis (what we suspected and why):**

  We suspected that the DHCP configuration contained the wrong DNS server address. Since the company uses the internal domain `novapeak.lab`, clients need to use the company's DNS server to resolve internal hostnames.

- **Proof (the command that confirmed the cause):**

  `grep -n "domain-name-servers" /etc/dhcp/dhcpd.conf`

  Output:

  ```
  8:      option domain-name-servers 8.8.8.8;
  ```

  This confirmed that the DHCP server was configured to advertise the wrong DNS address.

- **Fix (what we changed):**

  We changed the DHCP DNS server address from `8.8.8.8` to `192.168.19.10` using the following command:

  ```
  sed -i 's/option domain-name-servers 8\.8\.8\.8;/option domain-name-servers 192.168.19.10;/' /etc/dhcp/dhcpd.conf
  ```

  We then checked the DHCP configuration using:

  `dhcpd -t -cf /etc/dhcp/dhcpd.conf`

  After the configuration passed validation, we restarted the DHCP service.

  `systemctl restart isc-dhcp-server`

- **Confirmation (output showing it now works):**

  We checked the DHCP configuration again.

  `cat /etc/dhcp/dhcpd.conf`

  Relevant output:

  ```
  subnet 192.168.19.0 netmask 255.255.255.0 {
      range 192.168.19.104 192.168.19.144;
      option routers 192.168.19.1;
      option domain-name-servers 192.168.19.10;
  }
  ```

  We also ran the grading checker to verify the result.

  `python3 /root/grading/check_lab.py --group lab01 --phase build`

  Relevant output:

  ```
  [ 3/3 ] C9 DHCP advertises the company DNS (192.168.19.10)
  ```

  The DHCP server successfully advertised the correct company DNS address. After the remaining DNS and firewall problems were resolved, the client could also resolve the company website using the DHCP-provided DNS server.

- **Found/proved by:** [Hnin Ei Ei Win - 6705142036]
