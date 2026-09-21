# CI | Tidbits | Connector Usage

> **Bite-sized how-to** | ~10 min setup

---

## What is a connector?

A Harness **connector** is a reusable, scoped credential. Pipelines never embed passwords — they hold a `connectorRef`. At runtime Harness authenticates with the stored secret and (for log lines that match the secret) masks the value.

This tidbit covers the two connector families you hit first in CI:

| Family | What it authenticates | Examples in this repo |
|---|---|---|
| **Artifact** | Pull / push container images | Docker Hub, Amazon ECR, Google Container Registry (GCR) |
| **Infrastructure** | Where the build runs | Kubernetes cluster |

They are **not interchangeable**. A Kubernetes connector can schedule pods; it cannot log in to ECR. A Docker Hub connector can pull `alpine`; it cannot talk to the Kubernetes API. Stages that run on a cluster *and* pull private images need **both**.

The sample pipeline is deliberately tiny — one CI stage, one Run step, public `alpine:3.20` — so you can watch `connectorRef` do the work.

---

## Prerequisites

Before you start, make sure you have:

- A Harness account with a **Project** (note its org + project identifiers).
- Harness Cloud build credits (default on Harness-hosted runners). **No delegate and no cluster required** for the default path.
- Permission to create Connectors (and Secrets, if you add credentials later).

> **Note:** No repo fork is required for the default path. Paste the pipeline YAML into the studio. Fork only if you want the connector samples next to the pipeline in Git.

---

## Step 1 — Create a Docker Hub connector

In your Harness project:

1. Go to **Project Settings → Connectors → + New Connector → Docker Registry**.
2. **Overview:** Name `Docker Hub` (id becomes `dockerhub`).
3. **Details:**
   - Provider type: **Docker Hub**
   - Docker Registry URL: `https://index.docker.io/v2/`
   - Authentication: **Anonymous**
4. **Connectivity:** Connect through Harness Platform (Harness Cloud does not need a delegate).
5. **Test Connection** → it should pass → **Save**.

Anonymous is enough to pull public images. You will add a username + token in Step 4 if you hit Hub rate limits or need a private repo.

YAML equivalent: [`connectors/dockerhub.yaml`](./connectors/dockerhub.yaml).

---

## Step 2 — Import the pipeline

1. Go to **Pipelines → Create a Pipeline**.
2. Name it `Connector Usage`, choose **Inline**, open the **YAML** editor.
3. Paste the contents of [`.harness/connector_usage.yaml`](./.harness/connector_usage.yaml).
4. Edit the `# REPLACE:` lines: `projectIdentifier`, `orgIdentifier`, and `connectorRef` (use `dockerhub`, or `account.dockerhub` / `org.dockerhub` if you created it at a higher scope).
5. Save.

---

## Step 3 — Run the pipeline (expect a GREEN build)

Click **Run → Run Pipeline**.

Harness Cloud starts a Linux/Amd64 runner, the Run step authenticates to Docker Hub via your connector, pulls `alpine:3.20`, and prints `/etc/os-release`. Expected: green in under a minute.

```
Pulled this image through the artifact connector on connectorRef.
Image: alpine:3.20
---
NAME="Alpine Linux"
...
---
SUCCESS: artifact connector authenticated the pull.
```

**Green is the correct outcome.** It proves the artifact connector is wired into the step.

---

## Step 4 — Authenticate (optional, but what you do in real pipelines)

Anonymous Hub pulls are rate-limited. Private images always need credentials.

