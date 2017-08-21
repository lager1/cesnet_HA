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

## Příprava

Pro souběžnou konfiguraci obou serverů je výhodné použít ssh multiplexor.
Lze použít například *mssh* nebo *csshx*.
Pokud není expliticně řečeno jinak, předpokládáme v jednotlivých krocích stejnou konfiguraci obou serverů.

K oběma serverům se připojíme pomocí:
```
mssh root@r1nren.et.cesnet.cz root@r2nren.et.cesnet.cz
```  

## Konfigurace clusteru

Před samotnou konfigurací cluster je nejprve třeba konfigurovat corosync.

Otevřeme soubor `/etc/corosync/corosync.conf` v našem oblíbeném textovém editoru
a provedeme následující změny:
  - změníme `crypto_cipher` parametr z `none` na `aes256`
  - změníme `crypto_hash` z `none` na `sha256`
  - změníme `bindnetaddr` na adresu veřejnou adresu serveru:
    - 78.128.211.51 pro r1nren.et.cesnet.cz
    - 78.128.211.52 pro r2nren.et.cesnet.cz
  - zakomentujeme `mcastport`
  - přidáme `two_node: 1` do quorum-bloku
  - přidáme `transport: udpu` do totem-bloku
  - přidáme blok nodelist, kde staticky definujeme IP adresy serverů:
  ```
nodelist {
        node {
                ring0_addr: 78.128.211.51
        }
        node {
                ring0_addr: 78.128.211.52
        }
}
  ```

Na r1nren.et.cesnet.cz dále spustíme:
```
corosync-keygen
```

Pro generování klíče je nutná dostatečná entropie. Pro generování entropie je možné příkazu dávat vstup z klávesnice.

Vygenerovaný klíč se použije pro zabezpečení komunikace mezi oběma servery. Klíč zkopírujeme na druhý server:
```
scp /etc/corosync/authkey root@r2nren.et.cesnet.cz:/etc/corosync/authkey
```

## Spuštění clusteru

Cluster je nakonfigurován, můžeme spustit corosync a pacemaker:
```
service corosync start
service pacemaker start
```

Zkontrolujeme stav clusteru:
```
crm status
```

Očekávaný výstup:
```
Stack: corosync
Current DC: r2nren.et.cesnet.cz (version 1.1.15-e174ec8) - partition with quorum
Last updated: Fri Apr 21 20:59:36 2017      Last change: Fri Apr 21 20:58:55 2017 by hacluster via crmd on r2nren.et.cesnet.cz

2 nodes and 0 resources configured

Online: [ r1nren.et.cesnet.cz r2nren.et.cesnet.cz ]

Full list of resources:

```


## Konfigurace webserveru

Před další konfigurací clusteru dodáme webserveru patřičný obsah.
Implicitní konfigurace webserveru zobrazí obsah při dotazu na libovolné doménové jméno.
Vytvoříme tedy jednouduchou stránku, která bude sama reprezentovat server, který ji předal.

Na serveru r1nren.et.cesnet.cz provedeme:
```
cat > /var/www/html/index.html << "EOF"
<!DOCTYPE html>
<html>
<head>
<title>r1nren.et.cesnet.cz</title>
</head>

<body>
<h1>r1nren.et.cesnet.cz</h1>
</body>

</html>
EOF
```

Na serveru r2nren.et.cesnet.cz provedeme:
```
cat > /var/www/html/index.html << "EOF"
<!DOCTYPE html>
<html>
<head>
<title>r2nren.et.cesnet.cz</title>
</head>

<body>
<h1>r2nren.et.cesnet.cz</h1>
</body>

</html>
EOF
```

Provedeme kontrolu pro obě doménová jména:
```
curl http://r2nren.et.cesnet.cz/
```

Očekávaný výstup:
```
<!DOCTYPE html>
<html>
<head>
<title>r2nren.et.cesnet.cz</title>
</head>

<body>
<h1>r2nren.et.cesnet.cz</h1>
</body>

</html>
```

```
curl http://r1nren.et.cesnet.cz/
```

Očekávaný výstup:
```
<!DOCTYPE html>
<html>
<head>
<title>r1nren.et.cesnet.cz</title>
</head>

<body>
<h1>r1nren.et.cesnet.cz</h1>
</body>

</html>
```


