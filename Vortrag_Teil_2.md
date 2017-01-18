#Was ist Software Definied Storage?
- Findet Anwendung beim Skalieren in die Breite
- SDS-Lösungen fügen eine zusätzliche Eene zwischen physischen Datenträgern und dem Frontend ein
- Zusätzliche Ebene bietet über Frontend Zugriff auf Daten und verteilt auch im Hintergrund die Daten
- SDS-Lösungen betrachten Daten als zerlegbare binäre Objekte
- Beim Auslesen werden zerlegte Objekte wiederhergestellt

- Namensgebend für SDS ist diese zusäzliche Ebene, denn:
-- Eigentliche Speicherlösung ist als Software implementiert
-- Läuft auf handelsüblicher Hardware
-- Implementierung auf Software-Ebene bietet viele Vorteile im Vergleich zu klassischen Speicherlösungen (z.B. Skalierung)

- Frontends können als unterschiedliche Typen realisiert sein
- Z.B.: Könnte en Frontend ein klassisches Blockgerät nachbilden, sodass ein System wie mit einem Blockgerät arbeitet, aber im Hintergrund läuft SDS
- Denkbar auch: Realisierung als Datenbankschnittstelle

![](https://vslapp.files.wordpress.com/2014/04/sds-principles1.png)


#Software Definied Storage am Beispiel Ceph (Ein Überblick)
![](http://docs.ceph.com/docs/master/_static/logo.png)
##Geschichte
- Erfinder Sage Weil (geboren 17.03.1978)
- Ceph entstand im Rahmen einer Doktorarbeit (~2006) an der University of Santa Cruz in Kalifornien
- Aufgabe war es für das US Department of Energy eine Storage-Lösung zu finden die Nachteile einer Höhen-Skalierung nicht besitzt
- Am Anfang der Entwicklung stand das Ziel, ein mit POSIX kompatibles Dateisystem zu erstellen
- Sage Weil gründete später das Unternehmen Inktank Storage (später aufgekauft von Red Hat)
- Wird immer noch aktiv entwickelt: CephFS 10.2 im April 2016 veröffentlicht

##Was ist Ceph?
- Ceph ist ein klassischer Objektspeicher (Daten werden in mehrere Binäreobjekte zerlegt) 
- Die Lösung hieß urpsrpünglich Rados (Reliable Autonomic Distributed Object Store)
- Ceph hingegen war der Codename des POSIX-Dateisystems (Frontend für Zugriff auf RADOS-Cluster)
- Im Kontext des Cloud-Booms entschied sich Inktank den Namen Ceph zu verwenden
- Das POSIX-Dateisystem heißt heute aus diesem Grund CephFS

##Aufbau
![](http://www.bitoss.com/wp-content/uploads/2016/08/cephImage2.png)

- Ein typischer Ceph-Cluster besteht aus zwei Diensten, die innerhalb eines Clusters beinahe beliebig oft vorkommen: OSDs und MONs 

###OSD (Object Storage Device) 
- So bezeichnet Ceph den Dienst, der physische Platten in den Clusterverbund integriert
- Die zuvor erwähnte Ebene, die verteilte Storage-Systeme zwischen die Nutzer und die physischen Geräte einfügen, stellen genau diese OSDs dar 
- Die Kommunikation mit den physischen Blockgeräten wickeln die OSD-Server im Hintergrund ab
- Auch um die inhärente Replikation kümmern sich vorrangig die OSD-Dienste.

- Dabei gilt, dass pro physischem Blockgerät ein OSD-Dienst läuft
- Auf einem Server mit 24 Festplatten finden sich also 24 laufende OSD-Instanzen
- OSDs bilden obendrein den Anlaufpunkt für Clients (Daten Schreiben/Lesen)
- Die Clients liefern Daten bei einem OSD an, das sich danach um die Replikation kümmert
- Erst wenn die eingestellten Replikationsvorgaben erfüllt sind, erhält der Client die Nachricht, dass der Schreibvorgang erfolgreich war

### Die Monitoring-Server, kurz MONs, sind in Ceph die Cluster-Wachhunde
- Sie führen Buch über vorhandene MONs und OSDs und erzwingen im Cluster ein Quorum [Verfahren zur Gewährleistung der Datenintegrität] auf der MON-Ebene
- Das ist wichtig in Situationen, in denen der Cluster in mehrere Partitionen zerfällt, etwa weil Netzwerkhardware kaputtgeht
- Die MONs stellen in solchen Szenarien sicher, dass Clients nur auf die Partition des Clusters schreibend zugreifen können, die die Mehrheit der insgesamt im Cluster bekannten MON-Server hinter sich weiß
- MON-Server sind auch der erste Anlaufpunkt für Clients und OSDs gleichermaßen, wenn diese Informationen über die aktuelle Topologie des Clusters brauchen

![](https://4.bp.blogspot.com/-yupsGmuzduA/Ur2G0gUXaXI/AAAAAAAACV4/ki853OD6ATA/s1600/ceph-arch.png)

##Datenverteilung
- Die zwischengeschaltete Schicht trägt wie beschrieben die Verantwortung dafür, dass eingehende Daten auf den physischen Speichergeräten sinnvoll verteilt abgelegt werden
- Er ist obendrein zuständig, wenn Clients spezifische Informationen über eines der Cluster-Frontends wieder auslesen wollen, denn dann läuft der Vorgang einfach in umgekehrter Reihenfolge ab.

- Woher aber erfährt ein Client, der Daten in den Cluster laden möchte, welches OSD das richtige ist? 
- Und wo bekommt jenes Primary OSD die Information her, auf welchen anderen OSDs es von den hochgeladenen Daten Replikate anlegen muss, bevor es dem Client einen erfolgreichen Schreibvorgang vermeldet?
- ->CHRUSH-Algorithmus 
- Er ist Cephs Placement-Algorithmus und ermöglicht es Clients wie OSDs, sich das jeweils passende Ziel-OSD für bestimmte Datensätze auszurechnen.

#Crush-Algorithmus
- Algorithmus in Ceph der zentrale Punkt, wenn es um die Platzierung von Daten geht
- Steht für Controlled Replication Under Scalable Hashing
- Gemeint ist das Prinzip, anhand dessen Clients oder OSDs die Ziel-OSDs für bestimmte binär Objekte festlegen

- Wenn ein Client Daten in das Cluster laden möchte, findet zuerst die Aufteilung in Objekte statt
- Für jedes der Objekte stellt sich dann die Frage, an welches OSD es zu senden ist und wohin es von dort repliziert wird

- Der Client organisiert sich zunächst von den MON-Servern des Ceph-Clusters das aktuelle Verzeichnis aller OSDs und stößt danach die Crush-Berechnung für das jeweilige Objekt an

- Crush lässt sich auch von außen beeinflussen:
-- Crush-Map bietet dem Admin die Möglichkeit, Rechner oder OSDs logisch zu gruppieren
-- Legt der Admin zum Beispiel fest, dass bestimmte Server in Rack 1 hängen und andere Server in Rack 2, so kann er im nächsten Schritt bestimmen, dass Replikate eines binären Objektes in beiden Racks vorhanden sein müssen
-- Wenn der Client oder die OSDs danach die Crush-Kalkulation für ein bestimmtes Objekt durchführen, beziehen sie jene Faktoren mit ein und erhalten ein entsprechendes Ergebnis
-- Solange sich die Topologie des Clusters nicht ändert, bleibt das Crush-Resultat identisch. 

![](http://docs.ceph.com/docs/master/_images/ditaa-1fde157d24b63e3b465d96eb6afea22078c85a90.png)

#Parallelität
- Eine der größten Stärken von Ceph ist, dass Crush-Kalkulationen sowie Objekt-Uploads oder Objekt-Downloads parallel geschehen
- Ein Client zerteilt eine Datei also in viele kleine Objekte und lädt diese anschließend parallel auf verschiedene OSDs
- Er redet stets mit vielen gleichzeitig und kombiniert so die Bandbreite beim Hoch- oder Herunterladen
- Verglichen mit klassischen Storages erreicht Ceph so bei entsprechender Netzwerkhardware enorme Durchsatzwerte

#TODO

##Frontends
- Aus Nutzersicht gestaltet sich der Umgang mit Ceph deutlich weniger komplex als beim Blick unter die Haube. 
- Für Ceph existieren aktuell drei Frontends für verschiedene Aufgaben. 
- Das Ceph Block Device greift im Hintergrund auf den Objektspeicher zu und gibt nach außen ein Blockdevice aus. 
- Der Zugriff auf ein Ceph Block Device kann auf Linux-Systemen wahlweise über das dazu passende Kernel-Modul geschehen oder im Userspace über die C-Bibliothek Librados, die etwa in der virtuellen Maschine Qemu bereits integriert ist. 
- Ein mit Ceph-Support kompiliertes Qemu kann also auf virtuelle Block-Devices in Ceph ohne den Kernel-Umweg zugreifen.

- Kein fertiges Frontend, aber doch sehr nützlich ist die bereits erwähnte C-Bibliothek Librados.
- Diese bietet auf der Programmierebene die Möglichkeit, direkt auf Ceph zuzugreifen. 
- Für Librados existieren zudem diverse Anbindungen für Skriptsprachen. 
- Mittels PHP-Rados lässt sich etwa aus PHP-Anwendungen heraus direkt auf die Inhalte des Objektspeichers zugreifen.

- Was abgedreht klingt, hat durchaus einen praktischen Nutzen. 
- Viele Webanwendungen lagern statische Inhalte wie Bilder auf POSIX-Dateisystemen, obwohl sie deren Semantik nicht benötigen. 
- Im genannten Beispiel bezieht die Webanwendung die Bilder stattdessen als Objekt direkt aus Ceph und umgeht so Hilfskonstrukte auf Basis von NFS (Network File System) oder ähnlichen Lösungen. 
- Vergleichbare Lösungen mit Amazons Simple Storage Service (S3) haben sich mittlerweile etabliert.

- Auf Librados basiert auch das Ceph Object Gateway. Dieses exponiert nach außen das Amazon-S3-Protokoll und das Openstack-Swift-Protokoll und bietet so Zugriff auf Ceph per HTTP-basierter Restful-API. 
- Admins können Nutzern also über das Ceph Object Gateway einen objektbasierten Online-Speicherdienst im Stile von Dropbox, Amazon S3 oder Google Drive anbieten.

- Nicht zu vergessen ist freilich auch das bereits erwähnte CephFS. Erst vor ein paar Monaten erreichte dieses die Reife für den produktiven Einsatz und war damit ironischerweise das letzte der drei beschriebenen Frontends - und das, obwohl es eigentlich die Keimzelle von Ceph darstellt, dessen Entwicklung vor mehr als zehn Jahren begonnen hat.

#Zusammenfassung
- Ceph ist ein in die Breite skalierendes SDS
- Für Cluster Computing ausgelegt
- Setzt auf bestehendes FS (z.B.b BRFTS) auf
- Zerlegt Daten in mehrere Binärobjekte und verteilt sie auf verschiedene Blockgeräte
- Kann hohe Performance durch Parallelität erreichen
- Zugriff per Frontends
-


#Weiterführende Literatur
1. Weil, Sage; Leung, Andrew; Brandt, Scott A.; Maltzahn, Carlos (2007), "RADOS: A Fast, Scalable, and Reliable Storage Service for Petabyte-scale Storage Clusters"

2. Weil, Sage; Brandt, Scott A.; Miller, Ethan L.; Maltzahn, Carlos (2006), "CRUSH: Controlled, Scalable, Decentralized Placement of Replicated Data"

3. Weil, Sage; Brandt, Scott A.; Miller, Ethan L.; Maltzahn, Carlos; Long, Darrell D. E. (2006), "Ceph: A Scalable, High-Performance Distributed File System"

4. Weil, Sage; Wang, Feng; Xin, Qin; Brandt, Scott A.; Miller, Ethen L.; Long, Darrell D. E. (2006), "Ceph: A Scalable Object-Based Storage System"

5. Weil, Sage; Pollack, Kristal; Brandt, Scott A.; Miller, Ethan L. (2004), "Dynamic Metadata Management for Petabyte-Scale File Systems"

6. Weil, Sage; Brandt, Scott A.; Miller, Ethan L. (2004), "Intelligent Metadata Management for a Petabyte-Scale File System"

#Quellen
1. https://en.wikipedia.org/wiki/Sage_Weil
2. https://de.wikipedia.org/wiki/Ceph
3. Autor: Martin Loschwitz/Syseleven, http://www.golem.de/news/cloud-computing-was-ist-eigentlich-software-defined-storage-1610-122478.html
4. http://docs.ceph.com/docs/master/cephfs/
5. http://www.bitoss.com/valueaddedservices/ceph/
