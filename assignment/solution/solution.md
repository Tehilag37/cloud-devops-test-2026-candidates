## 1. Kubernetes Availability and Resilience
### Current risks
The API currently runs with a single replica and has no startup or readiness probes, resource requests or limits, or defined graceful termination period.

Both worker nodes are located in the same availability zone. A zone failure could therefore interrupt the entire application, even if multiple API replicas are running.

### Proposed changes
**API Deployment**

The proposed Kubernetes manifest is provided in [api-deployment.yaml](../kubernetes/api-deployment.yaml).

It includes:

- Two API replicas to reduce the impact of an individual Pod failure.

- RollingUpdate with `maxUnavailable: 0` to maintain availability during updates, assuming sufficient cluster capacity.

- Startup, readiness and liveness probes to control startup, traffic routing and container restarts.

- CPU and memory requests and limits to improve resource scheduling and control.

- A 30-second termination grace period. The application must handle SIGTERM and finish in-flight requests gracefully.

- Topology spread constraints to distribute Pods across availability zones where suitable nodes exist.

- A PodDisruptionBudget with `minAvailable: 1` to protect against voluntary disruptions.

The health-check endpoints (`/health/live` and `/health/ready`) and container port `8000` are assumptions and must match the actual application. Resource values are initial examples and should be adjusted based on measurements.

**Zone-level resilience**

The current single-zone GKE configuration is a single point of failure.

To improve resilience:

- Use regional GKE Standard with worker nodes across multiple availability zones.

- Ensure sufficient node capacity in each zone.

- Distribute API replicas across zones.

- Configure Cloud SQL for regional High Availability, with a standby instance in another zone.

- Verify that load balancing and other required components remain available during a zone failure.

### Verification
- Validate the Kubernetes manifest before deployment.

- Deploy in a controlled environment and verify that all API replicas become Ready.

- Delete one API Pod and verify that Kubernetes creates a replacement and the service remains available.

- Perform a controlled node drain and verify that the PodDisruptionBudget protects the remaining available replica.

- Verify rolling updates and application health checks.

- Test zone-failure recovery through a planned resilience exercise.

The manifest is a proposed configuration. It has not been deployed to a live GKE cluster.

## 2. Backup and Recovery
### Current risk
Cloud SQL currently uses daily automated backups with seven-day retention, but Point-in-Time Recovery (PITR) is disabled. Therefore, a daily backup alone cannot reliably recover database changes made shortly before an accidental deletion.

The Cloud Storage bucket has neither Object Versioning nor Soft Delete enabled. Deleted objects may therefore be unrecoverable.

No recovery drill has been performed, so the RPO and RTO targets have not been verified.

### Proposed changes
**Cloud SQL (PostgreSQL)**

- Enable Point-in-Time Recovery (PITR) and configure transaction log retention to support recovery to a point before the incident.

- Keep automated backups enabled with at least seven days of retention.

- Use a separate restore instance during recovery; do not overwrite the production database immediately.

**Cloud Storage**

- Enable Soft Delete with a seven-day retention period so accidentally deleted objects can be recovered.

- Consider Object Versioning if recovery from object overwrites is also required.

- Review storage retention and lifecycle policies to avoid deleting recoverable data too early.

### Recovery procedure
Assume records and uploaded objects were accidentally deleted at 10:20 and the incident was discovered at 10:35.

1. **Stop further changes:** restrict writes to the affected service and pause background workers to prevent additional changes while recovery is in progress.

2. **Restore the database:** use Cloud SQL PITR to restore the database to a point immediately before the deletion (for example, 10:19:59). Restore to a new Cloud SQL instance, not directly over production.

3. **Recover files:** restore the deleted Cloud Storage objects from Soft Delete. Recover the affected objects while preserving their original object names and validating their generations.

4. **Check consistency:** verify that database records reference existing objects and that the recovered files belong to the expected surveys. Identify any legitimate changes made after the recovery point and reconcile them where possible.

5. **Reconnect the application:** update the application configuration or secret references to use the restored database instance. Verify connectivity and permissions.

6. **Validate before reopening:** run database integrity checks, verify file access, test API requests and report generation, and inspect application and worker logs.

7. **Resume service:** re-enable traffic and background workers only after validation succeeds. Monitor errors and performance closely.

### RPO and RTO
- **RPO target: 15 minutes.** PITR provides recovery to a selected point in time, reducing potential data loss compared with daily backups alone.

- **RTO target: 2 hours.** The recovery process must be rehearsed and timed, including database restoration, object recovery, application reconnection and validation.

