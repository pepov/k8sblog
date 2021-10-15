Kubernetes in bunt - Dashboard Installation als Einzeiler
=========================================================

Manche fuehlen sich erst richtig wohl, wenn sie befreit sind vom Zwang jeglicher grafischer Benutzeroberflaechen. Andere fuehlen sich dann etwas verloren, wenn es sowas nicht gibt. Aber auch da gibt es bei Kubernetes eine Loesung: Das Kubernetes Dashboard:

![](</kubernetes.png>)

---

Installation:
-------------


:    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta4/aio/deploy/recommended.yaml

Fertig!
Natuerlich sollte man vorher mal in die Datei reinschauen, die man so arglos uebers Netz installiert. Was ist drin: Es wird ein Namespace <ins>kubernetes-dashboard</ins> erstellt, User und Rechte fuer <a href="https://kubernetes.io/docs/reference/access-authn-authz/rbac/">RBAC </a> installiert, und zum Schluss natuerlich die Applikation.

Damit wir von aussen auf den Dienst zugreifen koennen, brauchen wir wieder Ingress, wie in <a href="https://blog.eumelnet.de/blogs/blog8.php/letsencrypt-gesicherte-webseiten-in-kubernetes-mit-cert-manager">vorherigen Artikel</a> beschrieben. 

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    external-dns.alpha.kubernetes.io/target: 80.158.36.239
    nginx.ingress.kubernetes.io/backend-protocol: HTTPS
    certmanager.k8s.io/issuer: letsencrypt-production
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"extensions/v1beta1","kind":"Ingress","metadata":{"annotations":{"external-dns.alpha.kubernetes.io/target":"80.158.36.239","kubernetes.io/ingress.class":"nginx"},"name":"nginx","namespace":"kubernetes-dashboard"},"spec":{"rules":[{"host":"minikube.otc.eumel.de","http":{"paths":[{"backend":{"serviceName":"kubernetes-dashboard","servicePort":1443}}]}}]}}
    kubernetes.io/ingress.class: nginx
  creationTimestamp: "2019-09-29T21:42:19Z"
  generation: 1
  name: nginx
  namespace: kubernetes-dashboard
  resourceVersion: "1929459"
  selfLink: /apis/extensions/v1beta1/namespaces/kubernetes-dashboard/ingresses/nginx
  uid: c4be5d0f-86e5-45e2-9e5d-0b243b08c58a
spec:
  rules:
  - host: dashboard.otc.eumel.de
    http:
      paths:
      - backend:
          serviceName: kubernetes-dashboard
          servicePort: 1443
  tls:
  - hosts:
    - dashboard.otc.eumel.de
    secretName: dashboard-otc-eumel-de-tls
status:
  loadBalancer:
    ingress:
    - ip: 192.168.1.170
```

Erlaeuterungen:
---------------

Annotations:

Trage A-Records von allen Hostnamen mit dieser Ip-Adresse als Ziel ein (vgl. <a href="https://blog.eumelnet.de/blogs/blog8.php/kubernetes-openstack-teil-1-external-dns-mit-designate">hier</a>):

<pre>
    external-dns.alpha.kubernetes.io/target: 80.158.36.239
</pre>

Der Dashboard-Service wird schon mit HTTPS und selbstsignierten Zertifikaten ausgeliefert. Wenn wir HTTPS auf HTTPS im NGINX verbinden wollen, brauchen wir diese Option:

<pre>
    nginx.ingress.kubernetes.io/backend-protocol: HTTPS
</pre>

Zertifikat-Aussteller fuer alle TLS Hostnamen ist wieder LetsEncrypt (vgl <a href="https://blog.eumelnet.de/blogs/blog8.php/letsencrypt-gesicherte-webseiten-in-kubernetes-mit-cert-manager">hier</a>)

<pre>
    certmanager.k8s.io/issuer: letsencrypt-production
</pre>

Alle anderen Eintraege sollten selbsterklaerend sein. 
Wenn man keinen Ingress nutzen moechte/kann, gibt es die Moeglichkeit der <a href="https://kubernetes.io/docs/concepts/services-networking/service/#external-ips">ExternalIP und NodePorts</a>.

Zuerst ueberprueft man, welche NodePorts zur Verfuegung stehen:

```
kubectl describe pod kube-apiserver-minikube -n kube-system | grep service-node-port
      --service-node-port-range=80-35000
```

Nun machen wir aus dem ClusterIP-Services einen LoadBalancer-Service unter Verwendung eines Ports aus der Port-Range:

<pre>
apiVersion: v1
kind: Service
metadata:
  annotations:
    external-dns.alpha.kubernetes.io/target: joomla01.otc.eumel.de
  creationTimestamp: "2019-09-23T19:20:26Z"
  labels:
    app: joomla
    chart: joomla-6.1.12
    heritage: Tiller
    release: joomla
  name: joomla
  namespace: joomla
  resourceVersion: "1798208"
  selfLink: /api/v1/namespaces/joomla/services/joomla
  uid: 936599f3-5ea5-4ea7-8ba3-acb854bb557d
spec:
  clusterIP: 10.107.39.136
  externalIPs:
  - 80.158.36.239
  externalTrafficPolicy: Cluster
  ports:
  - name: http
    nodePort: 80
    port: 80
    protocol: TCP
    targetPort: http
  - name: https
    nodePort: 2443
    port: 2443
    protocol: TCP
    targetPort: https
  selector:
    app: joomla
  sessionAffinity: None
  type: LoadBalancer
</pre>

Einige Ports koennten schon belegt sein durch andere Services, weswegen dann das https auf Port 2443 laufen muss. Macht nicht wirklich Sinn, illustriert aber die Anwendung von Node-Ports.

Verwirrt? Hier ist ein interessanter Blogbeitrag zu <a href="https://medium.com/google-cloud/kubernetes-nodeport-vs-loadbalancer-vs-ingress-when-should-i-use-what-922f010849e0">Ingress, LoadBalancer und ClusterIP</a>

Wenn unser Dashboard-Service dann per Ingress oder Node-Port das Licht der Welt erblickt hat, sollten wir den Startbildschirm sehen:

<img src="/2019-10-02_9_.png" width="900" height="450" />


Zum Login brauchen wir eine Kube-Config, was aber bei fern installierten Rechnern und dort laufenden Minikube kompliziert sein kann, wenn nicht alle Informationen in einer Datei vorhanden sind.
Oder wir benutzen einen Bearer-Token. Den koennen wir uns selbst erstellen oder einen bereits vorhanden nutzen, wie etwa den von Tiller, der im Cluster weitreichende Rechte zwecks Softwareinstallation hat:

```
kubectl get secret $(kubectl get serviceaccount tiller -n kube-system -o json | jq -r '.secrets[].name') -n kube-system -o yaml | grep "token:" | awk {'print $2'} |  base64 -d -w0
eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJ0aWxsZXItdG9rZW4tbWo3
```

Startbildschirm nach dem Login:

<src="/2019-10-02_8_.png" width="900" height="450" />

Einige Informationen zu unserer Joomla-Installation:

<src="/2019-10-02_2_.png" width="900" height="450" />

Editieren von Resourcen im Browser:

<img src="/2019-10-02_7_.png" width="900" height="450" />

Mit <code>minikube dashboard</code> gaebe es zwar einen weiteren Einzeiler zur Dashboard Installation. Jedoch setzt diese die Installation auf dem eigenen Rechner mit X-Windows o.ae. voraus.
