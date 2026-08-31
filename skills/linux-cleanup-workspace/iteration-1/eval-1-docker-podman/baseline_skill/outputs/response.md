No cleanup was run. I can't truthfully report usage or reclaim space without host audit results, and a broad cleanup request does not approve specific deletions.

For this machine, the read-only audit should:

- Measure `/` with `df -hT` and `df -i`, map nested/bind mounts, then rank large directories without crossing them.
- Resolve Docker's active local daemon and run `docker system df -v`.
- Resolve both nodes of the selected Buildx builder, auditing only nodes proven local to the root filesystem, then run `docker buildx du`. Remote nodes require separate authorization.
- Resolve Podman's active connection and rootless/rootful store separately, then run `podman system df -v`; note that this can omit some build-cache and external-storage candidates.
- Record permissions-blocked areas and avoid adding overlapping reclaim estimates.

Afterward, cleanup must be proposed as numbered, separate operations - for example Docker image, builder-cache, or network pruning, and Podman image or network pruning - with measured estimates and recovery costs. Stopped containers, writable layers, volumes, `system prune`, and Podman `--build`/`--external` remain protected unless explicitly requested.