## Přidání zdrojů

Cluster je spusťen, takže můžeme přidávat zdroje. Zdroje budou mít následující jména:
- *standby_ip* - standby IP adresa
- *nginx* - webový server


### standby IP

Tuto část konfigurace budeme provádět pouze na jednom serveru. Konfigurace bude automaticky distribuována na oba uzly.

Spustíme crm shell:
```
crm configure
```

Vložíme následující příkazy:
```
property stonith-enabled=no
property no-quorum-policy=ignore
property default-resource-stickiness=100
```

Příkaz *no-quorum-policy=ignore* je důležitý, aby mohl cluster nadále bežet i v případě, že bude aktivní pouze jeden server.

Nyní přidáme vlastní zdroje:
```
primitive standby_ip ocf:heartbeat:IPaddr2 \
        params ip="78.128.211.53" nic="eth0" cidr_netmask="24" \
        meta migration-threshold=1 failure-timeout=300s \
        op monitor interval=60 timeout=300 on-fail=restart
primitive nginx ocf:heartbeat:nginx \
        meta migration-threshold=1 failure-timeout=300s \
        op monitor interval=60 timeout=300 on-fail=restart
colocation loc inf: standby_ip nginx
order ord inf: standby_ip nginx
location ip_pref standby_ip 50: r1nren.et.cesnet.cz
location nginx_pref nginx 50: r1nren.et.cesnet.cz
commit
```

Ověrímě, že vše proběhlo bez chyby:
```
show
```

Očekávaný výstup:
```
node 1317065523: r1nren.et.cesnet.cz
node 1317065524: r2nren.et.cesnet.cz
primitive nginx nginx \
	meta migration-threshold=1 failure-timeout=300s \
	op monitor interval=60 timeout=300 on-fail=restart
primitive standby_ip IPaddr2 \
	params ip=78.128.211.53 nic=eth0 cidr_netmask=24 \
	meta migration-threshold=1 failure-timeout=300s \
	op monitor interval=60 timeout=300 on-fail=restart
location ip_pref standby_ip 50: r1nren.et.cesnet.cz
colocation loc inf: standby_ip nginx
location nginx_pref nginx 50: r1nren.et.cesnet.cz
order ord inf: standby_ip nginx
property cib-bootstrap-options: \
	have-watchdog=false \
	dc-version=1.1.15-e174ec8 \
	cluster-infrastructure=corosync \
	cluster-name=debian \
	stonith-enabled=no \
	no-quorum-policy=ignore \
	default-resource-stickiness=100
```

### Vysvětlivky ke konfiguraci

#### Závislosti

Mezi jednotlivými konfigurovanými primitivy je možné definovat závislosti.
Závislosti definujeme pomocí direktivy `order`, například:
```
order o_foo_before_bar inf: foo bar
```

Definice závislostí může být povinná (mandatory) nebo doporučená (advisory).
Zavislosti sami nijak nedefinují to, kde mají být konkrétní primitiva v rámci clusteru umísťena.


#### Závislost umístění

Pro specifikaci závislosti umísťení v rámci clusteru se používá direktiva `colocation`.


# Simulace výpadku

Ověříme, že migrace adresy funguje správce simulací výpadku r1nren.et.cesnet.cz.

Výpadek serveru na 60 sekund můžeme simulovat například takto:
```
service pacemaker stop; service corosync stop; sleep 60; service pacemaker start; service corosync start;
```

Případně je možné použít alternativu:
```
iptables -P INPUT DROP ; sleep 60 ; iptables -P INPUT ACCEPT;
```


V dalším terminálu můžeme spustit ping na adresu nren.et.cesnet.cz:
```
ping nren.et.cesnet.cz
```

Po spuštění simulace výpadku přestane adresa na několik sekund odpovídat.
Potom, co cluster detekuje tento stav přemigruje adresu na r2nren.et.cesnet.cz.
Dostupnost adresy lze taktéž alternativně ověřit dotazem na adresu nren.et.cesnet.cz ve webovém prohlížeči.
Obsahem zobrazené stránky je jméno serveru, který ji poskytl.



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




