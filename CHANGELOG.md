# Changelog

All notable changes to this chart are documented in this file.

## [Unreleased]
- Add GitHub Actions workflow for Helm lint and kubeconform validation (strict, Kubernetes 1.28).
- Use repository name as Helm release in CI (no hardcoded release name).
- Add “Best practices used in this chart” section to README.

## [0.2.0]
- Initial version in this repository snapshot with Argo Rollouts, PDB, HPA, anti-affinity, topology spread, secure defaults, and optional NetworkPolicies/Ingress.
