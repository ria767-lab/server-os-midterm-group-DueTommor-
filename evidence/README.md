# Evidence Index

This folder contains the diagnostic and verification evidence
for the four DNS and DHCP faults in the Server OS midterm.

## Initial Diagnosis

- [Initial checker results](checker_initial.txt)
- [DNS client test before repairs](dns_client_before.txt)
- [DNS listener information before repairs](dns_listeners_before.txt)

## Fault 1 — Incorrect DNS Zone File Path

- [DNS configuration before repair](named.conf.local.before)
- [DNS zone file before repair](db.novapeak.lab.before)
- [DNS checker result after the first repair](checker_after_fix1.txt)

## Fault 2 — Firewall Blocking DNS

- [Firewall rules before repair](iptables_before.txt)
- [Firewall rules after repair](iptables_after.txt)
- [DNS client test after repair](dns_client_after.txt)
- [DNS checker result after the second repair](checker_after_fix2.txt)

## Fault 3 — Incorrect API DNS Address

- [DNS zone file before repair](db.novapeak.lab.before)
- [DNS zone file after repair](db.novapeak.lab.after)
- [API DNS lookup after repair](api_dns_after.txt)
- [DNS zone validation after repair](api_zone_validation_after.txt)

## Fault 4 — Incorrect DHCP DNS Server

- [DHCP configuration before repair](dhcpd.conf.before)
- [DHCP configuration after repair](dhcpd.conf.after)
- [DHCP configuration validation](dhcp_validation_after.txt)

## Final Verification

- [Final checker results](checker_after_fix4.txt)
