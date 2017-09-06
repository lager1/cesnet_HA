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



# Manuální přepnutí obou strojů

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




