---
layout: post
tag: de
title: "LetsEncrypt gesicherte Webseiten in Kubernetes mit dem Cert-Manager"
date: 2019-10-02
---

Nie war es so einfach, eine mit SSL-Zertifikat geschüWebseite zu erstellen wie heute.<a href="https://letsencrypt.org/de/"> LetsEncrypt</a> ist eine freie, automatisierte und offene Zertifizierungsstelle. Keine Formulare ausfuellen, kein Geld einwerfen (ausser zum Spenden), keine lange Wartezeiten bei der Zertifikatsausstellung. LetsEncrypt stellt sofort eine sichere Verbindung her.

<img src="/blog/images/kubernetes.png" alt="Kubernetes" title="Kubernetes Logo" align="middle" width="420" height="420" />

---
<!--more-->

Folgende Vorausetzungen seien gegeben: Wir haben unsere <a href="https://blog.eumelnet.de/blogs/blog8.php/joomla-installation-mit-kubernetes-und-helm-1">Joomla-Webseite mit Minikube und Helm</a> aufgesetzt. Die Seite hat einen <a href="https://blog.eumelnet.de/blogs/blog8.php/kubernetes-openstack-teil-1-external-dns-mit-designate">ordentlichen Namen</a> und ist aus dem Internet erreichbar.
Als Erstes ueberpruefen wir, ob bei minikube wirklich Ingress aktiviert ist:

```
minikube addons enable ingress
```

Bisher funktionierte der Dienst auch mit einem LoadBalancer Service und ExternalIP. Der <a href="https://docs.cert-manager.io/en/latest/">Cert-Manager</a> benoetigt aber zwingend den Ingress. Er wird das Ausstellen und Erneuern unserer Zertifikate verwalten. Zwei Wege seien hier zur Installation aufgezeigt:

```
git clone https://github.com/jetstack/cert-manager.git
cd cert-manager/deploy
git checkout release-0.10
```

Leider funktioniert die Installation mit Helm nicht mehr so sauber, da die CRD und der Namespace vorher angelegt werden muessen:

```
kubectl  apply -f manifests/00-crds.yaml
kubectl  apply -f manifests/01-namespace.yaml
```

Abhaengigkeiten aufloesen

```
cd deploy/charts/cert-manager/
helm dep update 
cd ../..
```

Man koennte jetzt das Original-Repo nehmen:

```
helm repo add jetstack https://charts.jetstack.io
helm install --name cert-manager --namespace cert-manager --version v0.10.0 jetstack/cert-manager
```

ODER nutzt gleich das Git-Repo:

```
helm install --name cert-manager --namespace cert-manager --version v0.10.0 charts/cert-manager
```

Der Cert-Manager sollte im Namespace cert-manager deployt sein.

<pre>
# kubectl  get all -n cert-manager
NAME                                          READY   STATUS    RESTARTS   AGE
pod/cert-manager-bf969f8db-s59nm              1/1     Running   0          32m
pod/cert-manager-cainjector-7944bc7bd-dpjgl   1/1     Running   0          32m
pod/cert-manager-webhook-5dd447844d-g5gtf     1/1     Running   2          32m


NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/cert-manager           ClusterIP   10.106.96.105   <none>        9402/TCP   32m
service/cert-manager-webhook   ClusterIP   10.98.224.159   <none>        443/TCP    32m


NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cert-manager              1/1     1            1           32m
deployment.apps/cert-manager-cainjector   1/1     1            1           32m
deployment.apps/cert-manager-webhook      1/1     1            1           32m

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/cert-manager-bf969f8db              1         1         1       32m
replicaset.apps/cert-manager-cainjector-7944bc7bd   1         1         1       32m
replicaset.apps/cert-manager-webhook-5dd447844d     1         1         1       32m
</pre>

Als naechstes brauchen wir die Konfiguration der Aussteller (Issuer):

issuer.yaml

<pre>

apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email:  eumel@eumel.de
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class:  nginx

apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    email: eumel@eumel.de
    http01: {}
    privateKeySecretRef:
      name: letsencrypt-production
    server: "https://acme-v02.api.letsencrypt.org/directory"
</pre>

Wie man sieht, ist die Konfiguration recht selbsterklaerend. Es sind 2 Aussteller - einer fuer Staging zum Testen und einer fuer Produktion. Bei beiden wird<a href="https://letsencrypt.org/de/how-it-works/"> http01-Challenge-Verfahren</a> angewendet und meine E-Mail-Adresse ist als Kontaktadresse angegeben, damit mich LetsEncrypt ueber Unregelmaessigkeiten oder Zertifikatablauf informieren kann.

