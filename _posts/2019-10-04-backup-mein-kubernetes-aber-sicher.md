---
layout: post
tag: de
title: "Backup mein Kubernetes - aber sicher!"
subtitle: Mit einiger Zeit sammelt sich so einiges im Kubernetes Cluster an. Sachen, die man nicht sauber mit Helm installiert hat. Oder die man irgendwie mal zwischendurch zum Ausprobieren geaendert und dann doch so belassen hat. Uns graust es schon vorm Datenverlust. Wie kann man sowas alles backupen?
date: 2019-10-04
background: '/images/k8s-cosmos.png'
---

Wie wir vielleicht schon mitgekriegt haben, liegen die meisten Resourcen in Kubernetes in <a href="https://de.wikipedia.org/wiki/YAML">YAML-Files</a> vor, Damit wurden zumindest die meisten Sachen erzeugt - sei es als HELM Chart oder einfaches Deployment-File - local oder uebers Netz, einerlei. 
Schmutzig, klein, genial ist <a href="https://github.com/pieterlange/kube-backup">Kube-Backup</a>, ein handliches Backup-Tool fuer Kubernetes.
Voraussetzungen sei wieder unser laufender <a href="https://blog.eumelnet.de/blogs/blog8.php/joomla-installation-mit-kubernetes-und-helm-1">Minikube</a>, es funktioniert aber mit jedem anderen Kubernetes. 
Zielbild ist es, unsere Cluster-Konfiguration auf Github zu sichern. Immer und immer wieder. Wie machen wir das? Mit einem Cronjob! Also automatisch. 
Als erstes erstellen wir ein SSH-Schluesselpaar auf unserem, Minikuberechner:

```
  ssh-keygen -f ./id_rsa
  ssh-keyscan github.com > known_hosts
```

 Dann erstellen wir ein neues Github-Repository. Gerne auch <ins>private</ins>, denn das ist neuerdings erlaubt. Unter <ins>Deploykeys</ins> fuegen wir den frisch erstellten SSH-Public-Key unseres Minikuberechners <ins>id_rsa.pub</ins> hinzu.

Vom SSH-Private-Key und die known_hosts erstellen wir eine Secret im Kubernetes:

```
kubectl create secret generic kube-backup-ssh -n kube-system --from-file=id_rsa --from-file=known_hosts
```

Wenn wir schon dabei sind, erstellen wir noch mit <code>gpg --generate-key</code>den GPG-Key und auch daraus ein Secret:

```
gpg --export-secret-key -a root@minikube > gpg-private.key
kubectl create secret generic kube-backup-gpg -n default --from-file=gpg-private.key
```

Das Git-Repo mit unserer Kubernetes ist mit git-crypt entsprechend zu preparieren:

```
cd git/minikube
git-crypt init
cat &lt;EOF&gt; .gitattributes
*.secret.yaml filter=git-crypt diff=git-crypt
.gitattributes !filter !diff
EOF
git-crypt add-gpg-user eumel@eumel.gnuu.de
```

Den GPG-Public-Key vom Minikube-Rechner muss ich erstmal von seinem GPG-Schluesselbund exportieren:

```
gpg --armor --output minikube.asc --export root@minikube
```

und in meinem persoenlichen GPG-Schluesselbund importieren:

```
gpg --import minikube.asc
```

Um den Schluessel wie oben mit <code>git-crypt add-gpg-user root@minikube</code> hinzufuegen zu koennen, muss ich diesem noch vertrauen:

```
gpg --edit-key root@minikube
>trust
5
y
```

Zurueck zum Kubernetes-Cluster. Dort gibts erstmal ein ClusterRoleBinding zu deployen:

rbac.yaml
<pre>
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kube-backup-reader
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs: ["get", "list"]
---

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kube-backup
  namespace: default
subjects:
- kind: ServiceAccount
  name: kube-backup
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: kube-backup-reader
  apiGroup: rbac.authorization.k8s.io
</pre>

```
kubectl apply -f rbac.yaml
```

Nun das Herzstueck, der CronJob:

cronjob-ssh.yaml
<pre>
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-backup
  namespace: kube-system

