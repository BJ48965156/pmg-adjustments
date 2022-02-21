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

#### Erweiterungen

Falls Spamhaus Subscription existiert

```
scp -r root@192.168.0.1:/etc/mail/spamassassin/SH.pm /etc/mail/spamassassin/
```

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

### Erweiterungen

```
apt install pyzor
cd /tmp
wget https://www.dcc-servers.net/dcc/source/dcc.tar.Z
tar xfvz dcc.tar.Z
cd dccXXX
./configure && make && make install
```

Infos zu pyzor unter https://www.pyzor.org/en/latest/
Infos zu DCC unter https://www.dcc-servers.net/dcc/

Bitte Bedingungen und Lizenzen berücksichtigen.

#### Nur am Master

Nicht gewünschte Module einfach aus der custom.cf entfernen. Ein Modul beginnt bei dem Kommentar und endet vor dem nächsten Kommentar. Bei dem Spamhaus Plugin muss jeder Eintrag mit PRIVATEKEY noch durch den subscription Key ersetzt werden.

Datei /etc/mail/spamassassin/custom.cf siehe Anhang custom.cf.txt
Datei /etc/pmg/templates/v320.pre.in siehe Anhang v320.pre.in.txt
Datei /etc/pmg/templates/local.cf.in siehe Anhang local.cf.in.txt

Zur local.cf.in:
Alles unter dem Kommentar # central Bayes store bis zum [% END %] ist nur nötig, wenn die gleiche Bayes Datenbank auf allen Servern genutzt werden soll

Pyzor und DCC siehe oben
OLEVBMacro siehe https://spamassassin.apache.org/full/3.4.x/doc/Mail_SpamAssassin_Plugin_OLEVBMacro.txt
FromNameSpoof siehe https://spamassassin.apache.org/full/3.4.x/doc/Mail_SpamAssassin_Plugin_FromNameSpoof.txt
Spamhaus siehe https://github.com/spamhaus/spamassassin-dqs
DMARC siehe https://notes.sagredo.eu/setting-dmarc-filter-in-spamassassin-236.html und https://random.sphere.ro/2019/08/12/dmarc-on-spamassassin/
Mailspike.net siehe https://mailspike.org/usage.html

die SH.pm von https://raw.githubusercontent.com/spamhaus/spamassassin-dqs/master/SH.pm muss im Ordner /etc/mail/spamassassin/ abgelegt werden, wenn Spamhaus Subscription existiert

### Geteilte Bayes Datenbank

Wer schon eine Bayes-Datenbank gepflegt hat, sollte diese vorher sichern mit

```
sa-learn --backup > /root/bayes.db
```

####Auf allen Servern:

```
apt install libdbd-mysql-perl mariadb-server libtie-cache-perl libdbd-mysql-perl
mysql_secure_installation
nano /etc/mysql/conf.d/galera.cnf
```

Dort dann, angepasst an den entsprechenden Node (!) folgendes eintragen:

```
[mysqld]
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
bind-address=192.168.0.1
# Galera Provider Configuration
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so
# Galera Cluster Configuration
wsrep_cluster_name="galera_cluster"
wsrep_cluster_address="gcomm://192.168.0.1,192.168.0.2,192.168.0.3"

# Galera Synchronization Configuration
wsrep_sst_method=rsync

# Galera Node Configuration IP und Name des Nodes auf dem diese Datei liegt!
wsrep_node_address="SERVER-IP-INTERN"
wsrep_node_name="SERVERNAME"
```

Dann einmal mysql neustarten.

```
systemctl restart mysql
```

Ob es erfolgreich war kann man wie folgt prüfen:

```
mysql -u root -p -e "show status like 'wsrep_cluster_size'"
```

Es sollte dann bei 3 Server folgendes angezeigt werden:

```
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 3     |
+--------------------+-------+
```

Bei mehr Server entsprechend dann anstatt 3 die Anzahl der Server.

####Nur auf auf einem Server

(z.B. master) dann noch den spamassassin Benutzer und die Datenbank anlegen:

```
mysqladmin -p create sa_bayes -u root
mysql -p
grant all privileges on sa_bayes.* to sa_user@localhost identified by 'SUPERGEHEIM';
use sa_bayes;
```

Die mysqlcli noch nicht beenden, sondern noch die benötigten Tabellen erstellen. Leider habe ich auf dem Server die Datei bayes_mysql.sql nicht gefunden. Daher habe ich die von hier genommen: https://fossies.org/linux/misc/Mail-SpamAssassin-3.4.6.tar.bz2/Mail-SpamAssassin-3.4.6/sql/bayes_mysql.sql?m=t

