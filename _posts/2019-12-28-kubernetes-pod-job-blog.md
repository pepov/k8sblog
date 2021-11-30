---
layout: post
tag: de
title: "Kubernetes Pod, Job, Blog"
subtitle: "Heute wollen wir mit Kubernetes eine Anwendung erstellen. Wie waere es mit einem Blog? Der Blog sollte den Inhalt als kompatiblen Source-Code zur Verfuegung haben. Ausserdem soll der Blog schnell aufgebaut werden und skalieren. Vorher schauen wir uns ein paar Kubernetes Resourcen an Pod, Job, Blog"
date: 2019-12-28
background: '/images/k8s-cosmos.png'
---

Voraussetzung
-------------

Laufender Kubernetes Cluster, erstellt etwa mit den <a href="https://blog.eumelnet.de/blogs/blog8.php/kubernetes-install-quickies">Install Quickies</a>

POD
---

Der POD ist die kleinste logische EInheit im Kubernetes. Er behinhaltet einen oder mehrere Container, in denen zum Beispiel ein Webserver laeuft. PODs sind hinter Services zusammengefasst und werden durch Deployments erstellt. Kubernetes bietet dazu sogar eine Autofunktion an.

```shell
# kubectl create deployment blog --image eumel8/nginx-none-root
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2019-12-28T19:01:15Z"
  generation: 1
  labels:
    app: blog
  name: blog
  namespace: default
  resourceVersion: "674312"
  selfLink: /apis/apps/v1/namespaces/default/deployments/blog
  uid: 98c97ac1-b6a5-4e2f-9a9f-e1b8b6e38f4e
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: blog
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: blog
    spec:
      containers:
      - image: eumel8/nginx-none-root
        imagePullPolicy: Always
        name: nginx-none-root
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  availableReplicas: 1
  conditions:
  - lastTransitionTime: "2019-12-28T19:01:23Z"
    lastUpdateTime: "2019-12-28T19:01:23Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2019-12-28T19:01:15Z"
    lastUpdateTime: "2019-12-28T19:01:23Z"
    message: ReplicaSet "blog-7b668d7956" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 1
  readyReplicas: 1
  replicas: 1
  updatedReplicas: 1
```

Dieses Kommando erstellt von dem Docker Image<a href="https://hub.docker.com/r/eumel8/nginx-none-root"> nginx-none-root</a> ein Deployment, welches wiederum einen POD startet. In diesem POD laeuft ein Container mit dem Nginx-Webserver auf Port 8080. Das spezielle none-root-Image wurde gewaehlt, da es grundsaetzlich keine gute Idee ist, Container als root laufen zu lassen. Es gibt sogar Kubernetes Cluster, die dies mit einer Pod Security Policy (PSP) verbieten.

JOB
---

Mit einem Kubernetes JOB koennen wir im Kubernetes Cluster einen Task ausfuehren lassen.

```shell
# kubectl create job blogjob --image  eumel8/python-none-root
```


Auch hier wird ein <a href="https://hub.docker.com/r/eumel8/python-none-root">none-root-Image</a> verwendet, diesmal mit Python als Base-Image. Aber das Erstellen des PODs fuer diesen Job wird nicht klappen:

```
NAME                             READY   STATUS                       RESTARTS   AGE
blogjob-9j7f5                    0/1     CreateContainerConfigError   0          5s
```

Die Fehlermeldung heisst in etwa: <pre> Error: container has runAsNonRoot and image has non-numeric user (appuser), cannot verify user is non-root</pre>. Das ist richtig. Das Image laeuft als appuser und der POD kennt diesen User nicht und brauch eine UserID, damit er diese zuordnen kann. Dazu brauchen wir eine Spec im securityContext

```yaml
      securityContext:
        fsGroup: 1000
        runAsUser: 1000
        runAsGroup: 1000
```

Ausserdem sollte der Job auch irgendwas tun. Dazu brauchen wir eine command Definition und ggf. Argumente dazu. Hier der Inhalt einer minimalen job.yaml Datei:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  labels:
    job-name: blogjob
  name: blogjob
spec:
  backoffLimit: 6
  completions: 1
  parallelism: 1
  template:
    spec:
      securityContext:
        fsGroup: 1000
        runAsUser: 1000
        runAsGroup: 1000
      restartPolicy: Never
      containers:
      - image: eumel8/python-none-root
        imagePullPolicy: Always
        name: blogjob
        command: ["sh","-c"]
        args: ["python --version"]
