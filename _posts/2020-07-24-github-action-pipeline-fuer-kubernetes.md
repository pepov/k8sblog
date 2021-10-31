---
layout: post
tag: de
title: Github Action Pipeline fuer Kubernetes
subtitle: "Github bietet seit einiger Zeit jetzt schon das Feature `Action` an - ein CI/CD-Werkzeug, was noch bischen ein Nischendasein fristet neben den ganzen anderen Platzhirschen wie Travis, CircleCI usw. Voellig unbegruendet! Heute zeige ich Euch, wie man in 5 Minuten Github Action Pipeline erfolgreich einrichtet, um eine Applikation in einem Kubernetes-Cluster neu zu deployen."
date: 2020-07-24
background: 'images/k8s-cosmos.png'
---

Github Action ist in jedem Github-Repo zu finden und wird automatisch im Menue mit angezeigt, wenn man ein Repo erstellt. 

<img src="images/2020-07-24-1.png" width="900" height="450" />

Man kann sich GitHub Action ein bischen wie <a href="https://blog.eumelnet.de/blogs/blog8.php/schwarzer-guertel-dan-5-kubernetes-operator">Kubernetes Operator</a> vorstellen: Da ist irgendwo hinten eine Business-Logik, die das umsetzt, was weiter vorne in einer Metasprache und einer Instanz beschrieben ist.  Letztlich liegt dort ein <a href="https://github.com/gnuu-de/kubectl/blob/master/Dockerfile">Dockerfile</a>, welches ein Ubuntu-Image startet und kubectl installiert und ueber den Entrypoint aufruft.
Eine `action.yml` beschreibt, welche Aktionen vom Github-Worker ausgefuehrt werden sollen. Ganz stumpfsinnig ist das hier: Starte docker.
Die Credentials brauchen wir vom Cluster, etwa angelegt <a href="https://blog.eumelnet.de/blogs/blog8.php/neue-user-anlegen-in-kubernetes-k3s-mit-rbac">im letzten Blog-Artikel</a>. Die Cluster-URL/IP-Adresse muss natuerlich aus der Welt erreichbar sein!
Diese Datei codieren wir base64:

```shell
cat /opt/k3s-clients/gen/kube/github.kubeconfig | base64
```

Den Inhalt kopieren wir in die Zwischenablage, gehen zu Settings/Secrets in unserem Repo:

<img src="images/2020-07-24-2.png" width="900" height="450" />

Dort legen wir das neue Secret `KUBE_CONFIG_DATA` an und fuegen den Inhalt der Zwischenablage, als die base64 codierte KubeConfig ein.

Im Repo erstellen wir eine Datei:
<a href="https://github.com/gnuu-de/www/blob/master/.github/workflows/nginx.yml">.github/workflows/nginx.yml</a>

```yaml
on: 
  push: 
    paths: 
    - 'html/**'
name: Deploy nginx
jobs:
  deploy:
    name: Deploy to cluster
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
      with:
        ref: master
    - name: Deploy to cluster
      uses: gnuu-de/kubectl@master
      env:
        KUBE_CONFIG_DATA: ${{ secrets.KUBE_CONFIG_DATA }}
      with:
        args: rollout restart deployment/nginx
    - name: Verify deployment
      uses: gnuu-de/kubectl@master
      env:
        KUBE_CONFIG_DATA: ${{ secrets.KUBE_CONFIG_DATA }}
      with:
        args: get pods -l app=nginx
```

Was passiert hier?
Bei Pushs in das Verzeichnis html wird der Job "Deploy to Cluster" angeschmissen. Der laeuft auf Ubuntu (momentan wird auch nur Ubuntu als Linux-Betriebssystem unterstuetzt - deswegen brauchen wir auch ein Ubuntu-Docker-Image).
Als Environment-Variable wird `KUBE_CONFIG_DATA` uebergeben. Der Inhalt wird im Docker-Container dekodiert und als KUBECONFIG-Variable abgelegt. Der Entrypoint von dem Container, dessen Manifest als Dockerfile im Github Reopo gnuu-de/kubectl liegt, ist "kubectl". Demnach reicht als Argument "rollout restart deployment/nginx", um auf meinem Kubernetes-Cluster ein Redeployment von Nginx anzustossen, um meine Aenderungen der HTML-Seiten sichtbar zu machen, da das Deployment das Repo im <a href="https://github.com/gnuu-de/k8s/blob/master/nginx/deployment.yaml#L37-L44">InitContainer</a> auscheckt und lokal im POD abspeichert. Ein zweiter Job zeigt mit "get pods" an, ob da wirklich was passiert ist:

<img src="images/2020-07-24-3.png" width="900" height="450" />

Fertig! Bei jeder Aenderung im Repo im Verzeichnis html wird die Github Action ausgefuehrt.

Ein weiterer Anwendungsfall ist docker build & push, also das Erstellen von Images und veroeffentlichen dieser auf hub.docker.io. Ein passener Github Action findet man auf https://github.com/marketplace/actions/build-and-push-docker-images
Ein passender Workflow ist hier im Einsatz und kann so aussehen:

```yaml
on: 
  push: 
    paths: 
    - 'Dockerfile.nginx'
name: Build & Push Dockerfile.nginx
jobs:
  deploy:
    name: Docker Build & Push
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
 
      - name: Build and push image
        uses: docker/build-push-action@v1
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
          dockerfile: Dockerfile.nginx
          repository: gnuu/nginx
          tags: latest
```

`DOCKER_USERNAME` und `DOCKER_PASSWORD` werden als Secret Variable in den Settungs gespeichert (Klartext, nicht base64).


Ein paar Fallstricke:

* Die Github-Action Datei liegt im Verzeichnis .github/workflows/. Der Dateiname ist egal, der Verzeichnisname ist wichtig!
* Wie schon erwaehnt, werden nur Ubuntu Images unterstuetzt, sowie <a href="https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#jobsjob_idruns-on">Windows und Mac</a>
* Neben Docker gibt es noch Umgebungen fuer <a href="https://docs.github.com/en/actions/language-and-framework-guides">NodeJS, Python und Java</a>
* Urspruenglich war fuer das Projekt hier angedacht, die Kube-API nur ueber Wireguard zugaenglich zu machen. Die Schluessel haette man auch mit Secrets injecten koennen, leider funktioniert dann das Tunneldevice im Docker nicht. Dazu muesste man seine <a href="https://docs.github.com/en/actions/hosting-your-own-runners">eigenen Runner</a> laufen lassen, waere aber auch doppelt-gemoppelt, da die Komunikation zum Cluster sowieso schon verschluesselt ist.

Fazit: Github Action ist sehr schnell. Und sicher. Secret-Verwaltung findet nur innerhalb des Jobs statt. Es gibt einen Marktplatz, auf dem jeder seinen Action Operator veroeffentlichen kann. Github selber steuert natuerlich <a href="github.com/actions/">auch welche</a> bei. Die <a href="https://docs.github.com/en/actions/">Dokumentation</a> ist sehr umfangreich und mit diesem Beispiel hier das Ganze vielleicht jetzt etwas leichter verstaendlich. Viel Spass. Und: Action!

