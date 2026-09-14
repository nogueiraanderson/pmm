# Packer templates for the pmm Jenkins agents

`aws.pkr.hcl` bakes the two `pmm-worker-3` agent AMIs (amd64 and arm64, AlmaLinux 9)
that the Docker Farm cloud on pmm.cd.percona.com and the pmm-staging pipelines
launch. `do.pkr.hcl` builds the DigitalOcean `pmm-ovf-agent` snapshot (unused since
June 2026, left as is). `pmm.json` is the legacy JSON template.

## The factory (`.github/workflows/agent-ami-factory.yml`)

The AWS AMIs are built by GitHub Actions, not by hand:

1. `check`: `scripts/check.sh` (fmt, validate both templates, playbook syntax with
   the runner collections from `ansible/requirements.yml`). Runs on every PR that
   touches this directory. No AWS access.
2. `bake` (one job per architecture): assumes the bake role
   `percona-ci-platform-gha-pmm-agent-ami-bake` (Terraform in
   `percona-cd-platform`, `terraform/iam-gha-pmm-agent-ami-factory.tf`), prunes
   candidates older than seven days and incomplete bakes older than one day, runs
   `packer build -only=jenkins-farm.amazon-ebs.<arch>-agent aws.pkr.hcl` over AWS
   Session Manager (no inbound SSH, egress-only security group, builder instance
   profile). The playbook runs on the builder itself: Packer uploads `ansible/`,
   installs the pinned `ansible-core` there and runs it with a local connection.
   Ansible through Packer's SSH proxy adapter was tried first and is not usable
   here, the adapter connection dies part way through the playbook and the build
   then hangs until the Session Manager idle timeout tears the tunnel down.
   Then `smoke/smoke.pkr.hcl` boots the candidate on a fresh instance and
   asserts Java 21 default plus Java 17 present, Docker usable by `ec2-user`, sshd
   and the cloud-init launch key (the consumers connect over SSH), SSM agent
   enabled, the dnf-makecache timer disabled, the tool set present. A failed smoke
   keeps the candidate tagged `role=smoke-failed` for a look, the janitor removes it
   a week later.
3. `promote` (one job, both architectures, only when both smokes passed, GitHub
   environment `pmm-agent-ami-factory-prod`, promote role
   `percona-ci-platform-gha-pmm-agent-ami-promote` that trusts only the environment
   subjects): checks read-only that each candidate is newer than every current live
   image, tags the candidate and its snapshots `iit-billing-tag=pmm-worker-3`, waits
   until the consumers' query returns it as the newest live image, then demotes the
   previous live image and its snapshots to `pmm-worker-3-previous` with a 30-day
   deprecation date, and asserts exactly one live image per architecture.

Triggers: push to `main` on the recipe paths, weekly Monday 05:00 UTC, manual
dispatch from `main` with `env=prod` or `env=test`. `env=test` uses the tag family
`pmm-worker-3-test*`, which no consumer selects, so a test bake never reaches
production. Before the first run a repository admin creates the two GitHub
environments (`pmm-agent-ami-factory-prod` with required reviewers,
`pmm-agent-ami-factory-test`) and the two repository secrets
`PMM_AGENT_AMI_BAKE_ROLE_ARN` and `PMM_AGENT_AMI_PROMOTE_ROLE_ARN` (the role ARNs are
Terraform outputs in `percona-cd-platform`, this public repo carries no account id).
Without the environments the prod gate is a plain job, without the secrets the bake
and promote jobs fail before touching AWS. The repository's Actions allowlist admits
no `hashicorp/*` or `aws-actions/*` actions, so the workflow installs Packer from the
verified release zip and assumes the roles with `aws sts assume-role-with-web-identity`.

### Tag contract

| `iit-billing-tag` | Meaning |
|---|---|
| `pmm-agent-ami-factory` | builder and smoke instances, volumes, key pairs, and the AMI for the seconds between CreateImage and its candidate tags |
| `pmm-worker-3-candidate` | baked, not yet smoke-tested or promoted (snapshots carry the same value) |
| `pmm-worker-3` | live pair, selected by the Docker Farm templates and the staging pipelines (newest `CreationDate` wins), snapshots follow |
| `pmm-worker-3-previous` | demoted predecessor, rollback target, deprecates 30 days after demotion, snapshots follow |

Consumers filter by tag plus the `architecture` attribute and pick the newest
image, so a promotion needs no configuration change on pmm.cd. The bake role can
delete only candidates, incomplete bakes and the test family. Nothing can delete a
live or previous prod image without admin credentials.

### Rollback

Order matters because the consumers take the newest live image: demote the bad
image first, then promote the rollback target. The promote role allows exactly
these transitions (previous to live included), so this works from a
`workflow_dispatch`-less shell with that role or with admin credentials:

```
bad=<bad-ami>; good=<good-ami>
snaps() { aws ec2 describe-images --region us-east-2 --image-ids "$1" --query 'Images[0].BlockDeviceMappings[].Ebs.SnapshotId' --output text; }
aws ec2 create-tags --region us-east-2 --resources "$bad" $(snaps "$bad") --tags Key=iit-billing-tag,Value=pmm-worker-3-previous Key=role,Value=previous
aws ec2 create-tags --region us-east-2 --resources "$good" $(snaps "$good") --tags Key=iit-billing-tag,Value=pmm-worker-3 Key=role,Value=live
aws ec2 disable-image-deprecation --region us-east-2 --image-id "$good"
```

Between the two calls that architecture has no live image for about a second.

### Emergency manual bake

Same templates, from a laptop with admin credentials and the workflow's pinned
Packer version (the manual path has no smoke test or promotion, so tag by hand
afterwards):

```
packer init aws.pkr.hcl
packer build -only=jenkins-farm.amazon-ebs.amd64-agent -var env=test aws.pkr.hcl
```

`env=test` keeps a hand bake out of production until you retag it.

### Known follow-ups

- The two hand-baked AMIs from 16 August 2026 become `pmm-worker-3-previous` on the
  first promotion. Their snapshots carry the live tag and are retagged with them.
  Nothing prunes `pmm-worker-3-previous` images yet, they deprecate but stay.
- The staging helpers `launchSpotInstance.groovy` and `runSpotInstance.groovy` in
  jenkins-pipelines read `Images[0]` without sorting. The promote job guarantees one
  live image per architecture, a `sort_by(CreationDate)` there would remove the
  dependency.
- Dependabot covers the workflow's actions but not the Packer plugin pins in
  `aws.pkr.hcl` and `smoke/smoke.pkr.hcl`. Bump them by hand.
- Every job calling bare `java` on these agents now gets Java 21. Java 17 stays
  installed under `/usr/lib/jvm/java-17-openjdk*` for jobs that address it by path.
- `do.pkr.hcl` (DigitalOcean) and `pmm.json` are untouched and still describe Java 17.

## Logging

```
PACKER_LOG_PATH="packer.log" PACKER_LOG=1 packer build -only=jenkins-farm.amazon-ebs.amd64-agent aws.pkr.hcl
PACKER_LOG_PATH="packer.log" PACKER_LOG=1 packer build -color=false do.pkr.hcl
```
