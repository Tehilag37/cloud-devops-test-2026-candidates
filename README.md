# GCP Cloud DevOps Engineer – Assignment Solution

## Overview

This repository contains my proposed solution to the GCP Cloud DevOps Engineer practical assignment.

The solution addresses five areas of the existing production environment:

1. Kubernetes availability and service resilience.
2. Database and Cloud Storage backup and recovery.
3. Linux pipeline agent disk space management.
4. Azure DevOps release pipeline and failed deployment recovery.
5. Git branching strategy and access controls.

## Assignment

[View the original assignment](assignment/ASSIGNMENT.md)

## Solution and Deliverables

| File | Description |
|---|---|
| [Solution document](assignment/solution/solution.md) | Analysis, proposed fixes, recovery procedures and verification steps for all five problems |
| [Kubernetes Deployment](assignment/kubernetes/api-deployment.yaml) | API Deployment configuration and PodDisruptionBudget |
| [Architecture diagram](diagrams/architecture.mmd) | Proposed target architecture and system connections |

## Key Design Decisions

- Run multiple API replicas with health probes and resource requests and limits.
- Distribute GKE workloads across multiple zones.
- Use Cloud SQL regional High Availability and Point-in-Time Recovery.
- Enable Cloud Storage Soft Delete with a seven-day retention period.
- Use immutable container image versions and record deployed image digests.
- Require Pull Request reviews, automated validation and production approval.

## Assumptions and Validation

The proposed changes are design recommendations for the fictional environment described in the assignment.

No GCP deployment or production changes were performed. Configuration values and health-check endpoints in the Kubernetes example must be verified against the actual application before deployment.

The pipeline excerpt is illustrative pseudocode and requires adaptation to the actual Azure DevOps and GCP environment.

AI assistance was used to support research, technical explanations
and solution drafting. The proposed solution was reviewed and
adapted, and the author is responsible for understanding its contents.

## Estimated Time Spent

Approximately 8 hours.