# Context

## Current State
Java Spring Boot scaffolder template deploys to AWS ECS Fargate via ECR. Shared AWS infrastructure deployed via CloudFormation (stack `backstage-shared-infra` in us-east-2). GitHub Actions plugin wired into CI/CD tab. CI/CD workflow fixed: added explicit task definition registration step before ECS service creation to prevent "TaskDefinition not found" error on first deploy.

## What Was Done
1. Created `examples/spring-boot-template/template.yaml` - Scaffolder template definition with wizard form (name, description, owner, javaPackage, springBootVersion, repoUrl)
2. Created template content skeleton under `examples/spring-boot-template/content/`:
   - `catalog-info.yaml` - Backstage catalog descriptor with `github.com/project-slug` annotation
   - `build.gradle` - Spring Boot + Gradle build with Java 21 toolchain
   - `settings.gradle` - Gradle settings
   - `gradle/wrapper/gradle-wrapper.properties` - Gradle 8.13 wrapper config
   - `Dockerfile` - Multi-stage build (Gradle build -> Eclipse Temurin 21 JRE)
   - `.gitignore` - Java/Gradle gitignore
   - `README.md` - Project documentation (updated with AWS deployment info)
   - `.github/workflows/ci-cd.yml` - GitHub Actions pipeline (build, test, deploy to AWS ECR + ECS Fargate)
   - `aws/task-definition.json` - ECS Fargate task definition template
   - `src/main/java/${{values.javaPackageDir}}/Application.java` - Spring Boot main class
   - `src/main/java/${{values.javaPackageDir}}/controller/HealthController.java` - Health endpoint
   - `src/main/resources/application.yml` - Spring config with actuator
   - `src/test/java/${{values.javaPackageDir}}/ApplicationTests.java` - Context load test
3. Added Spring Boot template location to `app-config.yaml` and `app-config.production.yaml` catalog locations
4. Installed `@backstage-community/plugin-github-actions` (v0.21.1) in frontend package
5. Updated `EntityPage.tsx` - Replaced CI/CD placeholder with GitHub Actions plugin
6. Created `aws/shared-infrastructure.yml` - CloudFormation template for shared infra (VPC, subnets, IGW, ECS cluster, security group, IAM role, CloudWatch log group)

## AWS Deployment Architecture
- Shared infra: CloudFormation stack `backstage-shared-infra` in us-east-2
  - VPC (10.0.0.0/16) with 2 public subnets
  - ECS cluster `backstage-apps`
  - Security group allowing TCP 8080 inbound
  - IAM `ecsTaskExecutionRole` with ECR pull + CloudWatch logs permissions
  - CloudWatch log group `/ecs/backstage-apps`
- Per-service: CI/CD creates ECR repo, registers task definition, creates/updates ECS service
- No ALB -- Fargate tasks get public IPs directly (demo tradeoff: IP changes on redeploy)
- Required GitHub secrets: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY

## What Does Not Exist Yet
- gradlew / gradlew.bat wrapper scripts (not needed for CI since setup-gradle uses gradle-version: '8.13')
- Kubernetes cluster configuration (separate from ECS deployment)
- Proper RBAC (still using allow-all policy)
- Production catalog sources beyond examples
- End-to-end testing of the full scaffolder flow

## Infrastructure Status
- AWS shared infrastructure: CloudFormation stack `backstage-shared-infra` deployed in us-east-2
- IAM user created with ECR + ECS permissions
- GitHub org secrets configured: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY (visibility: Public repositories only)
- GitHub fine-grained PAT (ca-backstage) created with admin, code, workflows permissions on all org repos
- ECR repos created automatically by CI/CD (comptest6, comptest7 exist)
- ECS cluster `backstage-apps` exists (created by CloudFormation stack)

## Next Steps
1. Scaffold a new repo (or re-run CI/CD on comptest7) to verify the task definition registration fix works end-to-end
2. Verify the deploy job succeeds: task def registered -> service created -> deployment stable
3. Test the full flow locally with `yarn start` (requires GITHUB_TOKEN env var)

## Deploy Command
```
aws cloudformation create-stack \
  --stack-name backstage-shared-infra \
  --template-body file://aws/shared-infrastructure.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-2
```
Wait for completion:
```
aws cloudformation wait stack-create-complete --stack-name backstage-shared-infra --region us-east-2
```
Verify outputs:
```
aws cloudformation describe-stacks --stack-name backstage-shared-infra --region us-east-2 --query "Stacks[0].Outputs"
```

## Recent Changes
- Fixed comptest7 deploy failure: "TaskDefinition not found" when creating ECS service. Root cause: the `amazon-ecs-render-task-definition@v1` action only modifies JSON locally, does not register with AWS. The create-service step ran before the deploy action could register it. Fix: added explicit `aws ecs register-task-definition` step between render and create-service, and changed create-service to use the registered task definition ARN instead of the family name.
- Diagnosed comptest6 deploy failure: CloudFormation stack `backstage-shared-infra` was never deployed to AWS. Build/test/ECR push all succeed. Failure at "Get shared infrastructure outputs" step.
- Fixed: Added `repoVisibility: public` to template.yaml publish:github step. Scaffolded repos were private by default, but org secrets on free plan are only available to public repos. This caused "Could not load credentials from any providers" in the deploy job.
- Created `aws/shared-infrastructure.yml` CloudFormation template for shared AWS infrastructure
- Created `examples/spring-boot-template/content/aws/task-definition.json` ECS task definition template
- Replaced CI/CD workflow: GHCR push -> ECR push + ECS Fargate deploy
- Updated template.yaml description and tags (added `aws` tag)
- Added spring-boot-template to `app-config.production.yaml` catalog locations
- Updated template README.md with AWS deployment docs and required secrets
