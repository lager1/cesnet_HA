# Úvodní informace

Účelem tohoto repozitáře je dokumentace pro testování HA RADIUS serverů CESNETu, které zajištují službu eduroam pro ČR.

# Informace o serverech

toplevel:
  - server1: r1nren.et.cesnet.cz, 78.128.211.51
  - server2: r2nren.et.cesnet.cz, 78.128.211.52
  - standby IP: nren.et.cesnet.cz, 78.128.211.53

"organizace":
  - server1: org1.et.cesnet.cz, 78.128.211.72, RadSec, realm1.cz
  - server2: org2.et.cesnet.cz, 78.128.211.73, IPsec, realm2.cz


Stroje budou zapojeny v režimu aktivní + standby.
Jako operační systém všech serverů byl zvolen debian Stretch.

# Konfigurace toplevel strojů

## Prerekvizity

Oba servery musí mít vzájemnou konektivitu. Taktéž je vhodné, aby měly oba servery vzájemně stejný čas, ideálně jednotný dle ntp.

## Spuštění konfigurace pomocí ansible

Konfigurace předpokládá, že oba uzly nemají nakonfigurované žádné zdroje v clusteru.
V případě, že mají, nebudou prováděny žádné změny.
Pokud v clusteru existují nějaké zdroje a mají jiné vlastnosti než
v aktuální ansible konfiguraci, nebudou změny konfigurovány.
Pro konfiguraci změn je třeba manuálně vymazat konfiguraci clusteru pomocí příkazu:
```
cibadmin -E --force
```

Vlastní konfigurace:
```
git clone https://github.com/lager1/cesnet_HA.git
cd cesnet_HA
# před spuštením konfigurace se podívejte, zda není třeba vyřešit nějaké TODO!!!
grep -r TODO *
# pokud jsou všechna TODO vyřešena, můžeme spusit konfiguraci
cd ansible
ansible-playbook -i inventory nren_ha_playbook.yml
```

### Problémy

#### Spuštění služeb na slave (dle konfigurace) uzlu

V případě, že služby běží na slave uzlu a následně je ručne vymazána veškerá konfigurace,
po nasazení nové konfigurace se cluster pokusí služby sputit na master uzlu,
ale pravděpodobně z důvodu preferencí jednotlivých zdrojů se mu to nepovede a následkem toho
jsou všechny služby spušteny na slave uzlu.

Další příčiny tohoto problému nejsou známy.

## Konfigurace pomocí ansible

Oba toplevel uzly jsou spravovány pomocí ansible. Veškerá konfigurace je umísťena v adresáři ansible.
Struktura adresáře je následující:

```
ansible
  `-- ansible.cfg 			- konfigurace ansible
   -- group_vars			- konfigurace skupin
     `-- nren_radius.yml		- konfigurace skupiny nren_radius
   -- host_vars				- proměnné hostů
     `-- r1nren.et.cesnet.cz.yml	- konfigurace r1nren.et.cesnet.cz
      -- r2nren.et.cesnet.cz.yml	- konfigurace r2nren.et.cesnet.cz
   -- inventory				- definice spravovaných serverů
   -- nren_ha_playbook.yml		- playbook pro HA
   -- roles
      `-- ansible-pacemaker		- role pro konfiguraci HA
```

Veškerá konfigurační logika je realizována pomocí role ansible-pacemaker.

## Popis role ansible-pacemaker

Roli tvoří 4 hlavní tasky, které realizují potřebnou konfiguraci:
  - install.yml
  - services.yml
  - corosync.yml
  - cluster.yml

### Task install.yml

Task install.yml je zodpovědný za instalaci veškerých potřebných balíků.

### Task services.yml

Task services.yml je zodpovědný za vytvoření unit filu všech podpůrných služeb
a vlastních skriptů, v případě, že jsou potřeba.
Task dále zodpovídá za to, že všechny clusterované služby NEjsou spravovány operačním systémem
a všechny cluster spravující jsou spravovány operačním systémem.

### Task corosync.yml

Task corosync.yml je zodpovědný za konfiguraci služby corosync.

### Task cluster.yml

Task cluster.yml je zodpovědný za konfiguraci vlastností, zdrojů a omezení clusteru.
Veškerá konfigurace tohoto tasku je definována v `ansible/group_vars/nren_radius.yml`.

Konfigurace je mimo jiné komentována v samotném souboru.

#### Důležité poznámky k této konfiguraci

