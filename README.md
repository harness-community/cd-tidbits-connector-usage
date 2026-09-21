# CI | Tidbits | Connector Usage

> **Bite-sized how-to** | ~15 min setup

---

## What is a connector?

A connector is a reusable login. Pipelines never hold passwords — they hold a `connectorRef`. Harness authenticates at runtime and masks secrets in logs.

Auth methods, scope (`account.` / `org.` / project), and how resolution works: [Connectors](https://developer.harness.io/harness-platform/3.0/in-harness-3.0/connectors).

This tidbit uses three of them to clone [podinfo](https://github.com/harness-community/podinfo), push an image to Docker Hub, and deploy it to EKS:

| Connector | Auth in this tidbit |
|---|---|
| **GitHub** | Username + PAT |
| **Docker Hub** | Username + access token |
| **AWS** | **Use OIDC** |

AWS also supports Access Key, Assume IAM Role on Delegate, IRSA, and Custom (Credential Broker). Those are covered in the same docs. This walkthrough uses OIDC so Harness never stores long-lived AWS keys — each run gets a short-lived token tied to your account/org/project.

---

## Prerequisites

Before you start, make sure you have:

- A Harness account with a **Project** (note its org + project identifiers, and your Harness account id).
- Harness Cloud build credits.
- A GitHub PAT, a Docker Hub access token, and an EKS cluster you can deploy to.
- Namespace `podinfo` in the cluster (`kubectl create namespace podinfo` if needed).

---

## Step 1 — Secrets and connectors

Create two Text secrets: `github_pat` and `dockerhub_token`.

Then create three connectors (**Project Settings → Connectors**). Test each one. YAML samples: [`connectors/`](./connectors/).

1. **GitHub** (id `github`) — URL `https://github.com`, Account, Username and Token → `github_pat`, API access on.
2. **Docker Registry** (id `dockerhub`) — Docker Hub, `https://index.docker.io/v2/`, Username and Password → `dockerhub_token`.
3. **AWS** (id `eks_oidc`) — **Use OIDC**, your IAM role ARN, cluster region. Connect through the Harness Platform (or a Delegate if the API is private).

---

## Step 2 — Environment, infrastructure, and service

Paste these into your project and edit every `# REPLACE:` line:

| File | Creates |
|---|---|
| [`.harness/environment.yaml`](./.harness/environment.yaml) | Environment `tidbit` |
| [`.harness/infrastructure.yaml`](./.harness/infrastructure.yaml) | EKS target on `eks_oidc` |
| [`.harness/service.yaml`](./.harness/service.yaml) | Service `podinfo` (Docker Hub image + `manifests/`) |

If you forked this repo, set `repoName` on the service to your fork.

---

## Step 3 — Import the pipeline

1. **Pipelines → Create a Pipeline** → YAML editor.
2. Paste [`.harness/pipeline.yaml`](./.harness/pipeline.yaml).
3. Set org, project, and `<YOUR_DOCKERHUB_USER>/podinfo` (same path as the service).
4. Save.

---

## Step 4 — Run the pipeline (expect a GREEN build)

Click **Run**, keep branch `main`, click **Run Pipeline**.

Build clones `harness-community/podinfo` and pushes to Docker Hub. Deploy rolls that image out to EKS. Expected: green.

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
    codebase: { connectorRef: github, repoName: harness-community/podinfo }
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
                spec: { connectorRef: dockerhub, repo: <YOUR_DOCKERHUB_USER>/podinfo }
  - stage:
      name: Deploy
      type: Deployment
      spec:
        service: { serviceRef: podinfo }
        environment:
          environmentRef: tidbit
          infrastructureDefinitions: [{ identifier: eks_direct }]
```

---

## Common Issues & Tips

**Clone or manifests fail.** Test the GitHub connector; check `github_pat` **Id** and `repoName`.

**AWS test fails.** OIDC provider URL and role trust must match the connector scope — see the [connectors](https://developer.harness.io/harness-platform/3.0/in-harness-3.0/connectors) page.

**Deploy cannot reach the cluster.** Private API? Use a Delegate. Forbidden? Map the IAM role on the cluster for namespace `podinfo`.

**`ImagePullBackOff`.** Image path must match the push; re-test `dockerhub`.

---

## What's next?

- Swap the GitHub PAT for a GitHub App.
- Add an approval between Build and Deploy.
- Store `.harness/` as Remote so reviews see connector *structure*, never secret values.

---

## Resources

- [Connectors](https://developer.harness.io/harness-platform/3.0/in-harness-3.0/connectors)
- [podinfo](https://github.com/harness-community/podinfo)
