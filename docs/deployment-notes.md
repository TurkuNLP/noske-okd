# NoSketchEngine on LUMI-K — deployment log and runbook

NoSketchEngine (a corpus query web interface) deployed on LUMI-K as a Fin-CLARIAH
deliverable: externally accessible, persistent, maintainable.

| | Dev | Prod |
|---|---|---|
| Namespace | `turkunlp-noske-dev` (throwaway; LUMI project 462000999) | `turkunlp-noske-prod` (LUMI project 462001491, lifetime to 2027-05-28) |
| URL | `https://noske-turkunlp-noske-dev.apps.lumi-k.eu` | `https://noske-turkunlp-noske-prod.apps.lumi-k.eu` |
| Manifests | `manifests/` | `manifests/prod/` |
| Corpus | `hplt_toy` (placeholder) | `hplt_toy` (placeholder) |

---

## Log

### 2026-04-23 — platform verified, non-root image patch

- `oc` CLI authenticated to `https://api.v1.lumi-k.eu:6443`.
- Storage classes: `rook-ceph-fs` (CephFS, RWX), `rook-ceph-block` (default, RWO), `lvms-local` (RWO, no replication).
- Dev namespace `turkunlp-noske-dev` created. A LimitRange is applied automatically: container defaults 1 CPU / 2 GiB RAM; max 8 CPU / 16 GiB RAM / 100 GiB per PVC. No ResourceQuota.
- The admission webhook requires `lumi_project: XXXXXXXX` in the namespace description.
- Image patch (`Dockerfile.okd` + `run_lighttpd.sh`): `chown 0:0 + chmod g=u` on all runtime-writable dirs, `chmod g=u /etc/passwd`, UID injection in the entrypoint. Validated locally with `docker run --user 12345`; pushed to `ghcr.io/tuomaslundberg/noske-okd:latest`.

### 2026-04-27 — all five manifests applied; dev instance live

- `01-pvc.yaml` (10 GiB CephFS PVC for corpus data), `02-configmap.yaml` (manatee registry), `03-deployment.yaml` (init container + NSE container), `04-service.yaml` (ClusterIP, port 8080), `05-route.yaml` (edge TLS route).
- Issues resolved:
  - `runAsUser: 1001` rejected — the namespace UID range is `[1001230000, 1001239999]`. Fix: drop `runAsUser`, keep `runAsNonRoot: true`.
  - CPU limit:request ratio is capped at 5× — request raised from 500m to 1 CPU.
  - Image built on arm64 while the cluster is x86_64 — build with `--platform linux/amd64`.
  - GHCR package was private — made public.
  - `BROWSER_URL_BONITO` must be the base URL only; crystal appends `/bonito/run.cgi` itself.

### 2026-06-03 — real corpus loading via LUMI-O

- `scripts/to_vert.py` converts HPLT v3 JSONL to manatee vertical (whitespace tokenisation; doc metadata `id`, `lang`, `url`). HPLT v3 JSONL: each line is a shard record `{filename, documents: [...]}`; doc fields are `u` (URL), `lang` (list, primary first), `id`, `text`.
- `scripts/compile_and_upload.sh`: vertical → `compilecorp` in Singularity → rclone upload to LUMI-O.
- `registry/hplt_toy` is the registry source of truth, mirrored into the ConfigMap.
- `Dockerfile.okd` installs rclone (used by the init container). The init container downloads the pre-compiled corpus from LUMI-O, with credentials from the `lumi-o-credentials` Secret (`manifests/00-secret.yaml`, gitignored).
- On LUMI-C, `singularity` is at `/usr/bin/singularity` (no module needed; `apptainer` is not available). `oc` is installed in `~/.local/bin/`, so manifests can be applied from LUMI directly.
- Dev verified: 6/6 checks pass (`docs/verification-2026-06.md`). 4.1 M words; ~59 % of documents are `fin_Latn`, as expected for a multilingual HPLT shard.

### 2026-06-23 — production namespace live

- `turkunlp-noske-prod` under LUMI project 462001491, serving `hplt_toy`; 6/6 checks pass.
- LUMI-O bucket `turkunlp-noske-corpora` under 462001491; `hplt_toy` copied over from the 462000999 bucket.
- `oc new-project --description` satisfies the admission webhook; no web console needed.
- Prod Secret `manifests/prod/00-secret.yaml` is gitignored. Its namespace field must be the prod namespace — a copy of the dev template still says `turkunlp-noske-dev`.

---

## Runbook

### Authenticate
`oc` tokens expire after about 24 h. LUMI-K uses SSO, so re-authenticate from the OKD web console: username → *Copy login command*.

### Build and push the image
```bash
docker build --platform linux/amd64 -t ghcr.io/tuomaslundberg/noske-okd:latest -f Dockerfile.okd .
docker push ghcr.io/tuomaslundberg/noske-okd:latest
```

### LUMI-O credentials
Generate keys for the project at my.csc.fi, then on LUMI-C:
```bash
module load lumio
lumio-conf        # writes ~/.config/rclone/rclone.conf with remote lumi-<project>-private
rclone lsd lumi-462001491-private:
```
Fill the same keys into `manifests/prod/00-secret.yaml` (gitignored) before applying it.

### Deploy from scratch
```bash
oc apply -f manifests/prod/00-secret.yaml
oc apply -f manifests/prod/01-pvc.yaml
oc apply -f manifests/prod/02-configmap.yaml
oc apply -f manifests/prod/03-deployment.yaml
oc apply -f manifests/prod/04-service.yaml
oc apply -f manifests/prod/05-route.yaml
```

### Add a corpus
1. `scripts/compile_and_upload.sh` — generate the vertical, compile, upload to LUMI-O.
2. Add the registry file under `registry/`, mirror it into `manifests/prod/02-configmap.yaml`, and add the corpus name to `CORPLIST`.
3. `oc apply -f manifests/prod/02-configmap.yaml -f manifests/prod/03-deployment.yaml`
4. `oc rollout restart deployment/noske -n turkunlp-noske-prod`
5. Run the verification checklist below.

The init container skips the download when the corpus is already on the PVC, so restarts do not re-copy.

### Verification checklist
Open the instance URL and check:
1. **Page loads** — the Crystal UI loads without errors and shows the corpus selector.
2. **Corpus selector** — the corpus is listed and opens.
3. **Basic search** — `[word="ja"]` in CQL returns concordance lines of real text.
4. **Frequency list** — the wordlist returns plausible high-frequency words.
5. **Structural attribute filter** — `[word=".*"] within <doc lang="fin_Latn"/>` returns a subset, not 100 %, of the corpus (confirms `doc`/`lang` are indexed).
6. **Errors** — no application errors, broken UI elements or empty-corpus indicators.

---

## Open steps

- Load the real corpora into prod (see *Add a corpus*).
- Custom domain + TLS, if needed.
- PVC deletion protection.
- Fin-CLARIAH deliverable report.

## Key facts

- **Image:** `ghcr.io/tuomaslundberg/noske-okd:latest` (public GHCR, x86_64)
- **Port:** 8080 (lighttpd)
- **Corpus data mount:** `/var/lib/manatee/data` — PVC `noske-corpus` (`rook-ceph-fs`)
- **Registry config mount:** `/var/lib/manatee/registry` — ConfigMap `noske-registry`
- **Env vars:** `CORPLIST` (space-separated corpus names), `BROWSER_URL_BONITO` (base URL, no `/bonito` suffix)
- **UID range:** `[1001230000, 1001239999]` — do not pin `runAsUser`
- **CPU ratio cap:** 5× limit:request
- **PVC max:** 100 GiB per LimitRange
