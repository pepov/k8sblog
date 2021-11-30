---
layout: post
tag: de
title: Multi Kubernetes Cluster Foederation mit KubeFed
subtitle: "Was kann es besseres geben als einen Kubernetes Cluster? Richtig: ZWEI Kubernetes Cluster! Optimalerweise sollten diese sich miteinander unterhalten koennen und man sollte Arbeitslast leicht von einem Cluster auf einen anderen verschieben koennen. Die Antwort dazu heisst: KubeFed"
date: 2020-08-04
background: '/images/k8s-cosmos.png'
---

Die Frage stellte sich unlaengst bei der Evakuiering eines leicht ledierten Clusters, der aus dem <a href="https://rancher.com/docs/rancher/v2.x/en/backups/restorations/k3s-restoration/">Backup-Recovery</a> wiederauferstanden war und doch nicht wieder richtig lief. Die Idee war, die Arbeitslast wie Deployments, Services und Configmaps schnell auf einen anderen Cluster zu migrieren. Die Foederation der beiden Cluster wird mit KubeFed realisiert.

<strong>Wir brauchen:</strong>

<ul>
  <li>Zwei (1-Node) Kubernetes Cluster im selben Netzwerk, mindestens freier Zugang zur KubeAPI Port tcp/6443</li>
  <li>Ein Kube-Config-File, welches jeweils <a href="https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/">mit Kontext zu beiden Clustern</a> konfiguriert ist.  </li>

Desweiteren
  <li>kubectl (etwa <code>snap install kubectl --classic</code>)</li>
  <li>helm cli (etwa <code>snap install helm --classic</code>)</li>
  <li>kubefedctl cli <code>
    wget "https://github.com/kubernetes-sigs/kubefed/releases/download/v0.3.1/kubefedctl-0.3.1-linux-amd64.tgz"
    tar xvfz kubefedctl-0.3.1-linux-amd64.tgz</code></li>
</ul>

<strong>Installation:</strong>

Kubefed kann auf beiden Clustern mit helm installiert werden:

```shell
# namespace erstellen, wenn helm das noch nicht kann
kubectl create namespace  kube-federation-system 
helm repo add kubefed-charts https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts
# helm > 3.2
helm --namespace kube-federation-system upgrade -i kubefed kubefed-charts/kubefed
# helm >3.2
helm --namespace kube-federation-system upgrade -i kubefed kubefed-charts/kubefed --create-namespace
```

Derzeit ist die Dokumentation noch fuer Helm v2. Der <a href="https://github.com/kubernetes-sigs/kubefed/pull/1260">PR fuer v3 </a>ist in Arbeit, interessant fuer die zu installierenden CRDs. Es lohnt sich also noch ein Blick in die Doku.

Wir erinnern uns, dass wir eine Kube-Config mit jeweils einem Kontext fuer die beteiligten Cluster, nennen wir sie quickstart-01 und quickstart-02, erstellen wollten. Beim Verbinden mit kubefedctl greifen wir auf diese Informationen zurueck:

```shell
./kubefedctl join quickstart-01 --cluster-context quickstart-01 --host-cluster-context quickstart-01
./kubefedctl join quickstart-02 --cluster-context quickstart-02 --host-cluster-context quickstart-01
```

Der quickstart-01 Cluster ist also sowas wie der Master (Host) und der andere Cluster foederiert mit diesem.
Den Status koennen wir sofort abfragen:

```shell
    kubectl -n kube-federation-system get kubefedcluster
    NAME AGE READY
    quickstart-01 3h25m True
    quickstart-02 3h21m True
```

Wenn READY nicht "True" ist, sollte man die Netzwerkverbindung ueberpruefen oder mit describe schauen, was es evtl. vom Cluster fuer Fehlermeldungen gibt.

Nach dem Verbinden ermoeglichen wir die Foederation fuer bestimmte Resourcen, hier etwa statefulsets:

```shell
./kubefedctl enable statefulsets --host-cluster-context=quickstart-01
```

Und dann die eigentliche Foederation:

```shell
./kubefedctl federate statefulset folding-at-home -n folding
```

Die Arbeitslast sollte sich jetzt verdoppeln: Auf quickstart-01 und auf quickstart-02

```shell
    # source cluster
    $ kubectl config set current-context quickstart-01
    $ kubectl -n folding get all
    NAME READY STATUS RESTARTS AGE
    pod/folding-at-home-0 1/1 Running 0 8d
    pod/folding-at-home-1 1/1 Running 0 8d
    pod/folding-at-home-2 1/1 Running 0 8d
    pod/folding-at-home-3 1/1 Running 0 8d
    pod/folding-at-home-4 1/1 Running 0 8d
    NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
    service/folding-at-home-0 ClusterIP 10.43.175.57 <none> 80/TCP 127d
    service/folding-at-home-1 ClusterIP 10.43.206.48 <none> 80/TCP 127d
    service/folding-at-home-2 ClusterIP 10.43.232.241 <none> 80/TCP 127d
    service/folding-at-home-3 ClusterIP 10.43.152.4 <none> 80/TCP 127d
    service/folding-at-home-4 ClusterIP 10.43.201.56 <none> 80/TCP 127d
    NAME READY AGE
    statefulset.apps/folding-at-home 5/5 127
     
    # target cluster 
    $ kubectl config set current-context quickstart-02
    $ kubectl -n folding get all
    NAME READY STATUS RESTARTS AGE
    pod/folding-at-home-0 1/1 Running 0 156m
    pod/folding-at-home-1 1/1 Running 0 156m
    pod/folding-at-home-2 1/1 Running 0 155m
    pod/folding-at-home-3 1/1 Running 0 155m
    pod/folding-at-home-4 1/1 Running 0 155m
    NAME READY AGE
    statefulset.apps/folding-at-home 5/5 156m
```

Wir man sieht, hat sich nur die Resource statefulset foederiert, services gibt es immer noch nicht. DIese muesste man auf aehnliche Weise anstossen. Einige Beispiele fuer Resource-Typen gibt es im <a href="https://github.com/kubernetes-sigs/kubefed/tree/master/example/sample1">Beispiel-Code</a>.
