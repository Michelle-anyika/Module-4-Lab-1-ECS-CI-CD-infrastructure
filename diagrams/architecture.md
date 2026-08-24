# Architecture Diagram — ECS CI/CD Lab

Diagram-as-code (Mermaid) describing the network, runtime, and CI/CD paths.
Renders natively on GitHub and in any Mermaid-aware viewer.

```mermaid
flowchart TB
    subgraph GitHub["GitHub"]
        AppRepo["App repo: source + Dockerfile + ecs/appspec+taskdef"]
        GHA["GitHub Actions\n(OIDC auth, no long-lived keys)"]
        AppRepo -- "push to main" --> GHA
    end

    subgraph AWS["AWS Account / Region"]
        ECR["Amazon ECR\nIMMUTABLE tags (git SHA)"]
        EB["EventBridge rule\nECR Image Action: PUSH"]
        S3A["S3 artifact bucket\npipeline-templates/templates.zip"]

        subgraph Pipeline["CodePipeline"]
            SrcECR["Source: ECR image"]
            SrcS3["Source: S3 templates.zip"]
            Deploy["Deploy: CodeDeployToECS"]
            SrcECR --> Deploy
            SrcS3 --> Deploy
        end

        CD["CodeDeploy\nBLUE_GREEN / WITH_TRAFFIC_CONTROL"]

        subgraph VPC["VPC (multi-AZ, 10.2.0.0/16)"]
            subgraph PublicAZ1["Public subnet AZ-1"]
                ALB["Application Load Balancer\ninternet-facing"]
            end
            subgraph PublicAZ2["Public subnet AZ-2"]
                ALB2["(ALB continues here)"]
            end

            ProdListener["Listener :80 (prod)"]
            TestListener["Listener :8080 (test, VPC-only)"]

            subgraph PrivateAZ1["Private subnet AZ-1"]
                TaskBlue["Fargate task (Blue)\nECS Service"]
            end
            subgraph PrivateAZ2["Private subnet AZ-2"]
                TaskGreen["Fargate task (Green)\nnew revision under test"]
            end

            VPCE["VPC Endpoints:\necr.api / ecr.dkr / logs / s3 (gateway)\nno NAT Gateway"]
        end

        Logs["CloudWatch Logs\n/ecs/ecs-cicd-app"]
    end

    Users["End users"] --> ALB
    ALB --> ProdListener --> TaskBlue
    ALB --> TestListener --> TaskGreen

    GHA -- "docker push (OIDC role)" --> ECR
    GHA -- "upload rendered taskdef/appspec" --> S3A
    ECR -- "push event" --> EB
    EB -- "StartPipelineExecution" --> Pipeline
    S3A --> SrcS3
    ECR --> SrcECR
    Deploy --> CD
    CD -- "register new task def,\nshift traffic Blue -> Green,\nthen terminate old Blue" --> TaskGreen
    TaskBlue -.-> VPCE
    TaskGreen -.-> VPCE
    TaskBlue --> Logs
    TaskGreen --> Logs
```

## Key points the diagram encodes

- **Two repos, two responsibilities**: the *app* repo owns build/push (GitHub
  Actions, OIDC to AWS, no static keys); the *infrastructure* repo owns
  everything else (network, ECS, ALB, pipeline), deployed via CloudFormation
  GitSync.
- **No NAT Gateway**: ECS tasks are fully private and reach ECR/CloudWatch
  Logs only through VPC interface/gateway endpoints — lower cost, smaller
  network attack surface than routing egress through a NAT Gateway.
- **Push-triggered pipeline**: EventBridge — not polling — starts
  CodePipeline the moment a new image lands in ECR.
- **Blue/green, not rolling**: CodeDeploy stands up a full new (Green) task
  set behind the test listener (port 8080, VPC-only), validates it, flips
  the prod listener (port 80) to Green, then tears down the old Blue set —
  so a bad deploy can be caught before it ever reaches port 80.
