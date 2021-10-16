---
layout: post
tag: de
title: "Kubernetes Bibliothek"
date: 2019-12-20
---

Kubernetes Bibliothek
=====================

In unserer Blog Serie zu Kubernetes haben wir schon einiges gelernt - wie Kubernetes aufgebaut ist und vor allem, was man damit machen kann. Jetzt moechte man vielleicht wieder mal etwas nachschlagen oder nachlesen. Wie geht? Kubernetes Bibliothek.

<img src="/kubernetes.png" alt="Kubernetes" title="Kubernetes Logo" align="middle" width="420" height="420" />

---

Die Dokumentation zu Kubernetes befindet sich auf https://kubernetes.io/docs/home/. Die gute Nachricht als erstes: Die Dokumentation gibts auch auf Deutsch! Oben rechts auf der Webseite befindet sich ein Auswahl-Menue der Sprachen. Das Source-Repo befindet sich auf https://github.com/kubernetes/website Die Dokumente sind im Markdown-Format verfasst. Damit ist das Uebersetzen nicht so einfach wie mit RST etwa im OpenStack Projekt. Weitere Hinweise befinden sich im erwaehnten Repo.

Konzepte
--------

Die Kubernetes Dokumentation beginnt auf der linken Seite der Startseite mit Konzepten. Es wird recht umfangreich und tiefgreifend die Architektur von Kubernetes und allen beteiligten Komponenten erklaert. Man geht auf Cloud Native Applikationen ein und moechte so sicherstellen, dass man Kubernetes richtig anwendet und es auch die richtige Loesung fuer sein Problem ist.

Ausprobieren/Installieren
-------------------------

Der Mittelteil der Dokumentation beschaeftigt sich mit dem Thema: Kubernetes schnell mal ausprobieren. Dazu wurden auch hier schon viele Tools erwaehnt, sei es k3s oder Minikube. Deren Installation ist dort nochmal intensiv beschrieben. In der Sparte Tutorials gibt es Informationen zu Onlinekursen zu Kubernetes, aber auch Beispieldeployments von Wordpress-Anwendungen mit MySQL und sowas.

Referenzen
----------

DIe Referenzen sind meiner Meinung der wichtigste Teil der Dokumentation. Das Herzstueck. Die Bibliothek. Die API-Referenz erreichen wir ueber den Link https://kubernetes.io/docs/reference/kubernetes-api/api-index/  Von dort geht es auf die autogenerierte Referenz der aktuellen API, z.B. https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.17/
Alles was eine Resource kann, also genau in dieser Kubernetesversion und der API-Version, und wie sie anzusprechen ist, ist hier dokumentiert.  Hier finden wir zum Beispiel <a href="https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.17/#podspec-v1-core">PodSpec</a> und dadrin <a href="https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.17/#container-v1-core">Container Array</a>. Dieses hat dann wieder eine <a href="https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.17/#probe-v1-core">Probe Beschreibung</a>, die die einzelnen Felder beschreibt, die man in einer Container Liveness-Probe einstellen kann. In unserem <a href="https://blog.eumelnet.de/blogs/blog8.php/kubernetes-pod-job-blog">Blog-Deployment</a> sah das dann so aus:

<pre>
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
</pre>

Zum Schluss noch ein paar Tips zu kubectl. Die Client-Beschreibung finden wir auch in den Referenzen. Interessant dort das <a href="https://kubernetes.io/docs/reference/kubectl/cheatsheet/">CheatSheet</a>. Ein Punkt daraus, die Autovervollstaendigung des kubectl Clients:

```
source <(kubectl completion bash) # setup autocomplete in bash into the current shell, bash-completion package should be installed first.
echo "source <(kubectl completion bash)" >> ~/.bashrc # add autocomplete permanently to your bash shell.
```

Kubernetes entwickelt sich ziemlich schnell weiter. Man sollte in der <a href="https://kubernetes.io/docs/home/">Bibliothek</a> oben rechts immer schauen, fuer welche Version die Dokumentation gueltig ist.
