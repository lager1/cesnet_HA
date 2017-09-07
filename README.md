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

TODO
Task instaluje i balíky postfix a mailutils, které nejsou nezbytné pro práci národního RADIUS
serveru ani clusteru. Taktéž konfiguruje postfix, což může být v produkčním prostředím nežádoucí.

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

# Manuální přepnutí obou strojů

TODO

```
crm node standby
crm node online
```

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