- **File recovery target: 7 days.** Cloud Storage Soft Delete must retain deleted objects for at least seven days.

### Verification and limitations
Perform periodic recovery drills using a separate test environment and synthetic data. Record the selected recovery point, database restore duration, object recovery duration, validation results and total time until service is usable.

The targets should be considered achieved only after a successful timed drill.

A daily backup with seven-day retention alone does not guarantee a 15-minute RPO. PITR and adequate transaction log retention are required. Enabling Soft Delete also does not by itself prove that database records and files are consistent; both recovery paths must be tested together.

## 3. Linux Pipeline Agent – Disk Space Failure
### Cause and investigation
The Ubuntu pipeline agent has run out of disk space. The root filesystem is 100 GB and is 100% used.

The reported usage is:

- `/var/lib/docker`: 76 GB

- `/var/lib/other`: 4 GB

- `/opt/ado-agent/\_work`: 12 GB

Docker data is the largest identified consumer. The agent workspace may also contain old source checkouts, build outputs and temporary files.

Before deleting anything, I would identify which data is safe to remove and verify that no pipeline job or running container depends on it.

### Investigation commands
```bash

# Check filesystem capacity and inode usage

df -h /

df -i /

# Find the largest directories on the root filesystem

sudo du -xhd1 /var/lib | sort -h

sudo du -xhd1 /opt/ado-agent | sort -h

# Inspect Docker disk usage

docker system df -v

# List containers, including their sizes

docker ps -a --size

# Inspect Docker volumes before considering any cleanup

docker volume ls

# Check Azure DevOps agent workspaces

sudo du -sh /opt/ado-agent/\_work/* 2>/dev/null

# Check system journal size

sudo journalctl --disk-usage

```

### Safe immediate recovery
1. Pause new pipeline jobs on the affected agent and verify that no build or deployment is currently running.

2. Review Docker disk usage and identify unused images and build cache.

3. Remove only confirmed unused Docker resources. For example:

```bash

# Remove dangling images not used by containers

docker image prune

# Remove build cache older than seven days

docker builder prune --filter until=168h

```

4. Review old Azure DevOps workspaces. Remove only workspaces belonging to completed jobs, preferably through the agent's supported workspace cleanup mechanism.

5. Recheck disk usage:

```bash

df -h /

docker system df

```

6. Resume the agent and run a small test build.

### What must not be deleted blindly
- Do not remove Docker volumes without verifying their contents and ownership.

- Do not delete files used by running containers or active pipeline jobs.

- Do not remove the entire Docker directory manually.

- Do not delete the complete Azure DevOps `\_work` directory while jobs are running.

- Review logs and build artifacts before deleting them.

### Prevention
- Configure Azure DevOps workspace cleanup so old job data does not accumulate.

- Periodically prune unused Docker images and build cache using a controlled maintenance process.

- Monitor disk usage and alert before the filesystem reaches 80–90%.

- Review disk usage trends and increase the agent disk size if normal workloads consistently exceed available capacity.

- Keep cleanup automated, limited to confirmed unused data and protected from running jobs.

### Verification
After cleanup, verify that disk space is available with `df -h /` and that Docker reports reduced usage with `docker system df`.

Run a test pipeline build and confirm that the agent can complete it successfully.

These commands and procedures are proposed for the Ubuntu agent. They have not been executed against the assignment's production VM.

## 4. Release Pipeline and Failed Deployment Recovery

### Current risks

The current pipeline deploys images tagged as `latest`. This tag can point to different images over time, so it does not provide a reliable way to identify or restore the exact version that was previously deployed.

There is also no production approval step. A failed application release may occur after a database migration has already completed.

### Proposed pipeline flow

The pipeline should include the following stages:

1. **Build:** build the API image and tag it with the Git commit SHA (`Build.SourceVersion`) instead of `latest`.

2. **Validate:** run automated tests and validate the Kubernetes manifests. Validate database migrations before production deployment.

3. **Push:** push the image to Artifact Registry and record its immutable image digest (`sha256`).
Publish the release metadata, including the image digest, as a pipeline artifact or output variable so the Production stage can retrieve and deploy the exact image.

4. **Production approval:** require an authorized person to approve the production deployment through an Azure DevOps Environment approval check.

5. **Deploy:** run the database migration and deploy the exact image digest to GKE.

6. **Verify:** wait for the rollout to complete and run health checks or smoke tests. Record the deployed image digest, commit SHA, build ID and migration version.

Build the image once and promote the same image to production. Do not rebuild it during deployment.

### Pipeline example (illustrative pseudocode)

