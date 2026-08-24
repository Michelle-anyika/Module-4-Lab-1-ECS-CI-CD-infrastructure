# ECS CI/CD Lab — Infrastructure

CloudFormation templates for a highly available, containerized Java web app
on ECS Fargate, with a fully automated blue/green CI/CD pipeline. Deployed
via **CloudFormation GitSync** (no manual console deploys after initial
setup).

Application source code, Dockerfile, and the GitHub Actions build workflow
live in a **separate repository** — see the top-level `README.md` one level
up for the link. This split matches the lab's own deliverable split
(infrastructure repo vs. application repo) and keeps IaC changes and app
changes independently reviewable/deployable.

## Templates (deploy in this order)

1. **`templates/network.yaml`** — VPC across 2 AZs: 2 public subnets (ALB),
   2 private subnets (ECS tasks). No NAT Gateway — private subnets reach
   ECR and CloudWatch Logs purely through VPC endpoints
   (`ecr.api`, `ecr.dkr`, `logs` interface endpoints + an `s3` gateway
   endpoint for ECR's layer storage).
2. **`templates/ecs-service.yaml`** — ECR repo (immutable tags), ECS
   cluster, Fargate task definition + service (`DeploymentController: CODE_DEPLOY`),
   public ALB with a prod listener (`:80`, public) and a test listener
   (`:8080`, VPC-only) each pointing at its own target group (blue/green),
   and CPU-based Application Auto Scaling (1–4 tasks, target 50% CPU).
   Imports network outputs via `Fn::ImportValue`.
3. **`templates/pipeline.yaml`** — S3 artifact bucket, CodeDeploy
   application/deployment group (blue/green, `WITH_TRAFFIC_CONTROL`),
   CodePipeline (ECR + S3 sources → `CodeDeployToECS` deploy action), and
   the EventBridge rule that starts the pipeline on every ECR image push.
   Imports outputs from `ecs-service.yaml`.

Each template is independently deployable and maps to its own stack — this
is what makes them work cleanly with GitSync (one tracked template per
stack) while still being cross-referenced via CloudFormation exports/imports
instead of nested-stack S3 packaging.

## One-time AWS bootstrap

- **CodeDeploy blue/green requires a bootstrap image in ECR before the
  service can be created.** `ecs-service.yaml`'s `ContainerImage` parameter
  defaults to a public placeholder (`public.ecr.aws/nginx/nginx:latest`) so
  the stack deploys standalone; the app repo's first successful pipeline run
  replaces it with the real image via CodeDeploy — no manual step needed
  here.
- **GitHub OIDC role** for the app repo's build workflow — see the app
  repo's README for the exact trust policy. It needs permission to push to
  ECR, and to `s3:PutObject` into this stack's `ArtifactBucket` (see the
  `pipeline.yaml` outputs after deploying it).
- **IAM role names are fixed** (`ecs-cicd-execution-role`,
  `ecs-cicd-task-role`, `ecs-cicd-codedeploy-role`, `ecs-cicd-codepipeline-role`,
  `ecs-cicd-eventbridge-role`) rather than CloudFormation-generated, so the
  app repo's static `taskdef.json` can reference the execution/task role
  ARNs directly without needing to query the infra stack at build time.

## Deploying via CloudFormation GitSync

1. Push this `infrastructure/` repo to GitHub.
2. In the CloudFormation console: **Stacks → Create stack → With new
   resources → Sync from Git**.
3. Connect the GitHub repo/branch (creates a CodeConnections connection on
   first use).
4. Create **three** GitSync-tracked stacks, one per template, using the
   default stack names referenced by the templates' parameters:
   - `ecs-cicd-network` → `templates/network.yaml`
   - `ecs-cicd-service` → `templates/ecs-service.yaml` (parameter
     `NetworkStackName=ecs-cicd-network`)
   - `ecs-cicd-pipeline` → `templates/pipeline.yaml` (parameter
     `ServiceStackName=ecs-cicd-service`)
5. Deploy them in that order the first time (each depends on the previous
   stack's exports). After that, GitSync redeploys automatically on every
   push that touches a tracked template.

## Live demonstration

1. **ALB endpoint**: `templates/ecs-service.yaml` output `ALBDNSName` —
   `curl http://<ALBDNSName>/` returns the app's name/lab HTML.
2. **Automated redeploy**: push a change to the app repo → GitHub Actions
   builds/pushes a new immutable image tag → EventBridge fires →
   CodePipeline runs → CodeDeploy stands up a Green task set behind
   `:8080`, validates it, flips the `:80` prod listener to Green, then
   terminates the old Blue task set.
3. **Watch it happen**: CodeDeploy console → the deployment group's
   deployment shows *Install → AllowTestTraffic → AllowTraffic (Blue→Green
   switch) → Terminate original*; CodePipeline console shows the same run
   end-to-end; ECS console → Service → **Tasks** tab shows the new task
   registering healthy in the target group before the old one is
   deregistered.
4. **Logs**: CloudWatch Logs group `/ecs/ecs-cicd-app` shows both task
   sets' output during the transition.
5. **Auto scaling**: drive load at the ALB endpoint (e.g. `hey`/`ab`/`wrk`)
   to push CPU over 50%; Application Auto Scaling adds tasks (up to 4) via
   the `CpuScalingPolicy` target-tracking policy, visible in the ECS
   service's **Metrics**/**Events** tabs.

## Security and cost-optimization notes

- No NAT Gateway: private ECS tasks reach only ECR and CloudWatch Logs, both
  served by VPC endpoints, so there's no need to pay for or expose a NAT
  path to the broader internet.
- ALB test listener (`:8080`) is restricted to the VPC CIDR, not
  `0.0.0.0/0` — it exists solely for CodeDeploy's pre-swap validation
  traffic, not for end users.
- ECS security group accepts traffic only from the ALB security group, never
  a raw CIDR.
- ECR repository uses `ImageTagMutability: IMMUTABLE` and a lifecycle policy
  capping retained images at 20, bounding both drift risk and storage cost.
- IAM roles are scoped per function (execution vs. task vs. CodeDeploy vs.
  CodePipeline vs. EventBridge) rather than one shared broad role.
