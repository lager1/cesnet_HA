# Úvodní informace

Účelem tohoto repozitáře je dokumentace pro testování HA RADIUS serverů cesnetu, které zajištují službu eduroam pro ČR.

# Informace o serverech

- server1: r1nren.et.cesnet.cz, 78.128.211.51
- server2: r2nren.et.cesnet.cz, 78.128.211.52
- standby ip: nren.et.cesnet.cz, 78.128.211.53

Stroje budou zapojeny v režimu aktivní + standby.
Jako operační systém obou serverů byl zvolen debian Jessie.

# Konfigurace strojů

## Prerekvizity

Oba servery musí mít vzájemnou konektivitu.

## Příprava

Pro souběžnou konfiguraci obou serverů je výhodné použít ssh multiplexor.
Zle použít například *mssh* nebo *csshx*.
Pokud není expliticně řečeno jinak, předpokládáme v jednotlivých krocích stejnou konfiguraci pro obou serverů.

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
systemctl disable pacamaker
```

## Konfigurace clusteru

Před samotnou konfigurací cluster je nejprve třeba konfigurovat corosync.

Otevřeme soubor `/etc/corosync/corosync.conf` v našem oblíbeném textovém editoru
a provedeme následující změny:
  - změníme `crypto_cipher` parametr z `none` na `aes256`
  - změníme `crypto_hash` z `none` na `aes256`
  - změníme `bindnetaddr` na adresu TODO
  -

# Použité zdroje
  - [1](https://www.digitalocean.com/community/tutorials/how-to-create-a-high-availability-setup-with-corosync-pacemaker-and-floating-ips-on-ubuntu-14-04) https://www.digitalocean.com/community/tutorials/how-to-create-a-high-availability-setup-with-corosync-pacemaker-and-floating-ips-on-ubuntu-14-04
  - [2](https://wiki.debian.org/Debian-HA/ClustersFromScratch) https://wiki.debian.org/Debian-HA/ClustersFromScratch
  - [3](https://www.linux-dev.org/2016/03/debian-jessie-8-3-short-howto-for-corosyncpacemaker-activepassive-cluster-with-two-nodes-and-drbdlvm/) https://www.linux-dev.org/2016/03/debian-jessie-8-3-short-howto-for-corosyncpacemaker-activepassive-cluster-with-two-nodes-and-drbdlvm/
  - [4](http://clusterlabs.org/quickstart-ubuntu.html) http://clusterlabs.org/quickstart-ubuntu.html
  - [5](https://vexxhost.com/resources/tutorials/how-to-create-a-high-availability-haproxy-setup-with-corosync-pacemaker-and-floating-ips-on-ubuntu-14-04/) https://vexxhost.com/resources/tutorials/how-to-create-a-high-availability-haproxy-setup-with-corosync-pacemaker-and-floating-ips-on-ubuntu-14-04/
  - [6](http://blog.non-a.net/2011/03/27/cluster_drbd) http://blog.non-a.net/2011/03/27/cluster_drbd
  - [7](https://www.theurbanpenguin.com/drbd-pacemaker-ha-cluster-ubuntu-16-04/) https://www.theurbanpenguin.com/drbd-pacemaker-ha-cluster-ubuntu-16-04/