Jetzt kann der mysqlcli beendet und die vielleicht vorher gesicherte Bayes-Datenbank geladen werden:

Zuvor muss die SpamAssassin Konfiguration auf die MySQL-Datenbak zeigen. Wie die aussehen muss steht unter "SpamAssassin Erweiterungen"

```
sa-learn --restore /root/bayes.db
```

Am besten über die Console ausführen, je nach Datenbestand dauert das einige Stunden.

Wenn autolearn aktiv ist (müsste standardmäßig sein), werden alle Mails mit Score < 0.1 als ham und Score > 12 als spam eingelernt. Zu der Bedingung Score >12 gilt sich noch, dass mindestens 3 Punkte aus dem Header kommen müssen und 3 Punkte aus dem Body. Also nicht wundern wenn mal mails einen Score von 20 haben aber nicht als spam gelernt wurden. Dann ist meist der Score vom Body < 3.

EDIT: Um die lokale Verbindung zum MariaDB Server zu beschleunigen, kann in der local.cf.in (siehe /etc/pmg/templates/) direkt die .sock file angegeben werden. Dafür den Eintrag bayes_sql_dsn anpassen:

```
bayes_sql_dsn       DBI:mysql:database=sa_bayes;mysql_socket=/run/mysqld/mysqld.sock
```

### Geteilte AWL Datenbank
Ich gehe davon aus, das das MariaDB-Cluster eingerichtet ist. Ansonsten einmal unter "Geteilte Bayes Datenbank" nachschauen. Am master Server muss noch der Event Scheduler aktiviert werden. Dafür in der Datei /etc/mysql/mariadb.conf.d/50-server.cnf 

```
event_scheduler=ON
```

in der Sektion [mysqld] ergänzen und dann den MySQL Server neustarten.

Dann einmal den mysql-client starten und folgende Befehle ausführen:

```
CREATE TABLE awl (
  username varchar(100) NOT NULL default '',
  email varchar(255) NOT NULL default '',
  ip varchar(40) NOT NULL default '',
  msgcount int(11) NOT NULL default '0',
  totscore float NOT NULL default '0',
  signedby varchar(255) NOT NULL default '',
  PRIMARY KEY (username,email,signedby,ip)
) ENGINE=InnoDB;;
```

Die Tabelle stammt von hier https://svn.apache.org/repos/asf/spamassassin/trunk/sql/README.awl allerdings musste ich die Datenbankengine anpassen. MyISAM funktioniert im Cluster nicht. Weiterhin löscht spamassassin wohl nicht automatisch aus der Tabelle. Darum müssen wir diese erweitern um später mit Hilfe von Events die Tabelle zu bereinigen.

Erweitert die Tabelle um einen Zeitstempel, der automatisch aktualisiert wird bei jedem Update: 

Code:
ALTER TABLE awl ADD lastupdate timestamp default CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP;

Ergänze zwei Events um die Tabelle zu bereinigen. Zum einen werden alle Einträge gelöscht, die seid 6 Monaten nicht aktualisiert wurden und zum anderen werden alle Einträge gelöscht, wo vom Absender bisher nur eine E-Mail gesendet wurde und der Eintrag älter als 1 Monat ist.

```
CREATE EVENT awl_clean
ON SCHEDULE EVERY 1 DAY
STARTS '2022-01-18 00:00:00'
DO
DELETE FROM awl WHERE lastupdate <= DATE_SUB(SYSDATE(), INTERVAL 6 MONTH);
CREATE EVENT awl_onemail_clean
ON SCHEDULE EVERY 1 DAY
STARTS '2022-01-18 00:00:00'
DO
DELETE FROM awl WHERE msgcount = 1 AND lastupdate <= DATE_SUB(SYSDATE(), INTERVAL 1 MONTH);
```

Danach muss die Datei local.cf.in (siehe /etc/pmg/templates/) vor dem include erweitert werden mit:

```
#AWL Cluster
auto_whitelist_factory Mail::SpamAssassin::SQLBasedAddrList
user_awl_dsn                 DBI:mysql:database=sa_bayes;mysql_socket=/run/mysqld/mysqld.sock
user_awl_sql_username        sa_user
user_awl_sql_password        SUPERGEHEIM
```

## PMG

