---
layout: post
tag: de
title: "Neue User anlegen in Kubernetes / K3S mit RBAC"
subtitle: "Wer sich ueber die <a href='https://blog.eumelnet.de/blogs/blog8.php/kubernetes-install-quickies'>Install Quickies</a> mit Minikube oder K3S einen Kubernetes-Cluster hochgezogen hat, hat meistens keine Probleme beim Adminzugriff mit kubectl - der allumfassende Superuser-Account ist meist bei der Installation mit dabei. Wie sieht es aber auch, wenn wir andere Benutzer aufs System lassen wollen? Oder Maschinenaccounts, die nur bestimmte Aktionen ausfuehren sollen? Zeit fuer einen Ausflug zu RBAC und der Authentifizierung in Kubernetes."
date: 2020-07-23
background: '/images/k8s-cosmos.png'
---

Kubernetes unterscheidet zwischen mehreren Authentifizierungsverfahren. Da waere die klassische Username/Passwort-Kombination. Wir finden sie bei K3S in der Datei `/etc/rancher/k3s/k3s.yaml`.
Ein anderes Verfahren ist der Bearer-Token - ein kryptisch langer String, mit dem man sich am Cluster authentifizieren kann.
Und dann ist das Zertifikat-Verfahren. Das wird auch zur inneren Cluster-Kommunikation sehr stark zwischen den einzelnen Komponenten genutzt. Bei der Installation ist eine Cluster-CA ausgestellt wurden und mit der sind alle anderen Zertifikate signiert oder mit einer Client-CA sogar zwischenzertififiert. 
Klingt alles hochkompliziert, hier ist ein <a href="https://github.com/gnuu-de/k8s/blob/master/createuser.sh">Shell-Script</a>, was uns mit 6 openssl-Kommandos Clientzertifikate von der Client-CA der Cluster-CA ausstellen kann. Das Script geht von einer K3S-Default-Installation aus und legt einen User "github" fuer den Namespace "gnuu" an. Die Werte kann man leicht modifizieren.
VORHER! Also bevor man das Script startet, muss man im Cluster die Berechtigungen festlegen. Das passiert durch RBAC, Role Based Access Control. Das passiert im Cluster bei internen Diensten mit <a href="https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/">ServiceAccounts</a>. Dann werden Rollen festgelegt. Diese Roles beschreiben die Rechte, wie API-Zugriffe, Kommandos und Resourcen. Das RoleBinding verbindet dann den Account mit der Rolle. Das wars. RBAC gibt es namespace-basiert oder cluster-weit - dann heisst es ClusterRoles und ClusterRoleBinding.

Wie sieht jetzt so ein RBAC fuer unseren "github" User mit Client-Zertifikat aus:

```yaml
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    app: k3s-clients
  name: k3s-clients
  namespace: gnuu
rules:
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["create", "get", "update", "list", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["create", "get", "update", "list", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    app: k3s-clients-github
  name: k3s-clients-github
  namespace: gnuu
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: k3s-clients
subjects:
- name: github@default
  kind: User
  apiGroup: rbac.authorization.k8s.io
```

Die Rolle enthaelt also 2 API-Bereiche - den Core ('') und 'apps'. In diesen Bereichen kann ich Resourcen ansprechen: Pods, Services, Deployments. Und auf den Resourcen kann ich bestimmte Taetigkeiten ausfuehren: Erstellen (create), Anzeigen (get/list) oder Loeschen (delete).
Das RoleBinding referenziert auf die Rolle, die oben vom Namen beschrieben ist und schaut dann nach Usern mit dem Subject: github@default - das ist das Subject des Client-Certificats. Und wenn das passt, kann der User diese Rechte ausfuehren. 
Probieren wir das mal aus. Wir applyen das rbac.yaml oben und fuehren dann das Shellscript aus:

```shell
 ./createuser.sh
#>>GEN-KEY
Signature ok
subject=CN = github@default, O = key-gen
Getting CA Private Key
#>>KUBE-CONFIG
Cluster "default" set.
User "github" set.
Context "github@default" created.
Property "current-context" set.
NAME                         READY   STATUS      RESTARTS   AGE
uucp-build-ggsfn             0/1     Completed   0          18d
repo-55d6c7d667-njt4m        1/1     Running     1          19d
mysql-5cbf8684cc-jfwh2       1/1     Running     1          24d
job-6dcb8c76cb-2clt5         1/1     Running     3          18d
riot-79bc8b44ff-6jjsn        1/1     Running     0          5d22h
matrixbot-678f8bfdc7-fxgpx   1/1     Running     0          5d22h
api-75475dbd95-d4p77         1/1     Running     0          5d14h
mail-0                       1/1     Running     0          2d10h
news-0                       1/1     Running     0          2d10h
uucp-86ffbc8b4-jwgzb         1/1     Running     0          2d10h
szip-build-g9br4             0/1     Completed   0          46h
usr-7d48566-xj2hx            1/1     Running     0          4h22m
adm-68cf8949bc-vp7b6         1/1     Running     0          62m
nginx-785d7c8959-8kjvz       1/1     Running     0          30m

```

Funktioniert! Wir koennen die PODs in unserem Namespace auflisten. 

Die Config koennen wir mit nach Hause nehmen:

```shell
export KUBECONFIG=/opt/k3s-clients/gen/kube/github.kubeconfig
```
