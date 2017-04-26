# Úvodní informace

Účelem tohoto repozitáře je dokumentace pro testování HA RADIUS serverů CESNETu, které zajištují službu eduroam pro ČR.

# Informace o serverech

- server1: r1nren.et.cesnet.cz, 78.128.211.51
- server2: r2nren.et.cesnet.cz, 78.128.211.52
- standby IP: nren.et.cesnet.cz, 78.128.211.53

Stroje budou zapojeny v režimu aktivní + standby.
Jako operační systém obou serverů byl zvolen debian Jessie.

# Konfigurace strojů

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

## Konfigurace prostředí

Podle použitých zdrojů nejsou ve verzi jessie dostupné balíky pro tvorbu clusteru.
Přidáme tedy repozitáře jessie-backports:
```
cat > /etc/apt/sources.list.d/jessie-backports.list << "EOF"
deb http://http.debian.net/debian jessie-backports main
EOF
```

Obnovíme seznam zdrojů:
```
apt-get update
```

Nainstalujeme podpůrné balíky z repozitáře jessie-backports:
```
apt-get install -t jessie-backports pacemaker crmsh
```
Tímto taktéž žávoreň nainstalujeme všechny potřebné závilosti.

Dále instalujeme nginx, který bude sloužit jako indikátor dostupnosti služby:
```
apt-get install nginx
```

Je třeba zakázat automatické spuštění služby:
```
systemctl disable nginx
```

Taktéž zakážeme automatické sputění pacemakeru:
```
systemctl disable pacemaker
```

TODO

However keep in mind that you will have to execute service start pacemaker whenever you rebooted (one of) your nodes.

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
- *fence_r1nren* - fencing-agent pro r1nren.et.cesnet.cz běžící na r2nren.et.cesnet.cz
- *fence_r2nren* - fencing-agent pro r2nren.et.cesnet.cz běžící na r1nren.et.cesnet.cz

### Nginx a standby IP

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
        params ip="78.128.211.53" nic="eth1" cidr_netmask="24" \
        meta migration-threshold=2 \
        op monitor interval=20 timeout=60 on-fail=restart
primitive nginx ocf:heartbeat:nginx \
        meta migration-threshold=2 \
        op monitor interval=20 timeout=60 on-fail=restart
colocation lb-loc inf: standby_ip nginx
order lb-ord inf: standby_ip nginx
commit
```



# Použité zdroje
  - [1](https://www.digitalocean.com/community/tutorials/how-to-create-a-high-availability-setup-with-corosync-pacemaker-and-floating-ips-on-ubuntu-14-04) https://www.digitalocean.com/community/tutorials/how-to-create-a-high-availability-setup-with-corosync-pacemaker-and-floating-ips-on-ubuntu-14-04
  - [2](https://wiki.debian.org/Debian-HA/ClustersFromScratch) https://wiki.debian.org/Debian-HA/ClustersFromScratch
  - [3](https://www.linux-dev.org/2016/03/debian-jessie-8-3-short-howto-for-corosyncpacemaker-activepassive-cluster-with-two-nodes-and-drbdlvm/) https://www.linux-dev.org/2016/03/debian-jessie-8-3-short-howto-for-corosyncpacemaker-activepassive-cluster-with-two-nodes-and-drbdlvm/
  - [4](http://clusterlabs.org/quickstart-ubuntu.html) http://clusterlabs.org/quickstart-ubuntu.html
  - [5](https://vexxhost.com/resources/tutorials/how-to-create-a-high-availability-haproxy-setup-with-corosync-pacemaker-and-floating-ips-on-ubuntu-14-04/) https://vexxhost.com/resources/tutorials/how-to-create-a-high-availability-haproxy-setup-with-corosync-pacemaker-and-floating-ips-on-ubuntu-14-04/
  - [6](http://blog.non-a.net/2011/03/27/cluster_drbd) http://blog.non-a.net/2011/03/27/cluster_drbd
  - [7](https://www.theurbanpenguin.com/drbd-pacemaker-ha-cluster-ubuntu-16-04/) https://www.theurbanpenguin.com/drbd-pacemaker-ha-cluster-ubuntu-16-04/



