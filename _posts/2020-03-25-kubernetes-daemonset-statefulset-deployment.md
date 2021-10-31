---
layout: post
tag: de
title: "Kubernetes: DaemonSet, StatefulSet, Deployment"
subtitle: "Neues Jahr, neuer Kubernetes Blogeintrag. Heute wollen wir gleich zwei nuetzliche Sache besprechen. Zum einen lernen wir 3 weitere wichtige Bestandteile von Kubernetes kennen. Zum anderen ist auch gleich ein praktisches Beispiel dabei, was gerade jetzt in der COVID-19 Krise extrem wichtig ist: Foding@Home."
date: 2020-03-25
background: '/images/k8s-cosmos.png'
---

<strong>Kubernetes</strong>

<strong>DaemonSet</strong>

Das Kubernetes DaemonSet stellt sicher, dass (sofern nicht per NodeSelector aussortiert) auf jedem Node ein POD des Daemonsets laeuft. Das kann zum Beispiel sinnvoll sein, wenn man direkt auf Resourcen des Nodes zugreifen moechte oder Informationen von alllen Nodes einsammeln moechte. Das sind natuerlich Dienste fuer Monitorring und Logging (also Prometheus und Fluentd) oder Speicherdienste wie GlusterFS. Sinnvoll ist ein DaemonSet auch, wenn alle Nodes gleich stark belastet werden sollen, ohne grosse LoadBalancing-Logik zu verwenden.

<strong>StatefulSet</strong>

Stafeful, wie der Name schon sagt, das Gegenteil von zustandslos. Man will immer ueber den Zustand eines PODs Bescheid wissen oder diesen exakt ansprechen wollen. Dazu hat der POD einen festen Namen, den man auch ueber einen Selector ansprechen kann. Praktische Anwendungen sind Datenbank-Cluster wie MySQL oder MongoDB, wo ich immer einen bestimmten Knoten ansprechen muss, der dann auf einem StateFul POD laeuft. Oder ich will den POD von aussen ansprechen, ohne einen NodePort zu verwenden. Das sehen wir dann noch unten.

<strong>Deployment</strong>

Praktisch das Gegenteil von Stateful und der Normalzustand in der cloud-nativen Welt. PODs haben einen zufaelligen Namen, werden auch ueber Selectoren oder Services angesprochen und es kuemmert einen nicht, wenn ein POD nicht verfuegbar ist, da dieser keinen festen Zustand enthaelt und nur zur Datenverarbeitung eingerichtet ist. Das ist zum Beispiel ein Webserver wie Apache oder NGINX, der nur Webseiten ausliefern soll, den Content ueber ein gemountetes Volume bereitgestellt bekommt, die Logfiles an den Fluentd abgibt und ansonsten nur die Requests der User von aussen bedient. Kernfunktion ist ein Container mit einem Image, welches ein Programm mit gewissen Parameter startet, die ueber Environment-Variablen, Secrets oder ConfigMaps bereitgestellt werden. Dazu gibt es Probes, die sicherstellen, dass der Container betriebsbereit ist, und gemountete Volumes, die zum Betrieb benoetigt werden, Wir haben das schon besprochen. Im Deployment werden all diese Informationen zusammengefasst und ueber <strong>ReplicaSet</strong> gesteuert, wie oft der POD gestartet werden soll. Der <strong>ReolicaController</strong> stellt dann sicher, dass die PODs auf mehrere Nodes verteilt werden und auch immer ausreichend viele PODs gleichzeitig laufen. Stichwort: RollingUpgrades - wenn eine neue Version des Deployments ausgerollt wird, wird immer ein POD nach dem anderen gekillt und mit der neuen Version gestartet.

<strong>Folding@Home</strong>

Das Projekt Folding@Home gibt es schon seit 10 Jahren und befasst sich mit Simulation der Proteinfaltung und der Bewegung von Proteinen, die an der Entstehung von vielen Krankheiten beteiligt sind. So wird zum Beispiel an Krebs geforscht, Alzheimer, Parkinson und nun auch an <a href="https://foldingathome.org/covid19/">COVID-19</a>. 
[video:youtube:Ybrl6b6A7Ls]
Zur Simulation wird sehr viel Rechenleistung benoetigt. Um jetzt keinen Super-Computer ausschliesslich mit diesem Projekt auszulasten, wurden die Simulationsberechnungen in kleine Pakete aufgeteilt, die mittels Grid-Computing in vielen kleineren Computern zu berechnen ist. 
Dazu wird ein Clientprogramm benoetigt, das aus einer Warteschlange ein Simulationspaket entgegennimmt, daran rumrechnet und das Ergebnis nach einiger Zeit dem Projekt zurueckuebermittelt. 
Populaer wurde das Projekt Anfang Maerz 2020, also die Firma <a href="https://twitter.com/NVIDIAGeForce/status/1238496311776653312">NVIDIA die Gamer-Community aufrief</a>, mit ihren leistungsstarken GPU-Maschinen sich an diesem Projekt zu beteoligen. Die Resonanz war so gross, dass die Warteschlange tagelang leer war und die Forscher erstmal neue Simulationen entwickeln mussten.
Fokusiert ist man jetzt natuerlich auf COVID-19. Wer Rechenleistung zur Verfuegung stellen moechte, findet auf der <a href="https://foldingathome.org/start-folding/">Projektseite </a> Clients fuer Windows oder Android. Man kann in den Clients noch einstellen, zu welchem Krankheitsprojekt genau man beitragen moechte (oder die Rechenleistung wird automatisch geroutet) und wieviel Nutzlast man auf seinen Computer geben moechte (wenig fuer Hintergrundberechnung oder viel fuer volle Rechenpower). 

<strong>Fuer Kubernetes gibt es 3 Implemtierungen </strong>, als 
<a href="https://github.com/wind0r/k8s-supporting-folding-at-home">DaemonSet</a> - Volle Power auf allen Nodes wie oben beschrieben.
<a href="https://github.com/eumel8/k8s-supporting-folding-at-home">StatefullSet</a>  - Hier haben die PODs feste Namen und koennen ueber einen Ingress-Service einzeln von aussen angesprochen werden. Es ist ein LoadBalancer Service und <a href="https://github.com/eumel8/k8s-supporting-folding-at-home">external-dns</a> erforderlich.<a href="https://codeberg.org/hjacobs/folding-at-home-on-kubernetes/src/branch/master/README.md">
Deployment</a> - Hier laesst sich fein granular die Anzahl der PODs ueber das ReplicaSet einstellen. Ueber CPU Resource Limits kann die Last des PODs ausserdem noch begrenzt werden.

Die Projekte sind entweder als Helm Chart zu installieren oder einfach mit <code>kubectl apply -f ...</code> zu installieren.
Aber nicht vergessen, die <a href="https://stats.foldingathome.org/team/251739">TeamID</a> und den User zu aendern, ansonsten werden die Berechnungen nur anonym gezaehlt. Es gibt naemlich <a href="https://foldingathome.org/statistics/">zahlreiche Statistiken</a>  zum Projekt.  Happy Folding!