1. Create a Docker Hub **access token** (Docker Hub → Account Settings → Personal access tokens). Store it as a Harness Text Secret, id `dockerhub_token`. Do **not** put the token in Git. See the [Secrets Management tidbit](https://github.com/harness-community/ci-tidbits-secrets-management) if you have not created a secret yet.
2. Edit the connector: Authentication → **Username and Password**. Username = your Hub user. Password = secret `dockerhub_token`.
3. Test Connection again, then re-run the pipeline. Same green, now with authenticated pulls.

The YAML shape:

```yaml
auth:
  type: UsernamePassword
  spec:
    username: <YOUR_DOCKERHUB_USER>
    passwordRef: dockerhub_token
```

---

## Step 5 — Same pipeline, different registries (ECR / GCR)

`connectorRef` is the only thing that changes. The Run step still pulls an image; Harness uses whichever connector you point at.

### Amazon ECR

Do **not** paste a 12-hour `docker login` token into a Docker Registry connector — it expires twice a day. Create an **AWS** connector ([`connectors/ecr.yaml`](./connectors/ecr.yaml)) with IAM keys (or IRSA / OIDC on a delegate). The IAM principal needs `ecr:GetAuthorizationToken` plus pull/push on the repos you use.

Then either:

- Keep this tidbit's Run step: create a Docker Registry connector whose URL is `https://<account>.dkr.ecr.<region>.amazonaws.com` **or**
- Graduate to a `BuildAndPushECR` step that takes the **AWS** connector directly — that is the [Docker Build & Push tidbit](https://github.com/harness-community/ci-tidbits-docker-build-push) with the ECR step type.

Swap in the pipeline:

```yaml
connectorRef: aws_ecr          # or your ECR Docker Registry id
image: <ACCOUNT>.dkr.ecr.<REGION>.amazonaws.com/<REPO>:<TAG>
```

### Google Container Registry (GCR)

GCR is legacy — new images belong in Artifact Registry. The same **GCP** connector covers both ([`connectors/gcr.yaml`](./connectors/gcr.yaml)). Store the service-account JSON as a Harness secret (`gcp_sa_key`); never commit the key file.

```yaml
connectorRef: gcp_gcr
image: gcr.io/<PROJECT>/<IMAGE>:<TAG>
# or Artifact Registry:
# image: <REGION>-docker.pkg.dev/<PROJECT>/<REPO>/<IMAGE>:<TAG>
```

The Docker Registry fallback uses URL `https://gcr.io` (or `https://us.gcr.io` / `https://eu.gcr.io` / `https://asia.gcr.io`) with username `_json_key` and `passwordRef` pointing at the same SA JSON secret.

---

## Step 6 — Infrastructure connector (Kubernetes)

Harness Cloud (the default in this pipeline) needs **no** cluster connector. To run the *same* steps on your cluster:

1. Install a Harness Delegate in the cluster (or on a host that can reach the API server).
2. Create a **Kubernetes Cluster** connector ([`connectors/k8s.yaml`](./connectors/k8s.yaml)):
   - **Inherit from delegate** if the delegate already runs in that cluster (simplest).
   - **Master URL + service account** if the delegate is elsewhere (required for Docker delegates — they cannot use inherit-from-delegate against `localhost:8080`).
3. In the stage, delete `platform` + `runtime` and uncomment the `infrastructure` block in [`.harness/connector_usage.yaml`](./.harness/connector_usage.yaml):

```yaml
infrastructure:
  type: KubernetesDirect
  spec:
    connectorRef: k8s_cluster
    namespace: harness-delegate-ng
    automountServiceAccountToken: true
    os: Linux
```

The Run step's `connectorRef: dockerhub` stays. Infrastructure connector = where the pod runs. Artifact connector = how the pod pulls `alpine:3.20`.

> **EKS / GKE:** you can use this platform-agnostic K8s connector *or* an AWS / GCP connector for the cluster. CI **build infrastructure** still wants a Kubernetes Cluster connector on the stage. Cloud-provider connectors are for artifacts and cloud APIs.

---

## Connector YAML reference

```
.
├── .harness/
│   └── connector_usage.yaml   ← executable pipeline (Cloud + Docker Hub)
├── connectors/
│   ├── dockerhub.yaml         ← Docker Registry / Docker Hub
│   ├── ecr.yaml               ← AWS connector for ECR
│   ├── gcr.yaml               ← GCP connector for GCR / GAR
│   └── k8s.yaml               ← Kubernetes cluster connector
└── README.md
```

Every `passwordRef` / `secretKeyRef` / `accessKeyRef` is a **secret id**, not a value. Secrets are created in the UI (or the secrets tidbit); connectors only reference them.

### Scope prefixes

Connectors follow the same scope rules as secrets:

| Connector lives at | `connectorRef` |
|---|---|
| Project | `dockerhub` |
| Org | `org.dockerhub` |
| Account | `account.dockerhub` |

Account-level connectors are visible to every org and project; project-level ones are not.

---

## Common Issues & Tips

**`docker pull` rate-limit / `toomanyrequests`.** Anonymous Hub is throttled. Add UsernamePassword to the Docker Hub connector (Step 4) or pull from GAR/ECR/the [Harness image registry](https://developer.harness.io/docs/platform/connectors/artifact-repositories/connect-to-harness-container-image-registry-using-docker-connector/).

**`CONNECTOR_NOT_FOUND` / Test Connection fails.** The id in `connectorRef` does not match, or the scope prefix is wrong. Copy the **Id** from the connector list, not the display name. Add `org.` / `account.` when the connector is not in this project.

**ECR: `unauthorized` after a few hours.** You stored a `get-login-password` token on a Docker Registry connector. Switch to an AWS connector ([`connectors/ecr.yaml`](./connectors/ecr.yaml)).

**K8s inherit-from-delegate fails with a Docker delegate.** Docker delegates are not in the cluster's network namespace, so `localhost:8080` is wrong. Use **Specify Master URL and Credentials** ([`connectors/k8s.yaml`](./connectors/k8s.yaml) variant B).

**Stage on K8s cannot pull a private image.** The *infrastructure* connector scheduled the pod; it did not authenticate to the registry. Set `connectorRef` on the Run / BuildAndPush step to an artifact connector.

**Visual / YAML out of sync after a connector edit.** Save the connector, then re-open the pipeline. `connectorRef` is a string; Harness does not rewrite it when you rename a connector — update the pipeline yourself.

---

## What's next?

- **Push an image, not just pull.** Swap the Run step for `BuildAndPushDockerRegistry` (Hub), `BuildAndPushECR`, or `BuildAndPushGAR` and follow [ci-tidbits-docker-build-push](https://github.com/harness-community/ci-tidbits-docker-build-push).
- **Put secrets in a Secret Manager first.** [ci-tidbits-secrets-management](https://github.com/harness-community/ci-tidbits-secrets-management) is the companion lesson for `passwordRef`.
- **Store connectors next to pipelines.** Move `connectors/*.yaml` into `.harness/` in a real repo, set the connector to **Remote** Git storage, and review credential *structure* in PRs (never the secret values).

---

## Resources

- [Add a Kubernetes cluster connector](https://developer.harness.io/docs/platform/connectors/cloud-providers/add-a-kubernetes-cluster-connector/)
- [Kubernetes cluster connector settings](https://developer.harness.io/docs/platform/connectors/cloud-providers/ref-cloud-providers/kubernetes-cluster-connector-settings-reference/)
- [Connect to the Harness container image registry](https://developer.harness.io/docs/platform/connectors/artifact-repositories/connect-to-harness-container-image-registry-using-docker-connector/)
- [CD artifact sources (Docker Hub, ECR, GCR)](https://developer.harness.io/docs/continuous-delivery/x-platform-cd-features/services/artifact-sources/)
- [Run step settings (`connectorRef`, `image`)](https://developer.harness.io/docs/continuous-integration/use-ci/run-step-settings/)
- [Harness Cloud build infrastructure](https://developer.harness.io/docs/continuous-integration/use-ci/set-up-build-infrastructure/use-harness-cloud-build-infrastructure/)
