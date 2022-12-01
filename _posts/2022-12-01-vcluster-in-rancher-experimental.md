---
layout: post
tag: de
title: "vcluster in Rancher Experimente"
subtitle: "Kubernetes gibt es in verschiedenen Ausprägungen. Als einzelner Node, bis zu tausenden. Als einziger Nutzer, bis zu tausenden. Das Problem ist die Trennung verschiedener Nutzer, ohne Beschränkung der Clusterrechte. Wie das geht zeigt das Experiment".
date: 2022-12-01
background: '/images/k8s-cosmos.png'
twitter: '/images/2022-05-27-1.png'
author: eumel8
---

<a href="https://www.vcluster.com">Vcluster</a> ist ein voll funktionsfähiger Kubernetes-Cluster, installiert in einem Kubernetes-Cluster.
Während der Nutzer in einem Namespace oder über ein Rancher Projekt in mehrere Namespaces Zugriff hat und ihm cluster-weite Resourcen
 verwehrt bleiben und so etwa keine CRDs oder Mutation-Webhooks installiert werden können. soll er im vcluster vollen Zugriff erhalten.
Wie weit das wirklich funktioniert und welche Einschränkungen es gibt, schauen wir uns hier mal an.

# Vorbedingungen

Was sofort funktioniert:

* <a href="https://k3s.io/">K3S</a> auf einer Ubuntu 20.04 VM installieren.
* vcluster cli von der vcluster Webseite herunterladen
* vcluster installieren mit `vcluster create my-vcluster`

Wir wollen allerdings eine etwas sichere Methode wählen, die auch mandatenfähig ist. Zur Verwaltung mandatenfähiger Cluster mit
zentraler Userverwaltung hat sich <a href="https://rancher.io">Rancher</a> etabliert.
Wir brauchen also:

* Existierenden Rancher Upstream-Cluster mit Rancher 2.6+
* Existierenden Rancher Downstream-Cluster mit mindestens einem Worker-Node
* Ein neues Projekt im Downstream-Cluster mit PodSecurityPolicy "restricted"
* Einen Namespace in diesem Projekt (vcl1)

# Installation

In diesem Namespace können wir vcluster installieren. Neben der Installation mit dem vcluster cli gibt es eine Installationsvariante
mit Helm. Hier ist unser values.yaml

```yaml
vcluster:
  image: rancher/k3s:v1.23.5-k3s1
securityContext:
  runAsUser: 1000
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
   drop:
   - all
  readOnlyRootFilesystem: true
fsGroup: 1000
```

Den vollen Konfigurationsumfang des Charts kann man <a href="https://github.com/loft-sh/vcluster/blob/main/charts/k3s/values.yaml">hier</a> sehen.
Zu beachten ist noch, dass vcluster verschiedene Cluster-Typen kennt: eks, k8s, k0s, k3s. Wir benutzen k3s.

Jetzt die Installation:

```bash
helm -n vcl1 upgrade -i cl1 vcluster --values values.yaml --repo https://charts.loft.sh
```

Überprüfung:

```bash
$ vcluster list

 NAME   NAMESPACE   STATUS    CONNECTED   CREATED                         AGE        CONTEXT            
 cl1    vcl1        Running               2022-12-01 15:21:35 +0100 CET   28s   frank-test-k8s-12
```

Das wars schon. 

# Import

Wir können den vcluster in Rancher importieren, indem wir die Importfunktion im Rancher benutzen. Es wird eine Deploy-URL generiert,
die wir im vcluster ausführen:

```bash
vcluster -n vcl1 connect cl2  -- kubectl apply -f https://k3s.otc.mcsps.de/v3/import/hjjnrlmj9xfsmw7wctsdpr7b6ftc5gkfr9fqv7rlvshzzj86f46q_c-m-84h8jjd7.yaml
```

Hier taucht schon das erste Problem auf. In einem Downstream-Cluster mit verschiedenen Projekten und Namespaces kann ich verschiedene
Sicherheitsrichtlinien festlegen. Systemnahe Dienste wie die Rancher-Agenten laufen im System-Projekt und dem cattle-system Namespace.
Dort können sie sich in aller Ruhe ausbreiten, ohne die Restriktionen eines Projekts mit einer Nutzer-Webapp.

Der vcluster läuft in einem Namespace des Downstream-Clusters und dort mit der festgelegten Restriktionen wie Pod Security Policy
oder Resource Quota. Man muss also entscheiden, ob dieser Cluster dann vollständig in diesem Modus laufen kann oder
etwa eine unresticted Pod Security Policy benötigt.
Für den Rancher Agenten können wir uns behelfen, indem wir im Deployment die Affinity löschen und den securityContext anpassen.

```yaml
spec:
  template:
    spec:
      container:
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
      securityContext:
        fsGroup: 1000
```

Dann haben wir den vcluster im Rancher:

<img src="/blog/images/2022-12-01-1.png" width="900" height="450" />

<img src="/blog/images/2022-12-01-2.png" width="900" height="450" />

<img src="/blog/images/2022-12-01-3.png" width="900" height="450" />

<img src="/blog/images/2022-12-01-4.png" width="900" height="450" />

<img src="/blog/images/2022-12-01-5.png" width="900" height="450" />


Happy vclustering
