---
layout: post
tag: de
title: "Kubernetes mit Cloud-Provider OpenStack"
subtitle: Mit Kubernetes koennen wir wunderbare Applikationen erstellen und APIs verwalten. Alles nuetzt aber nichts, wenn wir im Hintergrund keine Resourcen wie Netzwerk, Speicher oder Rechenleistung haben. Was liegt naeher, als mit unserem Kubernetes OpenStack Resourcen zu verwalten?
date: 2019-10-15
background: '/images/k8s-cosmos.png'
---

Kubernetes verwaltet Cloud Resourcen im sogenannten <a href="https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/">Cloud-Controller-Manager</a>. Fuer OpenStack gibt es den Cloud Provider OpenStack, beschrieben etwa <a href="https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/using-controller-manager-with-kubeadm.md">hier</a>. Das Beispiel bezieht sich auf <a href="https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/">Kubeadm</a>, ein weiteres Leichtgewicht zur Kubernetesinstallation aehnlich wie Minikube.

<strong>Installation Kubeadm</strong>
Wie benutzen einen Rechner/VM mit Ubuntu 16.04 (fuer neuere OS gab es noch keine Installationspakete) und installieren die erforderlichen Programmpakete:

```
apt-get update &amp;&amp; apt-get install -y apt-transport-https curl docker.io
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
 apt-get update
 apt-get install -y kubelet kubeadm kubectl
```

Kubeadm laesst sich mit <code>kubeadm  init</code> starten. Wir brauchen allerdings noch ein Providerr-Netzwerk. Am besten installieren wie <a href="https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#tabs-pod-install-3">Canal</a>

```
kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/canal.yaml
```

Kubernetes zuruecksetzen und mit der neuen Netzwerkeinstellung starten:

```
kubeadm reset
kubeadm init --pod-network-cidr=10.244.0.0/16
```

Cloud Controller OpenStack
--------------------------

Die Installation ist im <a href="https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/using-controller-manager-with-kubeadm.md">Projekt Repo</a> beschrieben. 
Zur Authenifizierung an der OpenStack Cloud brauchen wir Credentials, die wir in einem Secret ablegen. Dazu erstellen wir erst mal eine Textdatei <ins>cloud-config</ins>

```
[Global]
auth-url    = https://iam.eu-de.otc.t-systems.com:443/v3
username    = minikube
password    = xxxxxxxxxxxxxxxxx
tenant-name = eu-de
domain-name = OTC-EU-DE-000000000000xxxxx
region      = eu-de

[LoadBalancer]
floating-network-id = 0a2228f2-7f8a-45f1-8e09-9039e1d09975
subnet-id = 01c192c3-060c-4bbe-99d3-29a23369ef2f
```

Wie zu sehen, verwenden wir die Open Telekom Cloud als OpenStack Backend. Mit <code>openstack network list</code> kriegen wir die <ins>floating-network-id</ins> vom <ins>admin_external_net</ins>. Die <ins>subnet-id</ins>  ist die ID des Subnetzes im VPC.

Das Secret wird folgendermassen deployt:

```
kubectl create secret -n kube-system generic cloud-config --from-literal=cloud.conf="$(cat cloud-config)" --dry-run -o yaml > cloud-config-secret.yaml
kubectl -f cloud-config-secret.yaml apply
```

Mit <ins>taint</ins> wird unser Node <ins>quickstart</ins> auf die Installation von Cloud Controller Manager vorbereitet:

```
kubectl taint node quickstart node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule
```

Der Nodename kann natuerlich variieren. 
Der folgende Code-Block installiert den Cloud Controller Manager OpenStack mit RBAC:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/cluster/addons/rbac/cloud-controller-manager-roles.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/cluster/addons/rbac/cloud-controller-manager-role-bindings.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/manifests/controller-manager/openstack-cloud-controller-manager-ds.yaml
```

Wenn alles bis hier erfolgreich war, sollte ein POD mit dem CCM pro Node laufen:

```
kubectl get pods -n kube-system | grep openstack
openstack-cloud-controller-manager-pw4r8          1/1     Running   0          14m
```

Die Verwendung der OTC birgt auch ihre Tuecken. Die Neutron-API ist nicht 100%ig kompatibel mit der OpenStack-Spezifikation. Deswegen kann man hier und heute (Stand 10/2019) keine Floating-IPs erstellen. Dazu ist eine <a href="https://github.com/kubernetes/cloud-provider-openstack/blame/master/pkg/cloudprovider/providers/openstack/openstack_loadbalancer.go#L1294">geringfuegige Aenderung des Codes </a>und der <a href="mtr.external.otc.telekomcloud.com/mcsps/openstack-cloud-controller-manager:v0.0.2">Neubau des Docker-Images</a> notwendig.
Das Image tauschen wir mit <code>kubectl  edit daemonset.apps/openstack-cloud-controller-manager  -n kube-system</code> aus.

Netzwerk/Loadbalancer
---------------------

Im <a href="https://github.com/kubernetes/cloud-provider-openstack/tree/master/examples/loadbalancers">Beispielverzeichnis</a> gibt es 2 Deployments, fuer internen und externen Loadbalancer. Die Netzwerk-IDs haben wir im Secret schon weiter oben gesetzt. Im Prinzip koennen wir diese beiden Dateien direkt uebers Netz deployen:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/examples/loadbalancers/external-http-nginx.yaml -n kube-system
kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/examples/loadbalancers/internal-http-nginx.yaml -n kube-system
```

Ist das Deployment erfolgreich haben wir 2 neue Loadbalancer-Services mit unseren Nodes als Backend-Server:

<pre>
# kubectl  get services -n kube-system
NAME                          TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                  AGE
external-http-nginx-service   LoadBalancer   10.97.54.3      80.158.7.207   80:32133/TCP             37m
internal-http-nginx-service   LoadBalancer   10.107.129.85   192.168.1.28   80:31179/TCP             4s
kube-dns                      ClusterIP      10.96.0.10      none           53/UDP,53/TCP,9153/TCP   25h
</pre>

Speicher/Cinder
---------------

Den OpenStack Speicherdienst Cinder koennen wir als Speicherbackend in Kubernetes benutzen. Auch dafuer findet sich ein Beispiel im OpenStack Cloud Controller Manager Repo:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/examples/persistent-volume-provisioning/cinder/cinder-in-tree-full.yaml
```

Hier wird eine <ins>Storageclass</ins> angelegt und ein 1GB grosses Volume erstellt:
<pre>
# kubectl get pvc
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
myclaim-gold   Bound    pvc-8f6212eb-6ffd-4b6d-a0d3-4872324e8a40   1Gi        RWO            gold           77m
</pre>

Das Volume wird in einem POD als Mount zur Verfuegung gestellt.

<strong>Rechenleistung/Compute</strong>

Nicht Bestandteil des OpenStack Cloud Controller Managers. Andere Projekte:
<a href="https://github.com/kubermatic/machine-controller/blob/master/docs/cloud-provider.md#openstack">Kubermatic Machine Controller mit OpenStack</a>
<a href="https://github.com/kubernetes/autoscaler">Autoscaler</a><a href="https://github.com/kubernetes-sigs/cluster-api-provider-openstack">
Kubernetes Cluster API Provider OpenStack</a>

<strong>Fazit:</strong>
Das sind nur einige Anwendungsfaelle mit dem OpenStack Cloud Controller Manager. Entwicklungsmoeglichkeiten gibt es zur Genuege mit zunehmenden Diensten, die in der Cloud zur Verfuegung gestellt werden.

Vollstaendige Konfigurationsliste vom OpenStack Cloud Controller Manager: https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/provider-configuration.md