```

```shell
# kubectl create -f job.yaml
```

Der Job erstellt einen Pod, der dann das Kommando ausfuehrt. Das geht ziemlich schnell.

```shell
# kubectl logs -l job-name=blogjob
Python 3.7.3
```

Der Job ist also schon gelaufen, hat <pre>python --version</pre> gestartet und im Log-Output sehen wir das Ergebnis.

BLOG
----

Okay, wie koennen wir daraus einen Blog bauen? Nehmen wir also an, wir haben ein <a href="https://github.com/eumel8/blog/tree/master/source">Github-Repo</a> mit RST Dateien, oder MD geht auch. Dazu gibt es einige Konfigurationsdateien fuer <a href="http://www.sphinx-doc.org">Sphinx</a>, die aus den einzelnen Dateien zum Beispiel eine Webseite bauen. Das passiert ueber das Automatisierungstool <a href="https://pypi.org/project/tox/">tox</a>.  Das sieht so aus, als dass wir das mit einem JOB machen koennten. Im python-none-root-Image sind schon alle erforderlichen Programme installiert. Wir muessen nur das git-repo mit den RST files clonen und koennen unsere HTML-Seite bauen.

```shell
git clone https://github.com/eumel8/blog.git /tmp/repo && cd /tmp/repo && tox -edocs
```

Das Ergebnis muessen wir nur irgendwie ausserhalb des PODs abspeichern. Dazu erstellen wir einen PersistentVolumeClaim (PVC) und mounten diesen im JOB. Zusammen sieht das so aus:

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: repo-volume
spec:
  accessModes:
  - ReadWriteOnce
  dataSource: null
  resources:
    requests:
      storage: 1Gi
  storageClassName: sata
  volumeMode: Filesystem
---
apiVersion: batch/v1
kind: Job
metadata:
  labels:
    job-name: blogjob
  name: blogjob
spec:
  backoffLimit: 6
  completions: 1
  parallelism: 1
  template:
    spec:
      securityContext:
        fsGroup: 1000
        runAsUser: 1000
        runAsGroup: 1000
      restartPolicy: Never
      containers:
      - image: eumel8/python-none-root
        imagePullPolicy: Always
        name: blogjob
        command: ["sh","-c"]
        args: ["git clone https://github.com/eumel8/blog.git /tmp/repo &amp;&amp; cd /tmp/repo &amp;&amp; tox -edocs &amp;&amp; cp -r  html/* /tmp/blog"]
        volumeMounts:
        - mountPath: /tmp/blog
          name: repo-volume
      volumes:
        - name: repo-volume
          persistentVolumeClaim:
            claimName: repo-volume
```

Damit wird einmalig das Blog-Content gebaut und auf das Volume kopiert. Leider gibt es keine JOB-Editier- oder Neustartfunktion. Um Aenderungen im Blog zu verarbeiten, muesste man den JOB also immer wieder loeschen und neu anlegen, damit er neu gestartet wird. Dazu bietet Kubernetes auch den CRONJOB an. Ein passender Crronjob fuer den Bau unseres Blogs saehe etwa so aus:

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  labels:
    job-name: blogcronjob
  name: blogcronjob
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          securityContext:
            fsGroup: 1000
            runAsUser: 1000
            runAsGroup: 1000
          containers:
          - image: eumel8/python-none-root
            imagePullPolicy: Always
            name: testjob
            command: ["sh","-c"]
            args: ["git clone https://github.com/eumel8/blog.git /tmp/repo &amp;&amp; cd /tmp/repo &amp;&amp; tox -edocs &amp;&amp; cp -r  html/* /tmp/blog"]
            volumeMounts:
            - mountPath: /tmp/blog
              name: repo-volume
          restartPolicy: Never
          volumes:
            - name: repo-volume
              persistentVolumeClaim:
                claimName: repo-volume
```

Jede Minute wird also ein POD gestartet, der das BLOG neu baut und auf das Volume kopiert. Zugegeben, das ist nicht sehr elegant. Man wuerde lieber einen Webhook bauen, der von Github angesteuert wird, sobald wir Aenderungen in das Repo pushen. Aber das wuerde an dieser Stelle zu weit fuehren und ich wollte eigentlich bloss Job- und Cronjob-Funktionen in Kubernetes erklaeren.

Und wie erblickt unser BLOG nun das Licht der Welt? Mit einem POD, erstellt durch ein Deployment, welches das Volume von oben gemountet hat und den HTML-Content im Nginx ausliefert.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: blog
  name: blog
  namespace: default
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: blog
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: blog
    spec:
      containers:
      - image: eumel8/nginx-none-root
        imagePullPolicy: Always
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /blog/html/index.html
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 3
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        name: nginx-none-root
        resources: {}
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: repo-volume
      dnsPolicy: ClusterFirst
      restartPolicy: Always
```

Zu beachten ist die <pre>livenessProbe</pre> in den Container Specs aus der Reihe der <a href="https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes">Probes</a>. Nur wenn vom Webserver mit dem gemountetem Volume ein HTTP Status 200 zurueckkommt, ist der POD am Leben und wird so zum Beispiel in einen Service aufgenommen. Achja, den brauchen wir natuerlich zum Ausliefern unseres BLOGs, zusammen mit einem Ingress Dienst. Die vollstaendige Liste von Dateien findet man auf https://github.com/eumel8/blog/tree/master/kubernetes


<img src="/blog/images/2019-12-28_2_.png" width="900" height="450" />

Skalieren kann ich den Blog mit Erhoehen des Replicaset

```shell
# kubectl scale --replicas=3 deployment.apps/testdeployment
deployment.apps/testdeployment scaled

# kubectl  get ep blog-service
NAME           ENDPOINTS                          AGE
blog-service   10.42.5.214:8080,10.42.5.30:8080   27h

# kubectl  get ep blog-service
NAME           ENDPOINTS                                           AGE
blog-service   10.42.5.214:8080,10.42.5.220:8080,10.42.5.30:8080   27h

```

Man hat vielleicht schon gemerkt, das teure Volume vom PVC braeuchte man eigentlich gar nicht, man koennte die HTML-Daten des Blogs auch im Nginx-Pod halten. Dazu muesste man sie nur mit tox vor der POD-Erzeugung erstellen. Statt des JOB bietet sich vielleicht <a href="https://kubernetes.io/docs/concepts/workloads/pods/init-containers/#init-containers-in-use">InitContainers</a> an, wenn im InitContainer python mit tox laeuft und danach im Nginx-Container der Webserver. Aber das nur als Idee am Ende. Viel Spass mit Sphinx-Blog in Kubernetes.