### E-Mails blocken und in Quarantäne speichern
Bei der vorherigen Eingesetzen Lösung wurden auch E-Mails, die in der Quarantäne abgelegt wurden, vom Mailserver abgelehnt. Ich persönlich finde die Lösung sinnvoll und habe daher eine Anpassung vorgenommen. Im Tracking Center werden dann immer E-Mails 2x angezeigt, einmal mit Quarantäne und einmal mit Block. des Weiteren sollte das System soweit optimiert sein, dass E-Mails schnell verarbeitet werden. Zu lange SMTP-Transaktionen können Probleme verursachen.

1. Before Queue Filter aktivieren
2. Mail Filter Rules anpassen. Jede Regel mit Aktion "Quarantäne" muss kopiert werden mit geringere Priorität und "Block" als Aktion.
3. Auf allen Servern /usr/share/perl5/PMG/RuleDB/Quarantine.pm anpassen.

Ändern

```
sub final {
    return 1;
}
```

Zu

```
sub final {
    return 0;
}
```

Meh dazu unter https://forum.proxmox.com/threads/before-queue-filtering-block-quarantine-save-a-copy-of-blocked-email.98846/#post-444072

### Tracking Center - Alle Logs zentral

Nur auf dem Master. Dann kann man dort im WebGUI E-Mails aus allen Nodes sehen.

```
mv /usr/bin/pmg-log-tracker /usr/bin/pmg-log-tracker-local
touch /usr/bin/pmg-log-tracker
chmod +x /usr/bin/pmg-log-tracker
nano /usr/bin/pmg-log-tracker
```

Und da dann

```
/usr/bin/pmg-log-tracker-local "$@" | head -n -1
ssh -o ConnectTimeout=2 -o ConnectionAttempts=1 root@192.168.0.2 /usr/bin/pmg-log-tracker "$@" | sed '/^#/ d'
ssh -o ConnectTimeout=2 -o ConnectionAttempts=1 root@192.168.0.3 /usr/bin/pmg-log-tracker "$@" | sed '/^#/ d'
```

Einfügen. Danke an Sommer https://forum.proxmox.com/threads/tracking-center-not-in-sync.41903/#post-216707 und JoshuaSign https://forum.proxmox.com/threads/tracking-center-not-in-sync.41903/page-2#post-281834

## ClamAV

ClamAV wird um Signaturen erweitert mit Hilfe einen Skriptes, welches die Signaturen herunterlädt. Mehr dazu unter https://github.com/extremeshok/clamav-unofficial-sigs

####Auf allen Servern:

```
apt-get purge -y clamav-unofficial-sigs
mkdir -p /usr/local/sbin/
wget https://raw.githubusercontent.com/extremeshok/clamav-unofficial-sigs/master/clamav-unofficial-sigs.sh -O /usr/local/sbin/clamav-unofficial-sigs.sh && chmod 755 /usr/local/sbin/clamav-unofficial-sigs.sh
mkdir -p /etc/clamav-unofficial-sigs/
```

####Auf dem Master:

Die 3 Konfigurationsdateien (master.conf os.conf user.conf) wie im Link beschrieben herunterladen und ggf. anpassen. Auch hier kann man Subscriptions hinterlegen für die diversen Anbieter.

####Auf den Nodes:

```
scp -r root@192.168.0.1:/etc/clamav-unofficial-sigs/* /etc/clamav-unofficial-sigs/
```

####Wieder auf allen:

```
/usr/local/sbin/clamav-unofficial-sigs.sh --force
/usr/local/sbin/clamav-unofficial-sigs.sh --install-logrotate
/usr/local/sbin/clamav-unofficial-sigs.sh --install-cron
```

## Postfix

Leider ist die RegEx Prüfung für DNSBL Seiten recht restriktiv im WebGUI und postscreen kann auch nur IPs gegen DNSBLs prüfen. Daher passe ich das Template /etc/pmg/templates/main.cf.in an indem ich smtpd_sender_restrictions erweiter und rbl_reply_maps hinzufüge um meine privaten Schlüssel zu maskieren:

