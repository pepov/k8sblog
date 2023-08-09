---
layout: post
tag: en
title: Migrate Rancher Logging
subtitle: Howto migrate Rancher Logging to Kube Logging, and why
date: 2023-08-08
background: '/images/k8s-cosmos.png'
twitter: 'images/cosignwebhook.png'
author: eumel8
---

# Intro

Rancher provides an application named "Logging" in the Apps store. 

"Collects and filter logs using highly configurable CRDs. Powered by Banzai Cloud."

In fact it's Kube Logging, adapted and enriched with Rancher specific logging mechanism like RKE logging or logging for K3S.
In this blog post we will explain how to migrate from Rancher Logging to Kube Logging directly.

# Why migrate?

In the past Rancher developed his own services for logging and monitoring, version 1, controlled by the upstream cluster, with his own resources and software. It couldn't be adapted from other vendors, because it worked only with Rancher. 

In version 2 Rancher forked common Open Source projects like Kube Prometheus Stack for monitoring and Banzai Cloud for logging. The software was adapted for specific Rancher installations like RKE1, RKE2, K3S, tested and released. At the end you have astable version, integrated and running in Rancher. Only the support is limited. If you have issues, your problem will only forwarded to the Upstream community. This tests and integration takes time, where Rancher spends not so much in this sidecar services. Only after 3 or 6 months a new release will published.

The Upstream projects working much faster, new features are released in days or weeks. And you have support directly from the maintainer. If you are interested on new features and not so much on Rancher specific adaptions, you are a candidate for migration.

# Rancher Logging

Rancher Logging are shipped on two Helm charts, rancher-logging and rancher-logging-crds. With a dependency both are installed in parallel.

Search in the App store for Logging:

<img src="/images/rancher-logging-1.png" width="850" height="475" />

After select some information are presented:

<img src="/images/rancher-logging-2.png" width="850" height="475" />

In the next step some cluster specific configuration are discovered. Here K3S:

<img src="/images/rancher-logging-3.png" width="850" height="475" />

You can follow the output of the installation process:

<img src="/images/rancher-logging-4.png" width="850" height="475" />

After processing the CRDs a new menue point is created in Rancher, Logging:

<img src="/images/rancher-logging-5.png" width="850" height="475" />

Now you use resources like Logging, Flows, Outputs to setup a fluentd instance and define to ship logs to a final destination.

# Kube Logging

To install Kube Logging in Rancher you need to setup the Helm chart repository with the Logging Operator Helm chart.
Navigate to the Apps menue in Rancher:

<img src="/images/rancher-logging-6.png" width="850" height="475" />

Under Repositories you can create a new one. Choose your own name and HTTP target with the address of the Kube Logging Helm chart repository:

<img src="/images/rancher-logging-7.png" width="850" height="475" />

If the status is synced and active, some new apps should appear in the store:

<img src="/images/rancher-logging-8.png" width="850" height="475" />

# CRDs

Within Helm 3 there are still no option to upgrade CRDs. The full story is described in [hip-0011]({https://github.com/helm/community/blob/main/hips/hip-0011.md). For the migration process there are 2 option:

## Delete/Re-Create

A deinstallation of rancher-logging and rancher-logging-crd chart will not automatically remove the CRDs. But be careful, in rancher-monitoring-crd are jobs implemented with `annotations."helm.sh/hook": pre-delete`, which deletes all CRDs if the crd chart will deleted. Best case to make a backup of all resources, before start here:

```bash
% for i in `kubectl api-resources --api-group='logging.banzaicloud.io' -o name`; do kubectl get $i -A -o yaml > $i.yaml;done
```

After that you can safely remove the CRDs from the cluster:

```bash
% for i in `kubectl api-resources --api-group='logging.banzaicloud.io' -o name`; do kubectl delete crd $i;done
```
The installation of the kube-logging chart will create the new CRDs. After that you can apply the resources from the backup.

## Update

Update CRDs are often a manual process. Until the version of the CRD is `alpha` or `beta` this can be very often with changes in the same CRD. A cluster administrator can do this in the following way:

```bash
% git clone https://github.com/kube-logging/logging-operator.git
% cd logging-operator
% git checkout 4.3.0 # or the target Helm chart version what you want to install
% cd charts/logging-operator
% kubectl apply -f crds/ 
...
Warning: resource customresourcedefinitions/loggings.logging.banzaicloud.io is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.

The CustomResourceDefinition "loggings.logging.banzaicloud.io" is invalid: metadata.annotations: Too long: must have at most 262144 bytes
```

This will mostly fail because of the client-side metadata.annotations, where the completely content of the CRDs will held.

Better approach is to use the server-side flag:

```bash
% kubectl apply -f crds/ --server-side --force-conflicts
```


# Install Kube-Logging

Continue here to choose the logging-operator app in app store:

<img src="/images/rancher-logging-9.png" width="850" height="475" />

As ususally you can choose the target namespace and give them a name. Activate `Customize Helm option before start`:

<img src="/images/rancher-logging-10.png" width="850" height="475" />

The presented values.yaml show all options to configure, klick `Next` if the default settings are fine:

<img src="/images/rancher-logging-11.png" width="850" height="475" />

Ensure `Apply custom resource definition` is enabled:

<img src="/images/rancher-logging-12.png" width="850" height="475" />

After installation is complete, you have the same `Logging` menu in Rancher and can start with setup Logging, Flows, and Outputs.

Happy Logging!