---

apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: kube-state-backup
  namespace: kube-system
  labels:
    app: kube-backup
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: kube-backup
          name: kube-backup
        spec:
          containers:
         # - image: quay.io/plange/kube-backup:1.12.0-1
          - image: eumel8/kube-backup:latest
            imagePullPolicy: Always
            name: backup
            resources: {}
            securityContext:
              runAsUser: 1000
            env:
            - name: GIT_REPO
              value: "git@github.com:eumel8/minikube.git"
            - name: GITCRYPT_ENABLE
              value: "true"
            - name: RESOURCETYPES
              value: "ingress deployment configmap secret pvc svc rc ds thirdpartyresource networkpolicy statefulset storageclass cronjob"
            volumeMounts:
            - mountPath: /backup/
              name: cache
            - mountPath: /backup/.ssh
              name: sshkey
            - mountPath: /secrets
              name: gpgkey
          dnsPolicy: ClusterFirst
          terminationGracePeriodSeconds: 30
          serviceAccountName: kube-backup
          volumes:
          - name: sshkey
            secret:
              defaultMode: 420
              secretName: kube-backup-ssh
          - name: gpgkey
            secret:
              defaultMode: 420
              secretName: kube-backup-gpg
          - name: cache
            emptyDir: {}
          restartPolicy: OnFailure
</pre>

Ein kurzer Blick auf das File, bevor wir es deployen.
Ich habe das Repo <a href="https://github.com/eumel8/kube-backup">geforked</a> und ein eigenes Image gebaut und auf Dockerhub gepusht. Das geht mit ein paar Klicks, wenn man einen Gihub- und einen Dockerhub-Account hat. Im <a href="https://github.com/eumel8/kube-backup/blob/master/Dockerfile">Dockerfile</a> kann man schauen, was im Image alles drin ist. Ich habe Alpine und git-crypt aktualisiert. Auch eine neuere Version von kubectl ist drin. Ob es das richtige File aus dem Internet ist, wird mit einer sha256 Pruefsumme ueberprueft. So kann uns niemand Schadsoftware unterschieben.

Das entrypoint.sh ist die Arbeitslogik. Dort habe ich mal <a href="https://github.com/pieterlange/kube-backup/commit/eb46f08a993210c553ec18ce5dc20c3822348352">ein paar Debug-Marker</a> gesetzt, als es auf den ersten Versuch nicht funktionierte. Sollte aber wieder alles wie im Original sein und veranschaulichen, wie so ein Dockerfile bzw. Docker-Image fuer Kubernetes funktioniert.

Jetzt aber los:

```
kubectl apply -f cronjob-ssh.yaml
```

Wenn alles klappt, sollte in unserem Github-Repository mit der Kubernetes-Config etwas passieren. 

<pre>
commit d685ad67d808842258622a6f07326b5c0c3f2a56
Author: kube-backup &lt;kube-backup@example.com&gt;
Date:   Fri Oct 4 18:23:32 2019 +0000

    Automatic backup at Fri Oct  4 18:23:32 UTC 2019
</pre>

Auf der Github-Webseite sollten wir nur Datensalat bei den Secrets sehen (bin). Im Klartext (also base64 Inhalt) sehen wir die Dateien auf unserem eigenen Rechner und auf dem Minikube-Rechner wegen git-crypt.

Im Cluster sehen wir sowas wie

```
# kubectl get jobs -n kube-system
NAME                           COMPLETIONS   DURATION   AGE
kube-state-backup-1570216080   1/1           29s        3m3s
kube-state-backup-1570216140   1/1           29s        2m3s
kube-state-backup-1570216200   1/1           29s        63s
kube-state-backup-1570216260   0/1           3s         3s
```

Sicherlich keine perfekte Loesung, aber eine automatisierte. Ueber die Variable RESOURCETYPES kann ich noch steuern, welche Resourcetypen ich backupen moechte. Eine kleine Auswahl ist schon dabei.

Credits gehen an https://github.com/pieterlange | https://www.fullstaq.com | https://twitter.com/fullstaq
