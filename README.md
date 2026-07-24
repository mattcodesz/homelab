# homelab

GitOps repository for the personal Kubernetes homelab.

## Structure

- `clusters/staging` bootstraps Flux and points it at the staging environment.
- `infrastructure` installs controllers and cluster services: MetalLB, TrueNAS CSI, Traefik, cloudflared, and CloudNativePG.
- `infrastructure-configs` contains controller-specific configuration such as the MetalLB address pool.
- `apps` contains application namespaces, storage, deployments, services, and routes.

The current cluster keeps PostgreSQL data on Kubernetes `local-path` storage. TrueNAS is used for Jellyfin media and application PVCs, but not for the database. Linkding's PostgreSQL instances are required to use different Kubernetes nodes when two schedulable nodes are available.

## Changes currently applied

- Jellyfin and cloudflared images are version-pinned.
- All Helm releases have explicit chart versions.
- Jellyfin, Linkding, and cloudflared have startup/readiness/liveness checks.
- Application containers use restricted capabilities and the RuntimeDefault seccomp profile.
- cloudflared has two replicas, topology spreading, and a disruption budget.
- The Linkding database uses required CloudNativePG pod anti-affinity across `kubernetes.io/hostname`.
- The infrastructure Flux Kustomization health-checks MetalLB, democratic-csi, Traefik, and CloudNativePG before the apps layer proceeds.
- The staging root Kustomization explicitly includes Flux, infrastructure, infrastructure configuration, and apps; this is the path referenced by the Flux GitRepository sync.

Required database anti-affinity means the second PostgreSQL instance will remain Pending if the cluster has fewer than two suitable nodes. This is intentional: co-locating both instances would not provide node-failure protection.

## Roadmap

### Near term

1. Verify the pinned versions in staging and keep the previous versions available for rollback.
2. Add pull-request validation for Kustomize rendering, Kubernetes schemas, YAML formatting, and security policies.
3. Add application-specific NetworkPolicies after observing the actual traffic requirements.
4. Confirm the Cloudflare tunnel routes only the Linkding hostname. Jellyfin is intentionally LAN-only and remains HTTP on the LAN.

CI (continuous integration) means GitHub automatically runs checks for each pull request before it is merged. The planned workflow should check out the repository, render every Kustomize entry point, lint the YAML, validate Kubernetes schemas, and run a configuration security scanner. Branch protection should require those checks to pass. Renovate can later open pull requests for new image and Helm chart versions.

### When rack hardware is available

1. Keep PostgreSQL on Kubernetes, but give each instance its own durable cluster-local volume and keep the required node anti-affinity.
2. Configure CloudNativePG base backups and WAL archiving to Azure Blob Storage.
3. Store the Azure credentials in a SOPS-encrypted Secret, or use Azure Workload Identity once the cluster identity setup is ready.
4. Define retention and lifecycle rules, then test restoring Linkding into a temporary namespace.
5. Back up Jellyfin configuration and other irreplaceable application data separately from large media files.

### Optional LAN HTTPS

LAN HTTPS is not required for the current design. If it becomes desirable, add cert-manager with a narrowly scoped Cloudflare DNS-01 token, issue a certificate for the internal hostnames, configure Traefik's `websecure` entrypoint, and redirect `web` to HTTPS. Do not add this until the DNS and certificate ownership model is documented.

## Recovery prerequisites

The SOPS age private key, Flux deploy key, Cloudflare tunnel credentials, and any local operational documentation are required for a full rebuild. The encrypted secrets in this repository cannot be recovered from Git without the corresponding private key.