Seznam `pacemaker_cluster_resources` definuje zdroje clusteru.
Jde tedy o služby, u kterých je je snaha zajistit vysokou dostupnost.

Konfigurační volba  `provider` každého zdroje určuje agenta,
která bude daný zdroj spravovat.
Správa zdroje v tomto kontextu znamená spuštení, ukončení a restart.
Pomocí jména agenta je možné odvodit jméno manuálové stránky,
například manuálové stránky agenta `'ocf:pacemaker:ping'` je možné prohlížet příkazem `man ocf_pacemaker_ping`.

Všechny možné agenty je možné zobrazit příkazem: `pcs resource agents`.
Všechny možné providery je možné zobrazit příkazem: `pcs resource providers`.

Položka `options` konkrétního zdroje určuje doplňující volby, se kterými bude agent pracovat.
Všechny doplňující volby jsou popsány v manuálových stránkách daného agenta.

Položka `meta` konkrétního zdroje určuje jeho dodatečnou konfiguraci nezávisle na agentovi.

Položka `op` konkrétního zdroje určuje akce vykonávané clusterem na agentovi.
Položka `op_options` definuje dodatečné volby těchto akcí.



Seznam `pacemaker_cluster_cloned_resources` definuje klonované zdroje clusteru.
Klonované zdroje jsou takové zdroje, které běhí na každém uzlu clusteru (v případě, že je jich stejný počet).


Seznam `pacemaker_cluster_grouped_resources` definuje zdroje clusteru,
které mají být součástí skupiny.
Pořadí zdrojů určuje jejich závislosti.
Druhý definovaný zdroj závisí na prvním, třetí na druhém a podobně.
Aby mohl být spušten druhý zdroj, musí být aktivní první, dále analogicky.
Vypínání zdrojů je realizováno pomocí stejného principu pouze s opačným pořadím.
Clenství všech zdrojů ve skupině zajišťuje spuštení na stejném uzlu clusteru.

Seznam `pacemaker_cluster_constraints` definuje omezení clusteru.

Seznam `pacemaker_cluster_settings:` definuje nastavení celého clusteru.

Přidání nového zdroje do konfigurace ansiblu a následná konfigurace clusteru bez odebrání veskěré konfigurace clusteru
přidá nový zdroj do celého clusteru, ale zdroj nebude zařazen ve skupině.

# Užitečné příkazy

Zobrazí aktuální stav clusteru
```
crm status
```

Zobrazí aktuální konfiguraci clusteru
```
crm configure show
```

Zobrazí aktuální skóre všech konfigurovaných zdrojů:
```
crm_simulate -sL
```

Zobrazí aktuální stav clusteru. Přepínače umožňují různé typy výstupů a jednorázové nebo kontinuální sledování stavu.
```
crm_mon
```

Příkaz pro práci s počtem selhání konkrétního zdroje.
```
crm_failcount
```

Zobrazí aktuální operace zdrojů. Lze filtrovat na zdroj (-r) nebo uzel (-N).
```
crm_resource -O
```

Zobrazí všechny operace zdrojů. Lze filtrovat na zdroj (-r) nebo uzel (-N).
```
crm_resource -o
```

Přepnutí r2 do standby módu z r1:
```
root@r1nren:~# crm_standby -v true -N r2nren.et.cesnet.cz
```

Přepnutí r2 do online módu z r1:
```
root@r1nren:~# crm_standby -v false -N r2nren.et.cesnet.cz
```


# Testovací scénáře a jejich výsledky

## Vypnutí jednoho z uzlů

### Vypnutí aktivního uzlu

Acpi vypnutí postupně vypně všechny zdroje na aktivním uzlu a následně je zapne ne neaktivím.
Po opětovném zapnutí serveru má v rámci clusteru stav OFFLINE. Stav na "restartovanem" ulzu:

