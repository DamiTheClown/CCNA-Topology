# Podniková topologie — Cisco Packet Tracer Lab

Tenhle lab jsem si poskládal a nakonfiguroval v Packet Traceru, abych si v praxi pořádně zažil věci, co se probírají v CCNA. Mám teď čerstvě za sebou moduly CCNA1 (Introduction to Networks) a CCNA2 (Switching, Routing, and Wireless Essentials) a tohle je takový můj přechodový projekt, než se naplno pustím do CCNA3 (Enterprise Networking, Security, and Automation). 

Cílem bylo postavit funkční firemní síť od nuly – žádné klikání v GUI, všechno čistě přes CLI, od nahození rozhraní až po dynamické routování a simulaci aplikačních služeb.

---

## 🗺️ Architektura sítě

Plátno jsem si logicky rozdělil do tří barevných zón, které simulují reálné korporátní prostředí:

1. **Tyrkysová zóna (Pobočka / Branch Office):**
   * Tady sedí koncoví uživatelé rozdělení do logických skupin (VLAN 10 pro IT, VLAN 20 pro HR).
   * Switche se starají o přístupovou vrstvu, zatímco router funguje jako brána pro odchod ze sítě.
2. **Žlutá zóna (Tranzitní jádro / Core WAN):**
   * Představuje páteřní síť poskytovatele nebo interní Core vrstvu. 
   * Neřeší se tu žádné koncové stanice, routery v této zóně mají jediný úkol: co nejrychleji routovat traffic mezi pobočkou a centrálou. Propoje běží na úsporných `/30` subnetech.
3. **Zelená zóna (Centrála & Datacentrum / HQ):**
   * Zde běží klíčová infrastruktura sítě. Konkrétně dva servery: webový (HTTP/HTTPS) a jmenný (DNS), které poskytují služby celé firmě.

### Použité zařízení:
* **Routery:** Cisco 2911 (zvolen kvůli dostatku nativních GigabitEthernet portů a plné podpoře subinterfaců).
* **Switche:** Cisco 2960 (24-portový L2 switch, ideální na segmentaci VLAN).

---

## ⚙️ Konfigurace

Aby síť fungovala bezúdržbově a bezpečně, nasadil jsem kombinaci několika CCNA konceptů:

* **Router-on-a-Stick (Inter-VLAN Routing):** Propojení mezi routerem `R1-BRANCH` a switchem `S1-BRANCH` je nakonfigurované jako trunk. Na routeru jsem vytvořil virtuální subinterfacy (`g0/1.10` a `g0/1.20`) s enkapsulací `dot1Q`. Díky tomu spolu mohou IT a HR oddělení komunikovat, i když jsou v jiných broadcastových doménách.
* **Centralizované DHCPv4:** Koncoví uživatelé na pobočce nedostávají IP adresy ručně. Router `R1` pro ně drží DHCP pooly. Prvních 10 adres v každém subnetu je vyřazeno z přidělování (rezervace pro tiskárny, management switchů atd.). DHCP rovnou klientům posílá i adresu DNS serveru z centrály.
* **Spanning Tree Protocol (STP) - PortFast:** Na access portech switchů směrem k PC a serverům je zapnutý PortFast. Porty tak neprocházejí klasickým 30vteřinovým nasloucháním a učením, ale okamžitě přecházejí do stavu forwarding. Počítače tak dostanou IP adresu z DHCP hned po připojení kabelu.
* **Dynamické routování OSPFv2:** Místo psaní krkolomných statických cest se o směrování stará protokol OSPF pod procesem 1 v jedné páteřní oblasti (Area 0). Routery si samy vyměňují informace o sítích, které k nim jsou připojené. Každý router má natvrdo definované unikátní `router-id` kvůli přehlednosti při troubleshootingu.
* **Aplikační vrstva (DNS & HTTP):** V datacentru běží Web-Server s upraveným čistým HTML indexem. DNS server mapuje doménu `cisco.local` na IP adresu webu (`192.168.100.10`).

---

## 🚀 Ověření

Když je všechno správně nahozené, síť se dá otestovat několika kroky přímo v Packet Traceru:

### 1. Přidělení IP adresy přes DHCP
Když rozkliknem libovolný PC na pobočce (např. `PC-IT`) a přepnem IP konfiguraci na **DHCP**, musí úspěšně dostat adresu z rozsahu `192.168.10.x`, masku `255.255.255.0`, bránu `192.168.10.1` a DNS `192.168.100.11`.

### 2. Kontrola OSPF sousedství
Když skočíme do CLI prostředního routeru (`R2-CORE`) a napíšem:
```text
R2-CORE# show ip ospf neighbor