The following excerpt describes the intended Azure DevOps pipeline flow. It is pseudocode, not a ready-to-run pipeline: registry details, service configuration and test commands must be filled in for the actual environment.

```yaml

trigger:

  branches:

    include:

      - main

stages:

  - stage: Build

    jobs:

      - job: BuildAndValidate

        steps:

          - checkout: self

          - script: |

              # Run unit tests and validate Kubernetes manifests

              run-tests

              validate-kubernetes-manifests

              # Build and push an image with a unique commit-based tag

              IMAGE_TAG=$(Build.SourceVersion)

              build-image "$IMAGE_TAG"

              push-image "$IMAGE_TAG"

              # Resolve and save the registry image digest

              IMAGE_DIGEST=$(get-image-digest "$IMAGE_TAG")

              save-release-metadata "$IMAGE_TAG" "$IMAGE_DIGEST"

            displayName: Build, validate and push image

  - stage: Production

    dependsOn: Build

    jobs:

      - deployment: DeployToProduction

        environment: production

        strategy:

          runOnce:

            deploy:

              steps:

                - script: |

                    load-release-metadata

                    # Run a backward-compatible database migration

                    run-database-migration

                    # Deploy the exact image built in the previous stage

                    kubectl set image deployment/survey-api api="$(IMAGE_URI)@$(IMAGE_DIGEST)"

                    # Wait for rollout and verify application health

                    kubectl rollout status deployment/survey-api

                    run-smoke-tests

                  displayName: Migrate, deploy and verify


```

### Failed release recovery

If the application release fails after deployment:

1. Pause further production deployments and investigate the failure.
2. Retrieve the last known-good image digest from the deployment record.
3. Check whether the completed database migration is compatible with the previous application version.
4. If compatible, redeploy the previous application image using its exact digest.
5. Wait for the rollout to complete and verify readiness, health checks and application logs.
6. Record the failed and restored image digests, build IDs and migration version.

Database migrations should follow an expand-and-contract approach:
- Introduce backward-compatible schema changes first.
- Deploy the application version that uses the new schema.
- Remove obsolete schema elements only after the old application version is no longer needed.

If the migration is incompatible with the previous application version, do not simply roll back the application. Prefer a forward fix that restores compatibility. A database restore should be considered only after assessing potential data loss and coordinating the recovery.

## 5. Git Branching and Access Controls

### Current risks

Developers can currently push directly to the `main` branch, and any push may trigger a production deployment.

This creates a risk of unreviewed code reaching production, without automated validation or an explicit release decision.

### Proposed Git workflow

Use a lightweight branch-based workflow:

1. Developers create short-lived feature branches from `main`.

2. Changes are pushed to the feature branch.

3. A Pull Request (PR) is opened to merge the changes into `main`.

4. Automated build and validation checks must pass.

5. At least one authorized reviewer approves the PR.

6. After approval and successful checks, the PR can be merged into `main`.

7. The production pipeline runs from `main` and requires a separate production approval.

### Azure DevOps branch policies

Configure the following policies on the `main` branch:

- **Minimum number of reviewers:** require at least one reviewer.

- **Build validation:** require the validation pipeline to pass before allowing a merge.

- **Comment resolution:** require PR comments to be resolved.

- **Direct push restrictions:** remove direct push permissions for regular developers and allow changes through PRs.

- **Review requirements:** ensure that changes are reviewed before they are merged.

Production deployment permissions and approval checks should be configured separately from Git branch policies.

### Pipeline controls

- Configure the pipeline to trigger production releases only from `main`.

- Do not deploy feature branches directly to production.

- Require an authorized approval before the production deployment stage.

- Keep deployment permissions limited to the appropriate team members.

- Record the build, commit and deployed image digest for traceability.

### Urgent fixes

Urgent production fixes should follow the same controlled process:

1. Create a short-lived hotfix branch from the relevant production version or `main`, depending on the situation.

2. Make the smallest required change.

3. Run the required validation checks.

4. Obtain an expedited review from an authorized reviewer.

5. Merge through a Pull Request and deploy through the approved production pipeline.

6. Verify the fix and retain the deployment record.

Emergency handling may use an expedited approval process, but changes should remain traceable and reviewed. Direct pushes to `main` should not become the normal emergency procedure.

### Verification

- Verify that a developer without the required permissions cannot push directly to `main`.

- Open a test PR and confirm that merging is blocked until required checks and approvals pass.

- Verify that a feature branch push does not trigger a production deployment.

- Confirm that production deployment requires an authorized approval.

- Verify that hotfixes follow the same review and deployment controls.
