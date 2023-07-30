---
layout: post
tag: de
title: "Kubernetes Backup mit Velero und Minio"
subtitle: "In <a href='https://blog.eumelnet.de/blogs/blog8.php/backup-mein-kubernetes-aber-sicher'>Backup mein Kubernetes, aber sicher</a> haben wir uns schon mal mit dem Thema Backups auseinandergesetzt. Es ging dabei um Kubernetes Resourcen, die irgendwie mal angelegt worden sind, aber man weiss nicht mehr genau wie und von wem. Zeit sich eine neue Backuploesung anzusehen, mit der auch Volumes gesichert werden koennen, am besten nach S3-Speicher"
date: 2020-07-09
background: '/images/k8s-cosmos.png'
---

S3 Speicher von den verschiedenen Cloud-Anbietern ist gar nicht mal so teuer. Ein Blick lohnt sich nach AWS, Azure & Co. Bei <a href="https://aws.amazon.com/de/s3/pricing/">AWS</a> kostet One Zone oder Standard S3 Daten 0,01 USD/GB pro Monat. Bei AWS Glacier, mit groesseren Rueckspeicherzeiten sogar nur 0.0045 USD/GB. Fuer Backups sehr gut geeignet. 
Wer S3 Daten dennoch lokal vorhalten will, kann sich mit <a href="https://min.io/">Minio</a> selber einen AWS-kompatiblen S3-Dienst basteln. Die Installation im Kubernetes-Cluster ist denkbar einfach:

```shell
kubectl apply -f https://raw.githubusercontent.com/minio/minio-operator/master/examples/minioinstance.yaml
```

Wer keinen Ingress oder externen Loadbalancer hat, kann den Dienst ueber NodePort exposen:

```shell
kubectl patch  service minio-service -p '{"spec":{"type":"NodePort"}}'
```

<a href="https://velero.io/">Velero</a> ist ein Backuptool von VM-Ware fuer Kubernetes. Man kann sich den Client auf dem Rechner installieren und ueber KUBECONFIG env Variable auf den Kubernetes Cluster zugreifen:

```shell
wget https://github.com/vmware-tanzu/velero/releases/download/v1.4.0/velero-v1.4.0-linux-amd64.tar.gz
tar xvfz velero-v1.4.0-linux-amd64.tar.gz
cp velero-v1.4.0-linux-amd64/velero /usr/local/bin/
```

<a href="https://restic.net/">Restic</a> ist ein weiteres Programm zur Verwaltung von Kubernetes Volumes, welches in Velero mit integriert wird. Auch hier gibt es ein Commandline-Tool:

```shell
wget https://github.com/restic/restic/releases/download/v0.9.6/restic_0.9.6_linux_amd64.bz2
bunzip2 restic_0.9.6_linux_amd64.bz2
chmod +x restic_0.9.6_linux_amd64
mv restic_0.9.6_linux_amd64 /usr/local/bin/restic
```

Wir erstellen eine Datei credentials-velero

```shell
[default]
 aws_access_key_id = minio
 aws_secret_access_key = minio123
```

Und tackern alles zusammen, sprich: installieren den Velero-Server im Kubernetes mit restic Erweitung und Minio-S3 Backend. Vorher legen wir noch den S3-Bucket "backup" im Minio an - natuerlich bequem ueber die Webseite, die wir mit dem NodePort exposed haben:

```shell
 velero install \
     --provider aws \
     --plugins velero/velero-plugin-for-aws:v1.0.0 \
     --bucket backup \
     --secret-file ./credentials-velero \
     --use-volume-snapshots=true \
     --use-restic \
     --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://10.43.38.76:9000
```

Die s3Url ist der interne Service-Endpunkt von Minio. So verlaesst die Traffic nicht den Cluster.

Wir legen noch eine Kubernetes Resource Volumesnapshot-Location an:

```yaml
---
apiVersion: velero.io/v1
kind: VolumeSnapshotLocation
metadata:
  name: minio
  namespace: default
spec:
  provider: openebs.io/cstor-blockstore
  config:
    bucket: backup
```

Nehmen wir mal an, wir haben eine App MySQL installiert mit einem PVC "data", welches wir komplett backupen wollen. Zuerst brauch der POD eine Annotation, damit Velero etwas von dem PVC mitkriegt:

```shell
kubectl  annotate pod/mysql-5cbf8684cc-hwqq5 backup.velero.io/backup-volumes=data
```

Und schon kanns losgehen:

```shell
velero backup create mysql-backup --selector app=mysql
```

Die Daten sollen alle nach Minio gesichert werden: In den Unterordner backups die Kubernetes Metadaten und in restic die Volumedaten. 

Wiederherstellen geht natuerlich auch:

```shell
velero restore create --from-backup mysql-backup
```

Oder wenn es nur die Volumes sein sollen:

```shell
velero restore create --from-backup mysql-backup  --include-resources persistentvolumeclaims,persistentvolumes
```

Das wars!
Zusatzfunktion: Volumesnapshots erstellen, etwa von OpenEBS Volumes:

```yaml
apiVersion: volumesnapshot.external-storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: snapshot-mysql
  namespace: default
spec:
  persistentVolumeClaimName: mysql
```

Und auch noch den Hinweis, die Backups durch Zurueckspielen zu ueberpruefen!!!
