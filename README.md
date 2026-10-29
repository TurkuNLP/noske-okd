# noske-okd

OKD (OpenShift) deployment of [NoSketchEngine](https://nlp.fi.muni.cz/trac/noske) for
[TurkuNLP](https://turkunlp.org) / [Fin-CLARIAH](https://www.kielipankki.fi/organization/fin-clariah/).

## What this is

NoSketchEngine is a corpus query web interface (manatee-open + bonito-open + crystal-open).
This repo contains the OKD image patch and Kubernetes manifests to deploy it on
[LUMI-K](https://docs.lumi-supercomputer.eu/runjobs/webui/), CSC's OpenShift cluster.

## Contents

```
Dockerfile.okd          OKD non-root image patch (based on ghcr.io/acdh-oeaw/noske-ubi9/noske)
run_lighttpd.sh         Patched entrypoint (arbitrary UID support)
manifests/
  01-pvc.yaml           PersistentVolumeClaim — CephFS corpus storage
  02-configmap.yaml     manatee registry config
  03-deployment.yaml    Pod spec with init container + main NSE container
  04-service.yaml       ClusterIP service
  05-route.yaml         OKD Route with edge TLS
docs/
  deployment-notes.md   Deployment log, runbook and open steps
  verification-2026-06.md   Dev verification record (2026-06-03)
```

## Image

Patched image: `ghcr.io/tuomaslundberg/noske-okd:latest` (public, linux/amd64)

Base image: `ghcr.io/acdh-oeaw/noske-ubi9/noske` (UBI9, Oct 2024)

Patch: `chown 0:0 + chmod g=u` on all runtime-writable dirs; `/etc/passwd` group-writable;
entrypoint injects current UID into passwd at startup. Required for OKD's arbitrary-UID policy.

## Deployment

```bash
oc apply -f manifests/01-pvc.yaml
oc apply -f manifests/02-configmap.yaml
oc apply -f manifests/03-deployment.yaml
oc apply -f manifests/04-service.yaml
oc apply -f manifests/05-route.yaml
```

Production uses `manifests/prod/` (see the runbook). See `docs/deployment-notes.md` for cluster-specific constraints and how to load corpora.
