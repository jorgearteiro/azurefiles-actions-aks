# Copilot instructions for azurefiles-actions-aks

## What this repository is

This is a **tutorial / reference solution**, not a product. It demonstrates how to use an
Azure Files premium SMB share to cache NuGet packages and provide dynamic ephemeral volumes
for GitHub Actions self-hosted runners running on AKS via **GitHub ARC** (Actions Runner
Controller) and **Runner Scale Sets**. `README.md` is the canonical, step-by-step guide and
the primary artifact of this repo — treat it as source of truth and keep it in sync with any
change to `install/` or `.github/workflows/`.

## The two independent halves

1. **The .NET app (a stand-in workload, not the point).** `Program.cs`, `Components/`,
   `wwwroot/`, `*.csproj` form a stock **.NET 8 Blazor Server** template (default `Home`,
   `Counter`, `Weather` pages). `azurefiles-actions-aks.csproj` deliberately lists ~45
   unrelated `PackageReference`s (Azure SDKs, EF-adjacent libs, message buses, etc.) — this is
   intentional bloat to make `dotnet restore` download a large NuGet payload so the Azure Files
   caching layer can be observed. **Do not "clean up" or trim these packages**; do not treat the
   app as a real application to extend.

2. **The infrastructure (the actual subject).** `install/*.yaml` (Kubernetes/Helm manifests)
   and `.github/workflows/*.yml` (demo workflows that run on the self-hosted runners). Almost
   all meaningful work happens here.

## Build / run / test

The app builds with the standard .NET SDK toolchain (project targets `net8.0`):

```bash
dotnet restore
dotnet build --configuration Release --no-restore
dotnet publish --configuration Release --no-restore --output ./publish
```

There is **no unit test project** despite `xunit`/`Moq` appearing in `.csproj` (they are part of
the intentional package bloat). "Testing" in this repo means manually dispatching the workflows
under `.github/workflows/` against a live ARC runner set on AKS — there is no local test suite to run.

Deploying/validating the infrastructure requires a live AKS cluster and the CLI trio **`az`,
`kubectl`, `helm`**; changes to `install/*.yaml` cannot be verified locally without one.

## Key conventions and cross-file wiring

- **Fixed identifiers must stay consistent across files.** The share name `metadatacaching`,
  the PV/PVC claim name `azurefile`, the secret name `azure-storage-secret`, the namespaces
  `arc-runners` / `arc-systems`, the `hook-extension` ConfigMap, and the NuGet mount path
  `/home/runner/.nuget/` are referenced from multiple manifests and from `README.md`. Renaming
  one requires updating every referrer (README calls out several of these couplings explicitly).
- **fsGroup / uid / gid values are load-bearing.** `fsGroup: 123` (the GitHub runner image group)
  and the SMB `mountOptions` (`uid=1001`, `gid=123`, `dir_mode=0777`, `nobrl`, `nosharesock`,
  `cache=strict`, `actimeo=30`) appear across the PV, storage classes, values file, and pod spec.
  Keep them aligned — they are what make the shared SMB mount usable by the runner containers.
- **Two ways Azure Files is consumed**, and both must keep working: (a) a static, retained
  **PV/PVC** (`arc-runners-set-pv-pvc.yaml`) mounted read-write-many at `~/.nuget/` for NuGet
  caching; (b) **dynamic ephemeral** per-job volumes via the `github-azurefile` (Standard_LRS)
  and `github-azurefile-premium` (Premium_LRS) StorageClasses in
  `arc-runners-storage-class-files.yaml`.
- **Runner pod customization lives in `arc-runners-set-values.yaml`** (the Helm values passed via
  `--values`), using `containerMode.type: kubernetes` plus a custom `template.spec` that mounts
  the NuGet share and the `container-podspec-volume`. Non-templated Helm args (`githubConfigUrl`,
  `githubConfigSecret`, `minRunners`, `maxRunners`, `runnerGroup`, `--version`) are passed via
  `--set` on the `helm install`/`helm upgrade --install` command, not in the values file.
- **Controller and runner scale-set Helm chart versions must match** (README pins `0.9.3` in
  examples). If you bump one, bump the other.
- **Workflows are manual-only.** Every file in `.github/workflows/` uses `workflow_dispatch`
  with a required `arc_runner_set_name` input (default `arc-runner-set`) wired into `runs-on:`.
  Keep new demo workflows to this pattern so they target the self-hosted runners on demand.

## Guardrails

- `README.md` contains an **example RSA private key and GitHub App IDs** as illustration. Never
  reproduce, "complete", or treat these as real secrets, and do not add real credentials anywhere.
- Comments and prose in this repo contain non-blocking typos; do not do sweeping unrelated
  copy-editing when making a focused change.