```
Oct 09 14:15:54 r2nren.et.cesnet.cz systemd[1]: Starting Corosync Cluster Engine...
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [MAIN  ] Corosync Cluster Engine ('2.4.2'): started and ready to provide service.
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [MAIN  ] Corosync built-in features: dbus rdma monitoring watchdog augeas systemd upstart xmlconf qdevices qnetd snm
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [TOTEM ] Initializing transport (UDP/IP Unicast).
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [TOTEM ] Initializing transmit/receive security (NSS) crypto: aes256 hash: sha256
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [TOTEM ] The network interface is down.
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [SERV  ] Service engine loaded: corosync configuration map access [0]
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [QB    ] server name: cmap
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [SERV  ] Service engine loaded: corosync configuration service [1]
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [QB    ] server name: cfg
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [SERV  ] Service engine loaded: corosync cluster closed process group service v1.01 [2]
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [QB    ] server name: cpg
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [SERV  ] Service engine loaded: corosync profile loading service [4]
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [SERV  ] Service engine loaded: corosync resource monitoring service [6]
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [WD    ] No Watchdog /dev/watchdog, try modprobe <a watchdog>
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [WD    ] resource load_15min missing a recovery key.
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [WD    ] resource memory_used missing a recovery key.
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [WD    ] no resources configured.
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [SERV  ] Service engine loaded: corosync watchdog service [7]
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [QUORUM] Using quorum provider corosync_votequorum
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [VOTEQ ] Waiting for all cluster members. Current votes: 1 expected_votes: 2
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [SERV  ] Service engine loaded: corosync vote quorum service v1.0 [5]
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [QB    ] server name: votequorum
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [SERV  ] Service engine loaded: corosync cluster quorum service v0.1 [3]
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [QB    ] server name: quorum
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [TOTEM ] adding new UDPU member {78.128.211.51}
Oct 09 14:15:55 r2nren.et.cesnet.cz corosync[584]:   [TOTEM ] adding new UDPU member {78.128.211.52}
Oct 09 14:15:57 r2nren.et.cesnet.cz corosync[584]:   [TOTEM ] The network interface [78.128.211.52] is now up.
Oct 09 14:15:57 r2nren.et.cesnet.cz corosync[584]:   [TOTEM ] adding new UDPU member {78.128.211.51}
Oct 09 14:15:57 r2nren.et.cesnet.cz corosync[584]:   [TOTEM ] adding new UDPU member {78.128.211.52}
Oct 09 14:15:57 r2nren.et.cesnet.cz corosync[584]:   [QB    ] Denied connection, is not ready (584-1411-18)
Oct 09 14:15:58 r2nren.et.cesnet.cz corosync[584]:   [QB    ] Denied connection, is not ready (584-1411-18)
Oct 09 14:15:59 r2nren.et.cesnet.cz corosync[584]:   [QB    ] Denied connection, is not ready (584-1411-18)
Oct 09 14:16:00 r2nren.et.cesnet.cz corosync[584]:   [QB    ] Denied connection, is not ready (584-1411-18)
Oct 09 14:16:01 r2nren.et.cesnet.cz corosync[584]:   [QB    ] Denied connection, is not ready (584-1411-18)
Oct 09 14:16:02 r2nren.et.cesnet.cz corosync[584]:   [QB    ] Denied connection, is not ready (584-1411-18)
Oct 09 14:16:03 r2nren.et.cesnet.cz corosync[584]:   [QB    ] Denied connection, is not ready (584-1411-18)
Oct 09 14:16:04 r2nren.et.cesnet.cz corosync[584]:   [QB    ] Denied connection, is not ready (584-1411-18)
Oct 09 14:16:05 r2nren.et.cesnet.cz corosync[584]: corosync: votequorum.c:2065: message_handler_req_exec_votequorum_nodeinfo: Assertion `sender_node != NULL' failed.
Oct 09 14:16:04 r2nren.et.cesnet.cz systemd[1]: corosync.service: Main process exited, code=killed, status=6/ABRT
Oct 09 14:16:05 r2nren.et.cesnet.cz systemd[1]: Failed to start Corosync Cluster Engine.
Oct 09 14:16:05 r2nren.et.cesnet.cz systemd[1]: corosync.service: Unit entered failed state.
Oct 09 14:16:05 r2nren.et.cesnet.cz systemd[1]: corosync.service: Failed with result 'signal'.
```

Problém je, že corosync ani pacemaker na "restartovaném" uzlu neběží.

V případě restartu pomocí příkazu reboot k tomu nedojde. V případě, že je použit reboot z webového rozhraní vmware, k problému také nedojde.

Důvodem problémového stavu bylo pravděpodobně všeho dynamické nastavení sítě, které mohlo způsobit race condition mezi DHCP a startem démonů.
Při převedení na statickou konfiguraci se problém již neobjevuje.

### Vypnutí neaktivního uzlu

Stejné jako v případě aktivního uzlu, jen nemigruje žádné zdroje.

## Znemožnení komunikace mezi uzly

### zahazování komunikace pro druhý uzel

r1:
service firewall stop
iptables -A INPUT -s 78.128.211.52 -j DROP

r2:
service firewall stop
iptables -A INPUT -s 78.128.211.51 -j DROP

#### Spuštění na aktivním uzlu

Po přidání pravidla do firewallu na sebe oba uzly vzájemně nevidí.
Pasivní uzel usoudí, že aktivní uzel umřel a převezme veškeré služby.

Po vymazání pravidla je detekováno, že služby jsou spuštěny na obou uzlech.
Služby jsou na obou uzlech vypnuty a spuštěny na nodu s nejvyšší preferencí.


Nejvzyšší preferencí je myšleno:
```
root@r1nren:~# pcs constraint show --full
Location Constraints:
  Resource: group_eduroam.cz
    Enabled on: r1nren.et.cesnet.cz (score:50) (id:group_pref)
    Constraint: no_ping
      Rule: boolean-op=or score=-INFINITY  (id:no_ping-rule)
        Expression: not_defined pingd  (id:no_ping-rule-expression)
        Expression: pingd lte 0  (id:no_ping-rule-expression-0)