```
kubectl apply -f issuer.yaml -n joomla
```

Unser Joomla-Ingress-Service sieht so aus:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    certmanager.k8s.io/issuer: letsencrypt-staging
    external-dns.alpha.kubernetes.io/target: 80.158.36.239
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
  creationTimestamp: "2019-09-23T19:20:26Z"
  generation: 5
  labels:
    app: joomla
    chart: joomla-6.1.12
    heritage: Tiller
    release: joomla
  name: joomla
  namespace: joomla
  resourceVersion: "1929460"
  selfLink: /apis/extensions/v1beta1/namespaces/joomla/ingresses/joomla
  uid: 29fcda05-e414-46fe-8bcd-39401c0c90bc
spec:
  rules:
  - host: joomla.otc.eumel.de
    http:
      paths:
      - backend:
          serviceName: joomla
          servicePort: 80
        path: /
  tls:
  - hosts:
    - joomla.otc.eumel.de
    secretName: joomla-otc-eumel-de-tls
status:
  loadBalancer:
    ingress:
    - ip: 192.168.1.170
```

Abgerufen mit <code>kubectl get ingress joomla -n joomla -o yaml</code> und laesst sich mit <code>kubectl apply </code> auch genauso applizieren. 
Zur Erklaerung: 
Wir benutzen letsencrypt-staging zum Testen bis es funktioniert. LetsEncrypt hat ein Quota auf seine Services und wenn man zu oft Zertifikate mit derselben Domain und derselben IP ausstellt, wird man gesperrt.  Ansonsten haben wir mit der Annotation LetsEncrypt schon akiviert. Das Ingress verweist auf einen Service und einen Service-Port. Man sollte pruefen, ob der Service denselben Namen und Port hat und natuerlich auch funktioniert. Zum Beispiel hat der Service Endpunkte:

```
kubectl get ep -n joomla
```

Der Service hat zum Beispiel im spec einen Selector:

<pre>
  selector:
    app: joomla
</pre>

Der Selector wird fuer Pods im Deployment gesetzt:

```
kubectl describe deployment.apps/joomla -n joomla
Name:                   joomla
Namespace:              joomla
CreationTimestamp:      Mon, 23 Sep 2019 19:20:26 +0000
Labels:                 app=joomla
                        chart=joomla-6.1.12
                        heritage=Tiller
                        release=joomla
```

So haben wir die gesamte Wertschoepfungskette von Ingress->Service->Pod

Wo ist das LetsEncrypt geblieben? Das versteckt sich in den spec des Ingress:

<pre>
  tls:
  - hosts:
    - joomla.otc.eumel.de
    secretName: joomla-otc-eumel-de-tls
</pre>

Fuer den Host ist ein TLS-Zertifikat notwendig, welches in einem Secret abgespeichert wird.
Schauen wir uns das einmal an:

<pre>
# kubectl describe certificate joomla-otc-eumel-de-tls -n joomla
Name:         joomla-otc-eumel-de-tls
Namespace:    joomla
Labels:       app=joomla
              chart=joomla-6.1.12
              heritage=Tiller
              release=joomla
Annotations:  <none>
API Version:  certmanager.k8s.io/v1alpha1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2019-10-01T21:02:10Z
  Generation:          4
  Owner References:
    API Version:           extensions/v1beta1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  joomla
    UID:                   48e4d50b-4ace-4623-867b-b4db6eb16c72
  Resource Version:        1948542
  Self Link:               /apis/certmanager.k8s.io/v1alpha1/namespaces/joomla/certificates/joomla-otc-mcsps-de-tls
  UID:                     d0c1536e-ece6-40b1-8631-3c77d6ec147d
Spec:
  Acme:
    Config:
      Domains:
        joomla.otc.eumel.de
      http01:
        Ingress Class:  nginx
  Dns Names:
    joomla.otc.eumel.de
  Issuer Ref:
    Kind:       Issuer
    Name:       letsencrypt-production
  Secret Name:  joomla-otc-eumel-de-tls
Status:
  Conditions:
    Last Transition Time:  2019-10-01T21:02:37Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2019-12-30T20:02:36Z
Events:                    none
</pre>
Die Ausstellung kann ich auch in <code>kubectl get events -n cert-manager</code> verfolgen. Haengengebliebene Ausstellungsversuche stehen in <code>kubectl get challenges -A</code>. 
Wenn es keine Fehler gibt, sollte unsere Webseite jetzt mit LetsEncrypt gesichert sein und rechtzeitig vor dem 30.12. wird das Zertifikat automatisch erneuert. Toll!

Dokumentation zu Ingress: https://kubernetes.github.io/ingress-nginx/
