---
layout: post
tag: de
title: "Kubernetes & OpenStack Teil 1: External-DNS mit Designate in Minikube"
subtitle: "Sowie Kubernetes eine Weiterentwicklung von OpenStack Infrastruktur ist, waere es doch gut, bisher Entwickeltes und Bewaehrtes weiter zu nutzen. Designate ist der DNS-Dienst von OpenStack und ueber eine Erweiterung in Kubernetes zu benutzen."
date: 2019-09-29
background: '/images/k8s-cosmos.png'
---

Nehmen wir mal an, wir wollen unsere <a href="https://blog.eumelnet.de/blogs/blog8.php/joomla-installation-mit-kubernetes-und-helm-1">Joomla-Instalation mit Minikube</a> dahingehend erweitern, dass wir den Hostnamen unserer Seite in einen DNS-Server eintragen wollen. Dazu wollen wir als Erstes auch unseren Minikube mit dem Addon "Ingress" erweitern. Ingress stellt nichts weiter als die "Aussenkante" vom Kubernetes-Cluster dar. Dazu starten wir Minikube mit der speziellen Option:

```
./minikube start --vm-driver=none --extra-config=apiserver.service-node-port-range=80-35000 addons enable ingress
```

Als Naechstes erstellen wir <a href="https://kubernetes.io/docs/concepts/configuration/secret/#creating-a-secret-from-generator">ein Secret</a> mit unseren Cloud-Credentials in Openstack, hier als Beispiel Open Telekom Cloud:

kustomization.yaml
```
secretGenerator:
- name: otc-external-dns
  literals:
  - OS_USERNAME=xxxx
  - OS_PROJECT_NAME=eu-de
  - OS_IDENTITY_API_VERSION=3
  - OS_USER_DOMAIN_NAME=OTC-EU-DE-000000xxxxxxx
  - OS_PASSWORD=xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
  - OS_AUTH_URL=https://iam.eu-de.otc.t-systems.com:443/v3
```

Erstellen des Secrets, mit dieser Datei in einem extra Verzeichnis:

```
kubectl -n joomla apply -k .
```

Das Secret hat eine gehashte Endung, die sich mit jeder Aenderung des Inhalts ebenfalls aendert. DIesen Namen brauchen wir weiter unten.
Fuer POD-Security muessen wir einen Service-Account anlegen, sowie ein ClusterRole und ClusterRoleBinding, um unseren Deployment gewisse Rechte im Cluster einzuraeumen:

```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
  namespace: joomla
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: external-dns
  namespace: joomla
rules:
  -
    apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - watch
      - list
  -
    apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - get
      - watch
      - list
  -
    apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - watch
      - list
  -
    apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
  -
    kind: ServiceAccount
    name: external-dns
    namespace: joomla

```

Nun zum eigentlichen Deployment:

```
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      containers:
      - name: external-dns
        image: registry.opensource.zalan.do/teapot/external-dns
        args:
        - --source=service 
        - --source=ingress
        - --domain-filter=otc.eumel.de
        - --provider=designate
        - --registry=txt
        - --interval=10s
        env: # values from openrc file
        - name: OS_USERNAME
          valueFrom:
            secretKeyRef:
              key: OS_USERNAME
              name: otc-external-dns-tcc6t66fc2
        - name: OS_PROJECT_NAME
          valueFrom:
            secretKeyRef:
              key: OS_PROJECT_NAME
              name: otc-external-dns-tcc6t66fc2
        - name: OS_IDENTITY_API_VERSION
          valueFrom:
            secretKeyRef:
              key: OS_IDENTITY_API_VERSION
              name: otc-external-dns-tcc6t66fc2
        - name: OS_USER_DOMAIN_NAME
          valueFrom:
            secretKeyRef:
              key: OS_USER_DOMAIN_NAME
              name: otc-external-dns-tcc6t66fc2
        - name: OS_PASSWORD
          valueFrom:
            secretKeyRef:
              key: OS_PASSWORD
              name: otc-external-dns-tcc6t66fc2
        - name: OS_AUTH_URL
          valueFrom:
            secretKeyRef:
              key: OS_AUTH_URL
              name: otc-external-dns-tcc6t66fc2
      serviceAccountName: external-dns
```

Was passiert hier? Es wird von einem Docker-Container von Zalando ein Service erstellt fuer die Sub-Domain otc.eumel.de. Es werden Datenbankeintraege als TXT Eintraege im DNS verwendet. Dadurch wird registriert, welcher Cluster welche DNS-Eintraege in derselben Domain verwaltet. Alle 10 Sekunden wird der Status geprueft - das heisst, eventuell geloeschte DNS-Eintraege erneuert oder aktualisiert. Verwendet wird der OpenStack-Dienst Designate. Dazu werden ueber das erstellte Secret die entsprechenden Credentials im POD bereitgestellt.
DIe Dateien oben werden mit dem ueblichen [codespan]kubectl apply[/codespan] dem Kubernetes-Cluster appliziert.

Geben wir nun unserer Joomla-Anwendung einen Hostnamen. <a href="https://github.com/helm/charts/blob/master/stable/joomla/values.yaml">Im values.yaml des Joomla Charts</a> aktivieren wir Ingress:

```
ingress:
  enabled: true
  hosts:
  - name: joomla.otc.eumel.de
```

Nach der Aenderung erhoehen wir in Charts.yaml die Versionsnummer und deployen die neue Version mit:

```
helm upgrade --namespace joomla joomla  joomla
```

Den Ingress-Service editieren wir mit [codespan]kubectl edit ingress joomla -n joomla[/codespan]

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    external-dns.alpha.kubernetes.io/target: 80.158.36.239
  creationTimestamp: "2019-09-23T19:20:26Z"
  generation: 3
  labels:
    app: joomla
    chart: joomla-6.1.9
    heritage: Tiller
    release: joomla
  name: joomla
  namespace: joomla
  resourceVersion: "1502495"
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
status:
  loadBalancer: {}
```

Mit der Annotation fuer den external-dns setzen wir das Target auf eine IP-Adresse. Es wird praktisch ein A-Record im DNS erstellt. Den Hostnamen definieren wir in den Spec (sollte aus dem Helm Chart uebernommen sein). Backend ist der joomla Service auf Port 80.

Im Logfile des DNS-PODs ([codespan]kubectl  logs external-dns-78948679d-mp2q6 -n joomla[/codespan]) sollte sowas zu lesen sein:

```
time="2019-09-28T22:42:19Z" level=debug msg="Endpoints generated from ingress: joomla/joomla: [joomla.otc.eumel.de 0 IN A 80.158.36.239 []]"
```

und unter http://joomla.otc.eumel.de unser Joomla zu sehen sein.

Hinweise: Die Domain/Sub-Domain muss zum OpenStack-DNS delegiert sein:

```
# host -t ns otc.eumel.de
otc.eumel.de name server ns2.open-telekom-cloud.com.
otc.eumel.de name server ns1.open-telekom-cloud.com.
```

[codespan]helm uograde[/codespan] fuehrt zum Ueberschreiben manuell durchgefuehrter Aenderungen im Deployment. Es dient hier nur zur Veranschaulichung der Wirkungsweise


