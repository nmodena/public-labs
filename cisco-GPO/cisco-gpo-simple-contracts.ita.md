# GPO Configuration - Advanced School of Networking

Esempio di Configurazione di GBP/GPO (Group Based Policy/Group Policy Options) per semplice micro-segmentazione nel datacenter

Data: 05/2026
Release: 1.0
Autore: Nicola Modena - CCIE #19119 - JNCIE #986

- [GPO Configuration - Advanced School of Networking](#gpo-configuration---advanced-school-of-networking)
  - [Sommario](#sommario)
  - [Laboratorio](#laboratorio)
  - [Topologia fisica](#topologia-fisica)
  - [Requisiti](#requisiti)
    - [Step-1 | Configurare il system template](#step-1--configurare-il-system-template)
    - [Step-2 | Definire i Security Groups](#step-2--definire-i-security-groups)
      - [Verifica assegnazione Type-2 MAX-IP ESG/PCTAG](#verifica-assegnazione-type-2-max-ip-esgpctag)
      - [Verifica assegnazione Type-5 IP-PREFIX ESG/PCTAG](#verifica-assegnazione-type-5-ip-prefix-esgpctag)
    - [Step-3 | definizione dei servizi e delle policy di sicurezza](#step-3--definizione-dei-servizi-e-delle-policy-di-sicurezza)
    - [Step-4 | definizione dei security contacts](#step-4--definizione-dei-security-contacts)
  - [Security group all'interno delle VLAN](#security-group-allinterno-delle-vlan)
    - [Identificazione di ulteriori security group](#identificazione-di-ulteriori-security-group)
    - [Creazione delle security policy per gli ulteriori security group](#creazione-delle-security-policy-per-gli-ulteriori-security-group)
    - [security policy complessiva](#security-policy-complessiva)

## Sommario

L'obiettivo è applicare una soluzione di micro-segmentazione che separi le componenti di Front-End, Back-End e Database di un pool di applicazioni ospitate nello stesso VRF del datacenter ma in vlan separate

Related Documentation:  [Micro-segmentation for VXLAN fabrics using group policy option (GPO)](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/106x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-106x/microsegmentation_for_vxlan_fabrics_using_gpo.html)

## Laboratorio

Il laboratorio e' composto da una mini fabric ( 2 Leaf / 2 Spine ) NEXUS 9000v ed una coppia di firewall ( active/standby ) simulata con Router IOSv.

![Physical Lab Topology](img/GPO-LAB-PHY.svg)

La versione utilizzata per i Nexus 9000v è 10.5.4 

```sh
nx-01# sh ver
Cisco Nexus Operating System (NX-OS) Software
Nexus 9000v is a demo version of the Nexus Operating System
Software
  BIOS: version
  NXOS: version 10.5(4) [Maintenance Release]
  BIOS compile time:
  NXOS image file is: bootflash:///nxos64-cs.10.5.4.M.bin
  NXOS compile time:  10/10/2025 31:00:00 [10/16/2025 21:28:37]
```

Disclaimer 1 : La versione virtuale di Nexus **NON EFFETTUA L'ENFORCEMENT DELLE POLICY E NON SETTA CORRETTAMENTE il GBP*** nei pacchetti VXLAN. puo' essere utilizzata esclusivamente per verificare la configurazione e la segnalazione del dataplane.

Nota : nella versione "conteneirizzata" di Nexus 9000v utilizzata **in Containerlab non e' possibile modificare il system template**, e' necessario utilizzare una versione VM in eve-ng o gns-lab.

## Topologia fisica

Il lab prevede un VRF DATA16 composto da 4 vlan nelle quale sono ospitati Front-End, Application Server e Database, disposti nelle diverse VLAN.

![Tenant Configuration](img/GPO-LAB-VRF.svg)

Questa l'allocazione delle vlan ed il piano di indirizzamento

vrf: DATA16 l3vni: 30016 (vlan3016)

| vlan |  vni  |    prefix    |  use        |
|:----:|:-----:|:------------:|:-----------:|
|  16  | 10016 | 10.0.16.0/24 | Front-End   |
|  17  | 10017 | 10.0.17.0/24 | Application |
|  18  | 10018 | 10.0.18.0/24 | Application |
|  19  | 10019 | 10.0.19.0/24 | Data-Base   |

## Requisiti

Come parte del processo di aumento della sicurezza interna il CISO richiede che venga implementata una policy di micro-segmentazione all'interno del datacenter che preeda la separazione delle componenti di Front-End, Application e Database, abilitando esclusivamente la comunicaizone dei protocolli richiesti al funzionamento tra i diversi componenti.
In particolare i Front-End eseguono un controllo Layer-7 e comunicano in HTTPS verso gli application server. i Database sono MySQL e sono acceduti esclusivamente dagli Application Server.
L'accesso al VRF per la pubblicazione dei servizi, ed anche per la gestione dell'infrastruttura è controllata da un border firewall.

### Step-1 | Configurare il system template

Per utilizzre GPO è necessario modificare l'allocazione delle TCAM dello switch in modo da dedicare risorse alle funzionalità di sicurezza.
Questo viene anche segnalato nel momento in cui si cerca di caricare la feature dei security group:

```sh
nx-01(config)# feature security-group
Configured/Applied routing mode template is not security-groups
```

E' quindi necessario specificare il template specifico ed in questo caso anche ri-partizionare il file-system.
al termine dell'operazione si esegue il reboot dello switch.


```sh
nx-01(config)# system routing template-security-groups
Warning: This routing template is required for feature security-group. Please ensure the feature is enabled after applying routing mode.
Warning: This routing template requires extended SSD repartioning.
Please execute 'copy running-config startup-config' and 'system flash sda resize extended'
Warning: The command will take effect after next reload.

nx-01(config)# copy ru sta
[########################################] 100%
Copy complete, now saving to disk (please wait)...
Copy complete.

nx-01(config)# system flash sda resize extended

 lcm_cli_flash_resize SSD size is 64G. SSD resize is not supported on this device
nx-01(config)# reload
```

dopo il riavvio e' possibile caricare la feature che abilità GPO

```sh
nx-01(config)# feature security-group
```

ed e' possibile verificare quali leaf hanno abilitate le funzionalità GPO.

```sh
NX-01# sh nve peers detail
Details of nve Peers:
----------------------------------------
Peer-Ip: 10.100.0.2
    NVE Interface       : nve1
    Peer State          : Up
    Peer Uptime         : 00:33:44
    Router-Mac          : 5004.0000.1b08
    Peer First VNI      : 30016
    Time since Create   : 00:33:44
    Configured VNIs     : 10016-10019,30016
    Provision State     : peer-add-complete
    Learnt CP VNIs      : 10016-10019,30016
    vni assignment mode : SYMMETRIC
    Peer Location       : N/A
    Group policy capable: yes                       <--------------------
----------------------------------------
```

NOTA: l'abilitazione del GPO avviene riconoscendo che agli annunci type-2,type-3 e type-5 viene ora aggiunta la community PCTAG (solitamente con tag 0 = nessung gruppo)

Related Documentation: [Enable GPO](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/106x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-106x/microsegmentation_for_vxlan_fabrics_using_gpo.html#task_m2r_ckn_cbc)

### Step-2 | Definire i Security Groups

Per seguire le direttive di segregazione interna sono previsti 3 security group per la segregazione interna, mentre è previsto un security-group esterno assegnato al traffico controllato dal firewall.

![Security Group Assignement](img/GPO-LAB-ESG.svg)

Grazie alla presenza di una buona organizzazione in vlan diverse delle varie componenti si utilizza una mappatura dei security group per vlan. Nel caso del security group per le applicazioni sono indicate le due vlan. In questo modo si riesce a sfruttare il meccanismo di astrazione dei security-group, che li rendono indipendenti dal contenuto e dal meccanismo di selezione indicato.

```sh
security-group 100 name FE
  type layer4-7
  match vlan 16
!
security-group 200 name APP
  type layer4-7
  match vlan 17,18
!
security-group 300 name DB
  type layer4-7
  match vlan 19
!
```

E' possibile assegnare dei Security Group anche a zone di sicurezza esterne al dominio evpn/vxlan, in questo caso viene semplicemente scelta la default-route che rappresenta tutto il traffico destinato al firewall. Le policy di enforcemente da/per il VRF saranno in questo caso sono demandate e rafforzate in modo tradizionale dal firewall.

```sh
security-group 400 name OUTSIDE
  type layer4-7
  match external-subnets vrf DATA16 ipv4 0.0.0.0/0
!
```

In questo modo si applicherà la policy alla funzionalità di routing che utilizzerà come destinazione la default route, mentre anche le operazioni di routing all'interno del VRF, utilizzeranno la mappatura specifica dei prefissi associati alle vlan. questo significa ad esempio che i prefissi /24 associati alle varie vlan interne, assumono automaticamente i rispettivi security group.

#### Verifica assegnazione Type-2 MAX-IP ESG/PCTAG

l'host 10.0.16.161 nella vlan 16 di Front-End viene identificato nel Security Group 100 (community PCTAG:0:0:100)

```sh
NX-01# sh bgp l2vpn evpn 10.0.16.161
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 10.100.0.1:32783    (L2VNI 10016)
BGP routing table entry for [2]:[0]:[0]:[48]:[5001.0007.0000]:[32]:[10.0.16.161]/272, version 298
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    10.100.0.1 (metric 0) from 0.0.0.0 (10.100.0.1)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10016 30016
      Extcommunity: RT:65000:10016 RT:65000:30016 ENCAP:8 PCTAG:0:0:100         <-------------
          Router MAC:5003.0000.1b08
```

#### Verifica assegnazione Type-5 IP-PREFIX ESG/PCTAG 

il traffico verso l'esterno che utilizzerà la default route è identificato dal Security Group 400 (community PCTAG:0:0:400)

```sh
NX-01# sh bgp l2vpn evpn 0.0.0.0 vrf DATA16
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 10.100.0.1:4    (L3VNI 30016)
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 256
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  Gateway IP: 10.0.99.2
  AS-Path: 65400 , path sourced external to AS
    10.100.0.1 (metric 0) from 0.0.0.0 (10.100.0.1)
      Origin IGP, MED not set, localpref 110, weight 0
      Received label 30016
      Received path-id 1
      Extcommunity: RT:65000:30016 ENCAP:8 PCTAG:0:0:400 Router MAC:5003.0000.1b08  <------------
```

### Step-3 | definizione dei servizi e delle policy di sicurezza

Tra i security groups sono previsti 3 tipi di traffico, HTTPS,MySQL e vista la delega del controllo al firewall viene inviato tutto il traffico IP.
Risulta essere l'equivalente della definizione di un servizio (tcp/udp/etc) su di un firewall

```sh
class-map type security match-any HTTPS
  match ipv4 tcp stateful dport 443
!
class-map type security match-any MYSQL
  match ipv4 tcp stateful dport 3306
!
class-map type security match-any ANY
  match ipv4  
```

L'indicazione *statefull* disponibile per i servizi TCP è in realtà una verifica stateless ma che controlla i flag SYN/ACK permettendo l'instaurazione delle sessioni solo nel verso indicato. Non e' possibile aprire sessioni TCP nel verso opposto indicando come porta sorgente la 443 o 3306.

Le policy di sicurezza racconlgono l'insieme dei servizi che si vogliono abilitare tra i diversi security group, le azioni possibili sono accept(default), deny e redirect, ed e' possibile specificare opzionalmente anche il log.

```sh
policy-map type security PERMIT-HTTPS
  class HTTPS
!
policy-map type security PERMIT-MYSQL
  class MYSQL
!
policy-map type security PERMIT-ANY
  class ANY
```

### Step-4 | definizione dei security contacts

L'enforcement delle policy policy di sicurezza che controllano il traffico tra i diversi security group sono definite a livello di VRF. In ambiente Cisco prendono il nome di *contratti*

Viene quindi stabilito:

- dal Front-End alle APplicazioni e' permesso esclusivamente il traffico HTTPS
- solo le applicazioni possono comunicare con il security group del database utilizzando lo specifico protocollo
- è possibile utilizzre il selettore *any* per definire una policy generica da/verso il SG 400

```sh
vrf context DATA16
  security contract source 100 destination 200 policy PERMIT-HTTPS bidir
  security contract source 200 destination 300 policy PERMIT-MYSQL bidir
  security contract source any destination 400 policy PERMIT-ANY bidir
```

Le policy sono automaticamente generate in modo bidirezionale, questo significa ad esempio nel caso di HTTPS sono sviluppate come:

```text
SG 100 TCP HIGH PORT ( SYN without ACK )  to SG 200 TCP PORT 443
SG 200 TCP port 443 ( ACK )               to SG 100 TCP HIGH PORT 
```

Nel caso di traffico IP e' indifferente il senso di applicazione, la rule per il gruppo 400 poteva essere scritta anche invertendo i membri. In questo caso le policy di sicurezza, compresa la statefull inspection e' demandata al firewall.

E' ora necessario abilitare l'enforcement delle policy andando a selezionare **la modalità di enforcement per il traffico tra security groups dove non e' stata definita esplicitamente una policy/contratto**, è puo' essere deny oppure accept.

```sh
vrf context DATA16
  security enforce tag 999 default deny
```

Contestualmente deve essere indicato un Security Group ID da utilizzare per la gestione del traffico Broadcast/Multicast, che non e' soggetto ad enforcement, ma rimane specifico al VRF.

## Security group all'interno delle VLAN

A fronte dell'attivazione di un nuovo sistema di pagamento, alcuni application server e database dovranno essere segregati in un nuovo perimetro soggetto alle normative PCI. La richiesta è che l'operazione sia il meno invasiva possibile, non si vuole modificare l'allocazione delle vlan, il piano di indirizzamento del VRF e nemmeno rinumerare Application Server e Data-Base.

### Identificazione di ulteriori security group

Per minimizzaer gli impatti si adotteranno due ulteriori security group ed una mappatura su base IP per segregare il nuovo perimetro PCI.

I due Application Server hanno indirizzo 10.0.17.171 e 10.0.18.182
I due Data-Base server hanno indirizzo 10.0.19.193 e 10.0.19.194

Vengono quindi definiti 2 nuovi security group utilizzando una mappatura per ip dei membri.

![Security Group PCI](img/GPO-LAB-PCI.svg)

```sh
security-group 201 name APP-PCI
  type layer4-7
  match connected-endpoints vrf DATA16 ipv4 10.0.17.171/32
  match connected-endpoints vrf DATA16 ipv4 10.0.18.182/32
!
security-group 301 name DB-PCI
  type layer4-7
  match connected-endpoints vrf DATA16 ipv4 10.0.19.193/32
  match connected-endpoints vrf DATA16 ipv4 10.0.19.194/32
!
```

La mappatura per IP ha precedenza rispetto alla mappatura per vlan, e gli host identificati risultano associati ai nuovi security groups.

```sh
NX-01# sh bgp l2vpn evpn 10.0.17.171
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 10.100.0.1:32784    (L2VNI 10017)
BGP routing table entry for [2]:[0]:[0]:[48]:[5001.0008.0000]:[32]:[10.0.17.171]/272, version 300
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    10.100.0.1 (metric 0) from 0.0.0.0 (10.100.0.1)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10017 30016
      Extcommunity: RT:65000:10017 RT:65000:30016 ENCAP:8 PCTAG:0:0:201.   <----------------
          Router MAC:5003.0000.1b08
```

### Creazione delle security policy per gli ulteriori security group

E' ora sufficiente creare le nuove policy per permettere il traffico HTTPS dal fronted agli application server PCI, che a loro volta potranno accedere utilizzando il protocollo MySQL ai rispettivi database PCI.

```sh
vrf context DATA16
  security contract source 100 destination 201 policy PERMIT-HTTPS bidir
  security contract source 201 destination 301 policy PERMIT-MYSQL bidir
```

L'utilizzo del selettore *any* per l'accesso al firewall e la modalità di enforcement "default deny" assicurano rispettivamente la possibibilità di utilizzare il firewall per amministrare gli host, ed inibiscono qualsiasi altra comunicazione laterale tra tutti gli altri security group all'interno del VRF.

### security policy complessiva

Per completezza viene riportata la configurazione complessiva dei security group e delle policy configurate.

```sh
security-group 100 name FE
  type layer4-7
  match vlan 16
!
security-group 200 name APP
  type layer4-7
  match vlan 17,18
!
security-group 201 name APP-PCI
  type layer4-7
  match connected-endpoints vrf DATA16 ipv4 10.0.17.171/32
  match connected-endpoints vrf DATA16 ipv4 10.0.18.182/32
!
security-group 300 name DB
  type layer4-7
  match vlan 19
!
security-group 301 name DB-PCI
  type layer4-7
  match connected-endpoints vrf DATA16 ipv4 10.0.19.193/32
  match connected-endpoints vrf DATA16 ipv4 10.0.19.194/32
!
security-group 400 name OUTSIDE
  type layer4-7
  match external-subnets vrf DATA16 ipv4 0.0.0.0/0
!
vrf context DATA16
  security contract source any destination 400 policy PERMIT-ANY
  security contract source 100 destination 200 policy PERMIT-HTTPS
  security contract source 100 destination 201 policy PERMIT-HTTPS
  security contract source 200 destination 300 policy PERMIT-MYSQL
  security contract source 201 destination 301 policy PERMIT-MYSQL
  security enforce tag 999 default deny
```

