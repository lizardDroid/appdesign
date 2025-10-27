# Security Posture

- **Non-root**: `runAsNonRoot: true`, UID/GID 1000.
- **Read-only root FS**: `readOnlyRootFilesystem: true`; writable `/tmp` via `emptyDir`.
- **No privilege escalation**: `allowPrivilegeEscalation: false`.
- **Drop all Linux caps**: `capabilities.drop: ["ALL"]`.
- **Seccomp**: `seccompProfile: { type: RuntimeDefault }`.
- **NetworkPolicy**: optional; enable default-deny egress with DNS + required egress rules in production.
- **ServiceAccount**: dedicated with **no** Roles/ClusterRoles.
- **Image pinning**: prefer digests for reproducibility (`values.image.digest`).

Compatible with **Pod Security Admission: restricted** (assuming the image supports non-root and `/tmp` writes only).