Ordering Constraints:
Colocation Constraints:
Ticket Constraints:
```

#### Spuštění na neaktivním uzlu

Stejné jako v případě aktivního uzlu.

## Zapnutí služby na neaktivním uzlu

pro jiné než systemd zdroje lze pomocí:

```
pcs resource debug-start zdroj
```

### standby\_ip

Po přidání zdroje na neaktivní uzel mají oba uzly přidělenou horkou adresu.

Vlastní přidání zdroje nejspíš způsobí, že neaktivní uzel z adresy začne komunikovat
a komunikace na horkou adresu pak přichází na neaktivní uzel.

Není jasné, proč se celá testovací instance chová takto, není ani jasné, zda by se stejné chování objevilo na fyzickém HW.

### offline\_file

Přidání zdroje na neaktivním uzlu odebere soubor `/etc/OFFLINE` což je hlavní indikátor pro
určení master uzlu při nasazování playbooku.

TODO

### racoon

Spuštění démona způsobí chybu a démon odmítá nastartovat,
protože nemá k dispozici adresu, ze které potřebuje komunikovat.

### radiator

Služba je schopna nastartovat, ale neprovádí žádnou komunikaci,
protože nemá k dispozici horkou adresu.

### eduroam\_ping

Pokud není zároveň spuštena služba offline\_file služba nenastartuje.

## Výmaz konfigurace a následné vypnutí služby na aktivním uzlu


# Manuální přepnutí obou strojů

V případě, že potřebujeme přepnout neaktiviní a aktivní uzel
nebo případně provést na jednom ze serverů nějakou údržbu, zle využít tento přístup.

Na aktivím uzlu spustíme:

```
crm node standby
```

Tento příkaz způsobí, že se tento uzel přesune do standby režimu.
V standby režimu není dovoleno žádným zdrojům, aby byly na tomto uzlu spuštěny.
Tím docílíme toho, že z aktivního uzlu budou všechny zdroje přesunuty na jiný uzel.

Na standby ulzu můžeme následně provést požadovanou údržbu.
Potom spustíme příkaz:

```
crm node online
```

Tento příkaz způsobí, že se tento uzel přesune do online režimu a bude moci opět být hostitelem zdrojů.

# Manuální přepnutí adresy obou strojů

Přepnutím strojů je myšleno převzetí standby IP adresy zvoleným strojem.
U serverů ve stejné LAN se projevé změna také v případě použití obou způsobů popsaných níže.

## První způsob
`arpsend -U -c 1 -i 78.128.211.53 -e 78.128.211.1 eth0`
rychlost migrace je instantní, ale je třeba nastavit adresu brány jako cílovou adresu v ARPu!

Tento příkaz odesílá otázku na to, jakou MAC adresu používá brána se zdrojovou (standby) IP adresou a MAC adresou svého rozhraní.
Brána na to odpoví svojí MAC adresou a při tom si updatuje IP-MAC cache tak, aby to odpovídalo aktuálnímu stavu.

## Druhý způsob
`arping -c 1 -A -b -s 78.128.211.53 -I eth0 78.128.211.53`
Tento příkaz funguje také. Je odesílana pouze nevyžádaná odpověď s danou IP a MAC adresou.
Jde o tzv gratuitous ARP.

Pozor - arping je z balíku iputils-arping, ne arping!!


# Použité zdroje
  - [1](https://www.digitalocean.com/community/tutorials/how-to-create-a-high-availability-setup-with-corosync-pacemaker-and-floating-ips-on-ubuntu-14-04) https://www.digitalocean.com/community/tutorials/how-to-create-a-high-availability-setup-with-corosync-pacemaker-and-floating-ips-on-ubuntu-14-04
  - [2](https://wiki.debian.org/Debian-HA/ClustersFromScratch) https://wiki.debian.org/Debian-HA/ClustersFromScratch
  - [3](https://www.linux-dev.org/2016/03/debian-jessie-8-3-short-howto-for-corosyncpacemaker-activepassive-cluster-with-two-nodes-and-drbdlvm/) https://www.linux-dev.org/2016/03/debian-jessie-8-3-short-howto-for-corosyncpacemaker-activepassive-cluster-with-two-nodes-and-drbdlvm/
  - [4](http://clusterlabs.org/quickstart-ubuntu.html) http://clusterlabs.org/quickstart-ubuntu.html
  - [5](https://vexxhost.com/resources/tutorials/how-to-create-a-high-availability-haproxy-setup-with-corosync-pacemaker-and-floating-ips-on-ubuntu-14-04/) https://vexxhost.com/resources/tutorials/how-to-create-a-high-availability-haproxy-setup-with-corosync-pacemaker-and-floating-ips-on-ubuntu-14-04/
  - [6](http://blog.non-a.net/2011/03/27/cluster_drbd) http://blog.non-a.net/2011/03/27/cluster_drbd
  - [7](https://www.theurbanpenguin.com/drbd-pacemaker-ha-cluster-ubuntu-16-04/) https://www.theurbanpenguin.com/drbd-pacemaker-ha-cluster-ubuntu-16-04/
  - [8](http://clusterlabs.org/doc/en-US/Pacemaker/1.1-plugin/html/Clusters_from_Scratch/_prevent_resources_from_moving_after_recovery.html) http://clusterlabs.org/doc/en-US/Pacemaker/1.1-plugin/html/Clusters_from_Scratch/_prevent_resources_from_moving_after_recovery.html
  - [9](https://www.hastexo.com/resources/hints-and-kinks/mandatory-and-advisory-ordering-pacemaker/) https://www.hastexo.com/resources/hints-and-kinks/mandatory-and-advisory-ordering-pacemaker/
  - [10](http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-resource-ordering.html) http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-resource-ordering.html
  - [11](http://clusterlabs.org/doc/Colocation_Explained.pdf) http://clusterlabs.org/doc/Colocation_Explained.pdf
  - [12](http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-resource-colocation.html) http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-resource-colocation.html
  - [13](http://clusterlabs.org/doc/en-US/Pacemaker/1.0/html/Pacemaker_Explained/s-resource-operations.html) http://clusterlabs.org/doc/en-US/Pacemaker/1.0/html/Pacemaker_Explained/s-resource-operations.html


# Dodatečné zdroje - ansible
  - https://docs.ansible.com/ansible/pacemaker_cluster_module.html
  - https://github.com/styopa/ansible-pacemaker
  - https://github.com/sbitio/ansible-pacemaker
  - https://github.com/OndrejHome/ansible.ha-cluster-pacemaker
  - https://github.com/leucos/ansible-pacemaker-corosync
  - https://github.com/mrlesmithjr/ansible-pacemaker
  - https://github.com/redhat-openstack/ansible-pacemaker
  - https://github.com/sbitio/ansible-corosync


# TODO

Pustit cely playbook na "cistych" strojich.

# TODO 2

TODO: jak vubec muze takovyhle stav vzniknout?!!

```
Node r2nren.et.cesnet.cz: standby
Online: [ r1nren.et.cesnet.cz ]

Full list of resources:

 Clone Set: clone_ping_gw [ping_gw]
     Started: [ r1nren.et.cesnet.cz ]
     Stopped: [ r2nren.et.cesnet.cz ]
 Resource Group: group_eduroam.cz
     standby_ip	(ocf::heartbeat:IPaddr2):	Started r2nren.et.cesnet.cz
     offline_file	(systemd:offline_file):	Stopped
     radiator	(systemd:radiator):	Started r1nren.et.cesnet.cz
     racoon	(systemd:racoon):	Stopped
     eduroam_ping	(systemd:eduroam_ping):	Stopped
     mailto	(ocf::heartbeat:MailTo):	Started r1nren.et.cesnet.cz
```

