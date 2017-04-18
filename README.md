# Úvodní informace

Tento repozitář je určen pro testování HA RADIUS serverů cesnetu, které zajištují službu eduroam pro ČR.

# Informace o serverech

- server1: r1nren.et.cesnet.cz, 78.128.211.51
- server2: r2nren.et.cesnet.cz, 78.128.211.52
- standby ip: nren.et.cesnet.cz, 78.128.211.53

Stroje budou zapojeny v režimu aktivní + standby.
Jako operační systém obou serverů byl zvolen debian Jessie.

# Konfigurace strojů

## Prerekvizity

Oba servery musí mít vzájemnou konektivitu.


# Použité zdroje
  - [1](https://www.linux-dev.org/2016/03/debian-jessie-8-3-short-howto-for-corosyncpacemaker-activepassive-cluster-with-two-nodes-and-drbdlvm/) https://www.linux-dev.org/2016/03/debian-jessie-8-3-short-howto-for-corosyncpacemaker-activepassive-cluster-with-two-nodes-and-drbdlvm/
  - [2](https://wiki.debian.org/Debian-HA/ClustersFromScratch) https://wiki.debian.org/Debian-HA/ClustersFromScratch



