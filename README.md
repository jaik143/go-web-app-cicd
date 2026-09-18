# Go Web App — CI/CD to Kubernetes

A Go web service packaged into a distroless container image and delivered to Kubernetes by a fully automated GitHub Actions pipeline, with Helm as the deployment interface.

![Go](https://img.shields.io/badge/Go-00ADD8?style=flat-square&logo=go&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?style=flat-square&logo=helm&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-2088FF?style=flat-square&logo=githubactions&logoColor=white)
![License](https://img.shields.io/badge/license-Apache%202.0-blue?style=flat-square)

---

## What this demonstrates

The application itself is deliberately small. The point of the repository is everything around it: how a commit becomes a running, versioned workload in a cluster without anyone touching `kubectl`.

## Pipeline

```
 push to main
      |
      +-- build ---------> go build . go test ./...
      |
      +-- code-quality --> golangci-lint
      |
      +-- push ----------> docker buildx . push image:<run-id>
      |
      +-- update-tag ----> rewrite helm values.yaml image tag . commit back to main
                                      |
                                      v
                            Argo CD / helm upgrade
                                      |
                                      v
                           Kubernetes + Nginx Ingress
```

| Job | What it does | Why it is separate |
|---|---|---|
| `build` | Compiles the binary and runs `go test ./...` | Nothing is packaged until tests pass |
| `code-quality` | Runs `golangci-lint` | Lint failures stay visible independently of test failures |
| `push` | Buildx build, pushes image tagged with the workflow run ID | Every image traces back to exactly one workflow run |
| `update-newtag-in-helm-chart` | Writes the new tag into `values.yaml` and commits it | Desired cluster state lives in Git, not in a CI job's memory |

**Images are never tagged `latest`.** Each build is tagged with its workflow run ID, so a rollback is a Helm value change to a known-good tag rather than a rebuild.

## Container image

```dockerfile
FROM golang:1.22.5 AS base      # compile stage
...
FROM gcr.io/distroless/base     # runtime stage
COPY --from=base /app/main .
```

The runtime image is `gcr.io/distroless/base` — no shell, no package manager, no OS utilities. A compromised process has almost nothing to pivot to, and the image is a fraction of the size of a `golang`-based runtime.

## Repository layout

```
.
├── .github/workflows/          # CI/CD pipeline definition
├── helm/go-web-app-chart/      # Deployment, Service and Ingress as a chart
├── k8s/manifests/              # plain manifests (pre-Helm reference)
├── static/                     # served assets
├── dockerfile                  # multi-stage build -> distroless runtime
├── main.go / main_test.go      # application and unit tests
└── go.mod
```

## Running locally

```bash
go run main.go
# http://localhost:8080/courses
```

## Building and running the container

```bash
docker build -t go-web-app .
docker run -p 8080:8080 go-web-app
```

## Deploying to Kubernetes

With plain manifests:

```bash
kubectl apply -f k8s/manifests/
```

With Helm:

```bash
helm install go-web-app ./helm/go-web-app-chart
helm upgrade go-web-app ./helm/go-web-app-chart --set image.tag=<run-id>
```

The service is exposed through an Nginx Ingress Controller. Point the chart's configured host at the ingress controller's external address to reach it.

## Required repository secrets

| Secret | Purpose |
|---|---|
| `DOCKERHUB_USERNAME` | Registry namespace for the pushed image |
| `DOCKERHUB_TOKEN` | Registry push credential |
| `TOKEN` | Write access for the Helm tag-bump commit |

---

**Author** — Kadali Jayanth Kumar · [LinkedIn](https://linkedin.com/in/jayanth-kadali-419798182)
