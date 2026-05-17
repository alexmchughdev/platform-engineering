# 0004 - Supply chain security in CI: Trivy, Cosign, Syft, govulncheck, TruffleHog

Date: 2026-05-17
Status: Accepted

## Context

Every app on this platform follows the same delivery shape: GitHub Actions builds a container image, pushes it to GHCR, then commits a `kustomize edit set image` to this repo. ArgoCD reconciles the bump into the cluster within minutes. Until this ADR, "what is actually running in the cluster" depended on three implicit trusts:

1. That the source repo has not been tampered with between commit and build.
2. That the base image and Go module graph contain no known critical vulnerabilities.
3. That nothing in CI itself (an action, a step, a registry interaction) was substituted at build time.

Two things made this acceptable for the early phase of the platform: every image was built by a workflow owned by the same person reading the diffs, and the cluster is single-tenant. Neither will hold as the platform grows.

The March 2026 supply chain attack against `tj-actions/changed-files`, and the wave of follow-on compromises that hit `aquasecurity/trivy-action` shortly after, is live evidence. A pinned GitHub Action with a clean reputation had its tags rewritten, and every downstream workflow that referenced `@main` or a mutable major-version tag pulled malicious code on the next run. The cost of that scenario for this platform is not theoretical: a hostile build could push a working image, bump the overlay, and the cluster would reconcile it within three minutes.

Phase 6 of the platform roadmap adds Kyverno `verifyImages` to refuse any pulled image without a valid Cosign signature (`decisions/0002` covers the GitOps loop the policy sits in front of). That control is only useful if images arrive at the cluster already signed. The signing has to land first.

## Decision

Each app's CI workflow runs the following stack before, during, and after the image build. Two reference implementations exist: OT-edge (`.github/workflows/ci.yaml` in `alexmchughdev/OT-edge-asset-tag-generator`, Go) and `alexmchugh-dev` (`.github/workflows/ci.yaml` in `alexmchughdev/alexmchugh-dev`, Node/Next.js).

**Source-level controls (pre-build, language-specific)**

- Type and lint checks on every push and PR. OT-edge runs `go vet` and `golangci-lint`; `alexmchugh-dev` runs `tsc --noEmit` and `next lint`.
- Unit tests with the race detector where applicable (`go test -race` on OT-edge; `alexmchugh-dev` has no test suite yet, tracked as a follow-up).
- Dependency vulnerability scan with the best signal each ecosystem offers: `govulncheck` on OT-edge for call-graph reachability (a CVE in an imported package the app never reaches does not block the build); `npm audit --audit-level=critical --omit=dev` plus OSV-Scanner SARIF on `alexmchugh-dev` because Node has no call-graph equivalent.
- TruffleHog with `--only-verified` so the gate fails only on credentials TruffleHog could verify against a live service, not on regex matches alone.

**Filesystem scan (pre-build)**

- Trivy `fs` scan of the checked-out source. Two passes: one writes SARIF for the GitHub Security tab (MEDIUM and up, `ignore-unfixed`), one blocks the build at CRITICAL only.

**Build**

- `docker/build-push-action` with `provenance: true` and `sbom: true` so the OCI manifest carries a SLSA provenance attestation and an inline SBOM out of the box.

**Post-build signing and attestation**

- Cosign keyless signing via OIDC (Sigstore Fulcio for the cert, Rekor for the transparency log). No long-lived key material in CI; the signature is bound to the workflow identity that issued it.
- Syft generates two SBOM formats (SPDX JSON and CycloneDX JSON). Cosign attests the SPDX SBOM against the image digest so the SBOM is verifiable, not just downloadable.

**Image scan (post-build, pre-deploy)**

- Trivy `image` scan of the pushed image. Same two-pass shape as the filesystem scan: SARIF for the Security tab, blocking pass at CRITICAL only.

**Manifest bump (post-scan)**

- The `update-manifest` job is gated on every prior job. A failed scan, signature, or attestation never produces a manifest commit, so the cluster never reconciles an unsigned or known-CRITICAL image.

The CRITICAL-only blocking threshold is deliberate. HIGH and MEDIUM findings still appear in the Security tab and still need triage; they do not stop a deployment.

## Consequences

**Positive**

- Every image landing in the cluster carries a Cosign signature anchored in a public transparency log. Kyverno `verifyImages` (Phase 6) can enforce the signature at admission time. Without the signing in CI today, that policy would have nothing to verify.
- The SBOM is generated, attested, and uploaded as a workflow artifact. Downstream consumers (auditors, vulnerability re-scans, registries that read attestations) can pull it.
- The supply chain controls are layered. A bypass of any single step (a missed CVE in `govulncheck`, a Trivy false negative) still has the other steps. The pinned-action compromise pattern is partially mitigated because the layers are independently capable of catching the resulting image change.
- Findings flow into the GitHub Security tab via SARIF, so triage uses the standard GitHub UI rather than raw workflow logs.

**Negative**

- End-to-end CI time roughly triples versus a plain build-and-push. Most of the cost is the image scan and the SBOM generation. The trade is acceptable for a workflow that runs on every push to `main`, not on hot interactive feedback.
- The Cosign keyless flow writes an entry to the public Rekor transparency log for every signed image. Repository name, workflow ref, and image digest become public. Acceptable here because every repo in this platform is already public; would need re-evaluation for any private app added later.
- Trivy produces a steady background of HIGH and MEDIUM findings, mostly base-image CVEs without fixed versions. The CRITICAL-only blocking threshold prevents these from gating delivery, at the cost of letting a known HIGH ride into production until the base image is rebuilt.
- TruffleHog `--only-verified` will miss any credential it cannot verify against a live endpoint. Fewer false positives at the cost of some recall. Acceptable for repos that already have pre-commit hooks and no historical secret leaks.

## Roadmap link

Rolling this stack out to `fixmycampus` and `foghorn` is on the README roadmap. `lookout` will reach equivalent assurance via goreleaser plus SLSA L3 because it ships as a published binary rather than a container image. Phase 6 (Kyverno `verifyImages`) depends on this ADR being in place across every app whose image the cluster pulls.
