# relsre-metrics-prom-docker-android

## Overview

A docker image that runs on Mozilla's GCP v2 Kubernetes cluster.

Runs a Prometheus server that displays metrics about the `proj-autophone` TC provisioner.

For Bitbar hardware Android devices this includes:
- configured device counts per workerType (based on mozilla-bitbar-devicepool's config/config.yaml).
- information from the Bitbar API about offline status per workerType.

Bitbar information is currently provided by `prom_report.py` script in `android-tools/worker_health` (https://github.com/mozilla-platform-ops/android-tools/tree/master/worker_health). This data was previously provided by `influx_logger.py`.
- Sentry project: https://mozilla.sentry.io/settings/projects/relsre-metrics-prom-docker-android/

## development

#### running locally (gcp)

```bash
# open a interactive docker container
./docker_build
./docker_run
# view `http://localhost:8000/metrics` in a browser
```

# testing scripts
```bash
./docker_run /bin/bash
$ /opt/venv/bin/python3 /etc/android-tools/worker_health/prom_report.py
```

# kill leftover containers

```bash
docker stop $(docker ps | grep moz_telegraf_gcp_android | cut -f 1 -d ' ')
```

## deployment

### build image and push to artifact registry

```shell
# build the image
docker buildx build --platform linux/amd64 -t relsre-metrics-android .
# if things aren't getting pulled in, run with `--no-cache`:
#   docker buildx build --no-cache --platform linux/amd64 -t relsre-metrics-android .

# note the resulting image sha1 and use it below in the `docker tag` command
docker images | head

# replace D_VERSION with next version (check artifact registry, link below)
#   - https://console.cloud.google.com/artifacts/docker/moz-fx-relsre-metrics-prod/us/relsre-metrics-prod/relsre-metrics-android?authuser=1&invt=AboNNQ&project=moz-fx-relsre-metrics-prod)
export D_VERSION=""
export D_SHA=""
docker tag $D_SHA us-docker.pkg.dev/moz-fx-relsre-metrics-prod/relsre-metrics-prod/relsre-metrics-android:$D_VERSION
docker push us-docker.pkg.dev/moz-fx-relsre-metrics-prod/relsre-metrics-prod/relsre-metrics-android:$D_VERSION
```


### create and modify secrets

See .https://mozilla-hub.atlassian.net/wiki/spaces/SRE/pages/27919985/GKE+Cluster+Secrets+Management.

```bash

cd ~/git/
git clone https://github.com/mozilla-it/gsm-editor
```

```bash
# authenticate
gcloud container clusters get-credentials webservices-high-prod --region=us-west1 --project moz-fx-webservices-high-prod

# set the context (can use kubectx also)
#   from ntade:
#     so webservices-high-nonprod would be for i.e. dev and stage, and webservices-high-prod would be for prod
kubectl config use-context webservices-low-prod_us-west1

# start the tunnel (will use current context to infer details)
gcp-bastion-tunnel

# set the namespace (can use kubens also)
# kubectl config set-context --namespace=relsre-metrics????
#
# https://stackoverflow.com/questions/61171487/what-is-the-difference-between-namespaces-and-contexts-in-kubernetes
kubens relsre-metrics-prod

#
cd ~/git/gsm-editor
./gsm.py

# GCP Console for this:
#   https://console.cloud.google.com/security/secret-manager?referrer=search&authuser=1&project=moz-fx-relsre-metrics-prod

# list secret names (`app` is default)
./gsm.py names -p moz-fx-relsre-metrics-prod -e prod

# show versions of the secret
./gsm.py list -p moz-fx-relsre-metrics-prod -e prod

# edit default secret
./gsm.py edit -p moz-fx-relsre-metrics-prod -e prod
```


### helm deploy

#### first run

```shell
# Helm charts and terraform are located at https://github.com/mozilla-it/webservices-infra/tree/main/relsre-metrics.
cd ~/git
git clone https://github.com/mozilla-it/webservices-infra.git

# install helm diff plugin
helm plugin install https://github.com/databus23/helm-diff

# install gcp-bastion-tunnel
#   see https://github.com/mozilla-it/gcp-bastion-tunnel/tree/main

```

#### every run

```shell
# cd to helm directory
cd ~/git/webservices-infra/relsre-metrics/k8s/relsre-metrics

# authenticate
gcloud container clusters get-credentials webservices-high-prod --region=us-west1 --project moz-fx-webservices-high-prod

# set the context (can use kubectx also)
#   from ntade:
#     so webservices-high-nonprod would be for i.e. dev and stage, and webservices-high-prod would be for prod
kubectl config use-context webservices-low-prod_us-west1

# set the namespace (can use kubens also)
# kubectl config set-context --namespace=relsre-metrics????
#
# https://stackoverflow.com/questions/61171487/what-is-the-difference-between-namespaces-and-contexts-in-kubernetes
kubens relsre-metrics-prod

# start the tunnel (will use current context to infer details)
gcp-bastion-tunnel

cd ~/git/webservices-infra/relsre-metrics/k8s/relsre-metrics

# view diff between running and local
helm diff upgrade --install relsre-metrics . -f values-prod.yaml --namespace=relsre-metrics-prod

# apply it if it looks good
helm upgrade --install relsre-metrics . -f values-prod.yaml --namespace=relsre-metrics-prod --debug

# view deployments
helm list --namespace relsre-metrics-prod

# inspect pod for errors
kubectl describe pod relsre-metrics-telegraf-queues  -n relsre-metrics-prod

# view quotas
kubectl get resourcequota -n relsre-metrics-prod

# view cpu/ram usage for running pods
kubectl top pod

# misc
kubectl get pods
kubectl get services

# tunneling to a pod to check a web page
kubectl port-forward relsre-metrics-telegraf-queues-5c5647dfd5-hpwvp 8000
# then surf to http://localhost:8000/metrics

```
