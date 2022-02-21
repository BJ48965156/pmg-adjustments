# Meine Anpassungen für Proxmox Mail Gateway 7.1
Nachfolgend beschreibe ich meine Anpassungen für das Proxmox Mail Gateway.

## Mein Aufbau
3 VMs in Proxmox mit dem Image "debian-11-generic-amd64" und cloud-init. Die VMs haben jeweils 2 Netzwerkinterfaces. Eins für die externe Kommunikation und eins für die interne

Einer der Server wird der Master-Server. Hier findet die Administration statt. Die beiden anderen Server werden die Filter-Nodes. Ich bezeichne sie mit master, node1, node2. Das internet Netz ist 192.168.0.0/24.

master - 192.168.0.1
node1 - 192.168.0.2
node2 - 192.168.0.3

Weiterhin habe ich einige kommerzielle Dienste aboniert. Die Nutzung ist ggf. optional. Bitte immer die Lizenzen / Nutzungsbedinngungen der jeweiligen Anbieter berücksichtigen!

## Image spezifische Anpassungen
Durch den Einsatz des cloud-init Images musste ich folgendes anpassen:

1. Ersetzen des Repos durch einen optimaleren Spiegel
2. Pakete anpassen
```
apt remove apparmor
apt install locales-all
```
3. Zeitzone anpassen
```
dpkg-reconfigure tzdata
```

## Lokaler DNS-Resolver
Um wirklich sinnvoll diverse Dienste zu verwenden, sollten DNS-Anfragen von einem eignen Server durchgeführt werden. Dafür kann man entweder selbst Resolver zentral betreiben oder einfacher direkt auf dem PMG Server.

```
apt install unbound dnsutils
```

In der resolv.conf sollte nun als nameserver nur 127.0.0.1 eingetragen werden.

Mehr dazu unter https://pmg.proxmox.com/wiki/index.php/DNS_server_on_Proxmox_Mail_Gateway

## PMG Installation Master
Zunächst muss das Repo hinterlegt werden. mehr dazu unter https://pmg.proxmox.com/pmg-docs/pmg-admin-guide.html#pmg_package_repositories

Nach einem apt update dann die Installation mit

```
apt install proxmox-mailgateway ifupdown
```

ifupdown muss hier explizite genannt werden, da sonst das cloud-init image nicht mehr geht.

Zum Schluss ein Reboot

## PMG Installation Slave
Nodes erst installieren, wenn am Master alle anderen gewünschten Optimierungen durchgeführt wurden.

Kopieren der Repos vom Master

```
scp -r root@192.168.0.1:/etc/apt/* /etc/apt/
```

nach einem apt update dann wie beim master

```
apt install proxmox-mailgateway ifupdown
```

ifupdown muss hier explizite genannt werden, da sonst das cloud-init image nicht mehr geht.

Einmal neustarten und dann dem Cluster über die interne IP beitreten.

### Postfix Anpassungen kopieren

### SpamAssassin Anpassungen kopieren

#### Channels und Beschleunigung

```
scp -r root@192.168.0.1:/etc/cron.hourly/sa-update /etc/cron.hourly/
chmod +x /etc/cron.hourly/sa-update
```

Und die GPG Keys aus der Anleitung am Master importieren!

## SpamAssassin Anpassungen

### Channels und Beschleunigung

```
apt install re2c make gcc
```

re2c make gcc wird benötigt um SpamAssassin Regeln zu optimieren, damit es schneller wird

#### Nur am Master

Erweiterung um sa-channel und die Optimierung der Regeln für die Beschleunigung.
Ein gutes Beispiel mit Anleitung für das Skript findet ihr unter https://schaal-24.de/aktuelle-regeln-fuer-spamassassin-von-schaal-it/ - Danke an Herrn Schaal

```

cd /tmp

wget http://sa.zmi.at/sa-update-german/GPG.KEY
sa-update --import GPG.KEY
rm GPG.KEY
wget https://spamassassin.apache.org/updates/GPG.KEY
sa-update --import GPG.KEY
wget https://mcgrail.com/downloads/kam.sa-channels.mcgrail.com.key
sa-update --import kam.sa-channels.mcgrail.com.key

rm /etc/mail/spamassassin/channel.d/KAM_channel.conf

touch /etc/cron.hourly/sa-update
chmod +x /etc/cron.hourly/sa-update
nano /etc/cron.hourly/sa-update
```

Ich habe folgende channels genommen:

```
#!/bin/sh

# schaal @it
#
# Simple script to update SpamAssassin

SYSLOG_TAG=sa-update

compile=0

logger -d -t $SYSLOG_TAG "Start SA-Update"

sa-update
retval="$?"
if [ $retval -eq 0 ]; then compile=1; fi

sa-update --gpgkey 24C063D8 --channel kam.sa-channels.mcgrail.com
retval="$?"
if [ $retval -eq 0 ]; then compile=1; fi

sa-update --gpgkey 40F74481 --channel sa.zmi.at
retval="$?"
if [ $retval -eq 0 ]; then compile=1; fi

if [ $compile -eq 1 ]; then
    logger -d -t $SYSLOG_TAG "SA-Update found"
    sa-compile --quiet 2>/dev/null
    systemctl restart pmg-smtp-filter
else
    logger -d -t $SYSLOG_TAG "No SA-Update found"
fi
```

Der Channel sought.rules.yerp.org wurde eingestellt. Die Channel spamassassin.heinlein-support.de und sa.schaal-it.net werden wohl nicht mehr so regelmäßig gepflegt.
Malware Patrol bietet auch noch einen kommerziellen Channel https://www.malwarepatrol.net/spamassassin-configuration-guide/
Noch mehr Infos zu Channel gibt es hier https://cwiki.apache.org/confluence...le?contentId=119542298#content/view/119542298
