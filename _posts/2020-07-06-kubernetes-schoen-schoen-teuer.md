---
layout: post
tag: de
title: "Kubernetes schoen, schoen teuer"
subtitle: Nachdem man sich schon etwas mit Kubernetes vertraut gemacht hat, fallen einem vielleicht schon sinnvolle Anwendungsbeispiele ein. Zeit sich einen Kubernetes Cluster zu beschaffen, gern als "Managed Kubernetes", wie das Angebot heutzutage heisst. Als Privatanwender muss man aber schon recht tief in die Tasche greifen - im Vergleich zum V-Server oder Root-Server
date: 2020-07-06
background: '/images/k8s-cosmos.png'
---

Auf <a href="http://webhostlist.de">Webhostlist.de</a> findet man zahlreiche Angebote an V-Server und Root-Servern. Fuer 5 Euro im Monat bekommt man schon eine kleine VM, etwa als ssh-Jumphost mit fester IP-Adresse oder VPN-Gateway fuer den Privatgebrauch.
Ab 20-30 Euro gibts "Bare-Metal", also Server aus blanken Metall, mit dem man schon einiges anfangen kann: Mailserver, Webserver usw.

Wie siehts jetzt mit Kubernetes aus? Bei den grossen Drei ist die Verwaltung der Master-Knoten unter Umstaenden kostenlos. Damit kann man aber noch nicht viel anfangen, man brauch mindestens einen Worker-Knoten und wahrscheinlich noch etwas Festplatte. 
<strong>Google-Cloud</strong>

Ein Kubernetes Cluster mit einem Knoten und 10GB Speicher kostet 89,99 Euro, wenn er den ganzen Monat durchlaeuft. Mit einem zweiten Knoten steigt der Preis auf 155,15 Euro

<strong>AWS</strong>

Die Kubernetes Controlplane kostet bei Amazon 0,10 pro Stunde - macht schon mal 72 Euro im Monat. Ein Worker-Knoten wird nach den normalen EC2-Instanz-Preislisten berechnet. Macht etwa nochmal 72 Euro + Speicher + Netzwerk.

<strong>Azure</strong>

Bei Azure ist die Cluster-Verwaltung tatsaechlich kostenlos. Natuerlich sind die Worker-Knoten dadurch etwas teurer. Der Preisrechner schlaegt hier einen Knoten fuer 86 Euro im Monat vor, das Speicherkonto bis 32 GB ist mit 1,54 Euro vergleichsweise guenstig.

Alles andere aber nicht!

Andere Anbieter:

<strong>Netways</strong> machte schon oefters mit Angeboten zu OpenStack basierten Hosting auf sich aufmerksam. Es gibt ein Kubernetes-Bundle-Angebot fuer 152 Euro. Dort ist zum Durchstarten schon mal alles drin: Worker-Knoten, 2 Loadbalancer, 4 Floating-IPs

<strong>OVH</strong> bietet einen kostenfreien Managed Kubernetes Service an. Der kleinste Worker-Knoten kostet 53 Euro, 10 GB Speicher 0,47 Euro, ein Loadbalancer nochmal 11,44 Euro - bis jetzt das guenstigste Angebot, wenn es beim Speicher keinen Druckfehler gegeben hat.

<strong>Ionos</strong> wirbt auch mit kostenfreien Managed Kubenertes Service. Bei den Worker-Knoten wird auf die Compute-Preise verwiesen. Diese starten bei 14,40 Euro im Monat, da ist allerdings weder RAM noch Festplatte drin. Der RAM ist relativ teuer: 3,24 Euro pro GB im Monat. Da ist man schon bei 51 Euro, wenn man das zu anderen Angeboten mit 16 GB RAM vergleichen will. 10 GB Speicher starten bei 0,40 Euro im Monat.

Schon bei diesen 6 Angeboten hat man eine ungefaehre Vorstellung, wohin die Reise geht. Der Anbieter betreibt ein Kubernetes oder stellt dem Kunden ein Framework zur Verfuegung, an das er dann weitere IAAS-Produkte des Anbieters anflanschen kann. Hier waere vielleicht zu ueberlegen, ob man sich nicht normale V- oder Root-Server als Worker-Knoten zum Managed Kubernetes dazustellen kann - immerhin funktioniert die Kommunikation ueber API. In Summe kommt das aber alles etwas teuer, weswegen Kubernetes auf diesem Weg fuer den Privatanwender unattraktiv bleiben wird.

