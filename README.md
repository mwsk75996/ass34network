# Assignment 34 – SRX L2 Switch, SOHO & Destination NAT

Konfiguration til SRX240 i JSON: `r2-config.json`.

## Routeren

- **R2 / SRX-C1** – fungerer både som L2-switch og router. To interne VLANs: `mySwitch2Vlan` (USERLAN) og `mySwitch1Vlan` (DMZ). De routede gateway-porte ge-0/0/1 og ge-0/0/2 er patchet fysisk ind i hver sit VLAN (ge-0/0/1↔ge-0/0/10 og ge-0/0/2↔ge-0/0/6), så L2-switching og L3-routing er adskilt. R2 er DHCP-server på USERLAN. ge-0/0/3 er uplink til LabLan/R3.
- **Source NAT (SOHO):** USERLAN og DMZ skjules bag interface-adressen 10.56.16.85 på vej mod internettet.
- **Destination NAT:** LabLan → `10.56.16.85:80` oversættes til webserveren `192.168.4.10:80` i DMZ.
- **Zoner:** USERLAN, DMZ og untrust. Default `deny-all`; kun USERLAN→untrust, USERLAN→DMZ og untrust→DMZ (http) er tilladt.
- Testet med PC1 og PC2 på USERLAN (DHCP) og PC3 som webserver i DMZ, samt et hping3 SYN-flood fra LabLan mod webserveren.

## Adresser (LLD)

| Zone/VLAN | ID | Netværk | Port | Enhed | Adresse |
|---|---|---|---|---|---|
| untrust | – | 10.56.16.0/22 | ge-0/0/3 | Uplink → R3 | 10.56.16.85/22 (statisk) |
| USERLAN | 3 | 192.168.3.0/24 | ge-0/0/1 | Gateway (patch → ge-0/0/10) | 192.168.3.1/24 |
| USERLAN | 3 | 192.168.3.0/24 | ge-0/0/11 | PC1 | DHCP .100 |
| USERLAN | 3 | 192.168.3.0/24 | ge-0/0/12 | PC2 | DHCP .101 |
| DMZ | 4 | 192.168.4.0/24 | ge-0/0/2 | Gateway (patch → ge-0/0/6) | 192.168.4.1/24 |
| DMZ | 4 | 192.168.4.0/24 | ge-0/0/7 | PC3 / myWebServer | 192.168.4.10/24 (statisk) |

- **DHCP-scope USERLAN:** 192.168.3.100–200, gateway 192.168.3.1, DNS 8.8.8.8.
- **Default route:** 0.0.0.0/0 → 10.56.16.1 (R3).

## Filer

| Fil | Indhold |
|---|---|
| `r2-config.json` | Hele SRX-konfigurationen (JSON) |
| `topology.svg` | Netværksdiagram (topologi) |
| HLD | High/Low Level Design — ligger på Google Drive |
| `testplan.md` / `testplan.png` | Testplan + resultater |

## Test (kort)

11/11 tests bestået: interfaces, L2-switching (PC1↔PC2), DHCP, begge gateways, USERLAN→DMZ web, internet via source-NAT (1598 NAT-hits), destination-NAT (curl http://10.56.16.85 → PC3), DMZ-isolation og untrust→USERLAN deny, samt uplink. Se `testplan.md` for detaljer.

## DoS-test (hping3)

SYN-flood fra en fysisk Linux-maskine på LabLan mod webserveren:

```
sudo hping3 -S -p 80 --flood 10.56.16.85
```

Wireshark registrerer en strøm af TCP SYN-pakker. Webserveren bliver uresponsiv pga. udtømning af TCP's backlog af halvåbne forbindelser (SYN-flood).
