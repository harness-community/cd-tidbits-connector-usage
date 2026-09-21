# CI | Tidbits | Connector Usage

> **Bite-sized how-to** | ~15 min setup

---

## What is a connector?

A connector is a reusable login. Pipelines never hold passwords — they hold a `connectorRef`. Harness authenticates at runtime and masks secrets in logs.

Auth methods, scope (`account.` / `org.` / project), and how resolution works: [Connectors](https://developer.harness.io/harness-platform/3.0/in-harness-3.0/connectors).

This tidbit uses three of them to clone [podinfo](https://github.com/harness-community/podinfo), push an image to Docker Hub, and deploy it with KubernetesDirect:

| Connector | Auth in this tidbit |
|---|---|
| **GitHub** (`githubconnector`) | Username + PAT |
| **Docker Hub** (`dockerhubconnector`) | Username + access token |
| **Kubernetes** (`k8sconnector`) | Delegate (`InheritFromDelegate`) |

---

## Prerequisites

Before you start, make sure you have:

- A Harness account with a **Project** (note its org + project identifiers).
- Harness Cloud build credits.
- A GitHub PAT, a Docker Hub access token, and a Kubernetes cluster you can deploy to.
- A Harness **Delegate installed in that cluster**. The Kubernetes connector inherits credentials from the Delegate.
- Namespace `podinfo` in the cluster (`kubectl create namespace podinfo` if needed).

---

## Step 1 — Secrets and connectors

Create two Text secrets: `github-pat` and `dockerhub-pat`.

Then create three connectors (**Project Settings → Connectors**). Test each one. YAML samples: [`connectors/`](./connectors/).

1. **GitHub** (`github-connector` / `githubconnector`) — **Repo**, URL `https://github.com/harness-community/podinfo`, Username and Token → `github-pat`.
2. **Docker Registry** (`dockerhub-connector` / `dockerhubconnector`) — Docker Hub, `https://index.docker.io/v2/`, Username and Password → `dockerhub-pat`.
3. **Kubernetes Cluster** (`k8s-connector` / `k8sconnector`) — **Use the credentials of a Delegate** (`InheritFromDelegate`). Set `delegateSelectors` to tags on that Delegate.

---

## Step 2 — Environment, infrastructure, and service

Paste these into your project and edit every `# REPLACE:` line:

| File | Creates |
|---|---|
| [`.harness/environment.yaml`](./.harness/environment.yaml) | Environment `podinfoenv` |
| [`.harness/infrastructure.yaml`](./.harness/infrastructure.yaml) | KubernetesDirect target on `k8sconnector` |
| [`.harness/service.yaml`](./.harness/service.yaml) | Service `podinfo` (Docker Hub image + Kustomize from `harness-community/podinfo`) |

Manifests come from `kustomize/` on branch `master` of [harness-community/podinfo](https://github.com/harness-community/podinfo). You do not need a copy of those manifests in this repo.

---

## Step 3 — Import the pipeline

1. **Pipelines → Create a Pipeline** → YAML editor.
2. Paste [`.harness/pipeline.yaml`](./.harness/pipeline.yaml).
3. Set org, project, and `<YOUR_DOCKERHUB_USER>/podinfo` (same path as the service).
4. Save.

---

## Step 4 — Run the pipeline (expect a GREEN build)

Click **Run**, keep branch `master`, click **Run Pipeline**.

Build clones `harness-community/podinfo` and pushes to Docker Hub. Deploy rolls that image out with Kustomize. Expected: green.

```sh
kubectl -n podinfo get deploy,po,svc
kubectl -n podinfo port-forward svc/podinfo 9898:9898
# curl localhost:9898
```

**Green is the correct outcome.** All three connectors authenticated.

---

## Pipeline YAML reference

[`.harness/pipeline.yaml`](./.harness/pipeline.yaml) — key shape:

```yaml
properties:
  ci:
    codebase: { connectorRef: githubconnector, repoName: harness-community/podinfo }
stages:
  - stage:
      name: Build
      type: CI
      spec:
        cloneCodebase: true
        runtime: { type: Cloud, spec: {} }
        execution:
          steps:
            - step:
                type: BuildAndPushDockerRegistry
                spec: { connectorRef: dockerhubconnector, repo: <YOUR_DOCKERHUB_USER>/podinfo }
  - stage:
      name: Deploy
      type: Deployment
      spec:
        service: { serviceRef: podinfo }
        environment:
          environmentRef: podinfoenv
          infrastructureDefinitions: [{ identifier: podinfoinfra }]
```

---

## Common Issues & Tips

**Clone or Kustomize fetch fails.** Test the GitHub connector; check secret id `github-pat`. The connector is Repo-scoped to `harness-community/podinfo`. Manifests are on `master` under `kustomize/`.

**Kubernetes test fails.** Confirm the Delegate is running in the cluster and the connector `delegateSelectors` match.

**Deploy cannot apply resources.** The Delegate service account needs permission in namespace `podinfo`.

**`ImagePullBackOff`.** Image path must match the push; re-test `dockerhubconnector`.

---

## What's next?

- Swap the GitHub PAT for a GitHub App.
- Add an approval between Build and Deploy.
- Store `.harness/` as Remote so reviews see connector *structure*, never secret values.

---

## Resources

- [Connectors](https://developer.harness.io/harness-platform/3.0/in-harness-3.0/connectors)
- [podinfo](https://github.com/harness-community/podinfo)