```
smtpd_sender_restrictions =
        permit_mynetworks
        reject_non_fqdn_sender
        check_client_access     cidr:/etc/postfix/clientaccess
        check_sender_access     regexp:/etc/postfix/senderaccess
        check_recipient_access  regexp:/etc/postfix/rcptaccess
[%- IF pmg.mail.rejectunknown %] reject_unknown_client_hostname[% END %]
[%- IF pmg.mail.rejectunknownsender %] reject_unknown_sender_domain[% END %]
        reject_rhsbl_sender         PRIVATEKEY.dbl.dq.spamhaus.net=127.0.1.[2..99]
        reject_rhsbl_helo           PRIVATEKEY.dbl.dq.spamhaus.net=127.0.1.[2..99]
        reject_rhsbl_reverse_client PRIVATEKEY.dbl.dq.spamhaus.net=127.0.1.[2..99]
        reject_rhsbl_sender         PRIVATEKEY.zrd.dq.spamhaus.net=127.0.2.[2..24]
        reject_rhsbl_helo           PRIVATEKEY.zrd.dq.spamhaus.net=127.0.2.[2..24]
        reject_rhsbl_reverse_client PRIVATEKEY.zrd.dq.spamhaus.net=127.0.2.[2..24]
        reject_rhsbl_client         PRIVATEKEY.dblack.mail.abusix.zone
        reject_rhsbl_helo           PRIVATEKEY.dblack.mail.abusix.zone
        reject_rhsbl_sender         PRIVATEKEY.dblack.mail.abusix.zone
        reject_rhsbl_sender         dbl.abuse.ro
        permit_dnswl_client         list.dnswl.org=127.0.[0..255].[1..3]
        permit_dnswl_client         PRIVATEKEY.white.mail.abusix.zone
        permit_dnswl_client         ips.whitelisted.org
        permit_dnswl_client         wl.mailspike.net=127.0.0.[18..20]
        reject_rbl_client           PRIVATEKEY.zen.dq.spamhaus.net=127.0.0.[2..255]
        reject_rbl_client           PRIVATEKEY.combined.mail.abusix.zone
        reject_rbl_client           b.barracudacentral.org
        reject_rbl_client           dnsbl-1.uceprotect.net
        reject_rbl_client           rbl.abuse.ro
        reject_rbl_client           bl.0spam.org
        reject_rbl_client           bl.mailspike.net

rbl_reply_maps = texthash:/etc/postfix/rbl-reply-map
```

Als erstes efolgen RHSBL checks. Danach werden IPs von Whitelits zugelassen. Zum Schluss wird gegen RBL geprüft. Sender auf Whitelisten sollten nicht auf MTA Level geblockt werden, da sie in der Regel kein SPAM versenden. Die komemrziellen Listen die ich zuerst verwende sollten normalerweise Sorge tragen, dass whitelisted systeme dort nicht auftauchen.

rbl-reply-map:

```
PRIVATEKEY.sbl.dq.spamhaus.net=127.0.0.[2..255]      $rbl_code Service unavailable; $rbl_class [$rbl_what] blocked using sbl.spamhaus.org${rbl_reason?; $rbl_reason}
PRIVATEKEY.xbl.dq.spamhaus.net=127.0.0.[2..255]      $rbl_code Service unavailable; $rbl_class [$rbl_what] blocked using xbl.spamhaus.org${rbl_reason?; $rbl_reason}
PRIVATEKEY.pbl.dq.spamhaus.net=127.0.0.[2..255]      $rbl_code Service unavailable; $rbl_class [$rbl_what] blocked using pbl.spamhaus.org${rbl_reason?; $rbl_reason}
PRIVATEKEY.sbl-xbl.dq.spamhaus.net=127.0.0.[2..255]  $rbl_code Service unavailable; $rbl_class [$rbl_what] blocked using sbl-xbl.spamhaus.org${rbl_reason?; $rbl_reason}
PRIVATEKEY.zen.dq.spamhaus.net=127.0.0.[2..255]      $rbl_code Service unavailable; $rbl_class [$rbl_what] blocked using zen.spamhaus.org${rbl_reason?; $rbl_reason}
PRIVATEKEY.dbl.dq.spamhaus.net=127.0.1.[2..99]       $rbl_code Service unavailable; $rbl_class [$rbl_what] blocked using dbl.spamhaus.org${rbl_reason?; $rbl_reason}
PRIVATEKEY.zrd.dq.spamhaus.net=127.0.2.[2..24]       $rbl_code Service unavailable; $rbl_class [$rbl_what] blocked using zrd.spamhaus.org${rbl_reason?; $rbl_reason}
PRIVATEKEY.combined.mail.abusix.zone           $rbl_code Service unavailable; $rbl_class [$rbl_what] blocked using Abusix Mail Intelligence${rbl_reason?; $rbl_reason}
PRIVATEKEY.dblack.mail.abusix.zone             $rbl_code Service unavailable; $rbl_class [$rbl_what] blocked using Abusix Mail Intelligence${rbl_reason?; $rbl_reason}
