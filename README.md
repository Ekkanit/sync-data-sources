# sync-data-sources (SDS)

Single go binary that will manage Grimoire stack data gathering using configuration fixtures from dev-analytics-api


# Docker images

- First you will need a base image containing grimoire stack tools needed, follow instructions on `LF-Engineering/dev-analytics-grimoire-docker` to build minimal image.
- Use: `DOCKER_USER=docker-user BRANCH=test [API_REPO_PATH="$HOME/dev/LF/dev-analytics-api"] [SKIP_BUILD=1] [SKIP_PUSH=1] [SKIP_PULL=1] [PRUNE=1] ./docker-images/build.sh` to build docker image.
- Use: `DOCKER_USER=docker-user [SKIP_BUILD=1] [SKIP_PUSH=1] [PRUNE=1] ./docker-images/build-validate.sh` to build docker image file (validation image),
- Use: `DOCKER_USER=docker-user BRANCH=test|prod [PRUNE=1] ./docker-images/remove.sh` to remove docker image locally (remote version is not touched).
- Use: `` DOCKER_USER=docker-user BRANCH=test|prod [REPO_ACCESS="`cat repo_access.secret`"] ./docker-images/test_image_docker.sh [command] `` to test docker image locally. Then inside the container run: `./run.sh`.
- Use: `` DOCKER_USER=docker-user BRANCH=commit_SHA [REPO_ACCESS="`cat repo_access.secret`"] ./docker-images/test_validate_image_docker.sh [command] `` to test docker image locally. Then inside the container run: `./run.sh`.


# Manual docker run example

- Get SortingHat DB endpoint: `prodk.sh -n mariadb get svc mariadb-service-rw`.
- Get SortingHat DB credentials: `for f in helm-charts/sds-helm/sds-helm/secrets/SH_*.prod.secret; do echo -n "$f: "; cat $f; echo ""; done`.
- Get other env variables: `prodk.sh -n sds edit cj sds-0`.
- Finally just run `[EXEC=1] ./docker-images/manual_docker.sh prod`. If you specify EXEC=1 you will get bash instead of running SDS, so you can call `./run.sh` manually.
- Validation `[EXEC=1] ./docker-images/manual_docker_validate.sh commit_sha`. If you specify EXEC=1 you will get bash instead of running SDS, so you can call `./run.sh` manually.
- To see environment inside the container: `clear; env | sort | grep 'SDS\|SH_'`.
- Send info signal `` kill -SIGUSR1 `ps -ax | grep syncdatasources | head -n 1 | awk '{print $1}'` ``.
- To shell into the running SDS: `./docker-images/shell_running_sds.sh`.


# Kubernetes

- Use: `DOCKER_USER=docker-user BRANCH=test|prod ./kubernetes/test_image_kubernetes.sh [command]` to test docker image on kubernetes (without Helm chart). Then inside the container run: `./run.sh`.


# Locally

- Use: `cd sources; make; ./dry_run.sh prod` even if you don't have correct `p2o.py` stack installed.


# Helm

- Go to `helm-charts/sds-helm` and follow instructions from `README.md` file in that derectory. Helm chart contains everything needed to run entire stack.
- Note that `zippass.secret` and `helm-charts/sds-helm/sds-helm/secrets/ZIPPASS.secret` files should have the same contents.
- See documentation [here](https://github.com/LF-Engineering/sync-data-sources/blob/master/helm-charts/sds-helm/README.md).

# In short

- Install: `[NODES=n] [NS=sds] ./setup.sh test|prod`.
- Unnstall: `[NS=sds] ./delete.sh test|prod`.
- Other example (with external ES): `` NODES=2 NS=sds-ext FLAGS="esURL=\"`cat sds-helm/secrets/ES_URL_external.secret`\",pvSize=30Gi" ./setup.sh test ``.

# Debug

- If not installed with the Helm chart (which is the default), for the `test` env do: `cd helm-charts/sds-helm/`, `[NODES=n] [NS=sds] ./debug.sh test`, `pod_shell.sh test sds sds-debug-0` to get a shell inside `sds` deployment. Then `./run.sh`.
- When done `exit`, then copy CSV: `testk.sh -n sds cp sds-debug-0:root/.perceval/tasks_0_1.csv tasks_test.csv`, finally delete debug pod: `[NS=sds] ./debug_delete.sh test`.

# Fargate

- Create cluster via: `AWS_PROFILE=darst ./fargate/create_cluster.sh test sds-cluster`.
- List clusters via: `AWS_PROFILE=darst ./fargate/list_clusters.sh`.
- Create role for `awslogs` driver via: `AWS_PROFILE=darst ./fargate/create_role.sh`.
- List roles via: `AWS_PROFILE=darst ./fargate/list_roles.sh`. Note `"Arn": "arn:aws:iam::XXXXXXXXXXX:role/ecsTaskExecutionRole"`.
- Create task via: `[DRY=1] [DRYSDS=1] AWS_PROFILE=darst AWS_REGION=us-west-2 SDS_ROLE_ARN='arn:aws:iam::XXXXXX:role/ecsTaskExecutionRole' SDS_TASK_NAME=sds-projname ./fargate/create_task.sh test`.
- You can put this role ARN in a scret file, to avoid specifying it every time: `helm-charts/sds-helm/sds-helm/secrets/SDS_ROLE_ARN.secret`.
- Example task for `odpi/egeria` fixture and `git` datasource: `DRY='' DRYSDS='' AWS_PROFILE=darst AWS_REGION=us-west-2 SDS_TASK_NAME=sds-egeria-git SDS_FIXTURES_RE='^odpi/egeria$' SDS_DATASOURCES_RE='^git$' ./fargate/create_task.sh prod`.
- List tasks via: `AWS_PROFILE=darst ./fargate/list_tasks.sh`.
- Create VPC via: `AWS_PROFILE=darst ./fargate/create_vpc.sh`.
- List VPCs via: `AWS_PROFILE=darst ./fargate/list_vpc.sh`. Note `VpcId` for newly created VPC (it has `"CidrBlock": "10.0.0.0/16"`). Put that value in `helm-charts/sds-helm/sds-helm/secrets/SDS_VPC_ID.secret` file.
- Create subnet via: `AWS_PROFILE=darst ./fargate/create_subnet.sh`.
- List subnets via: `AWS_PROFILE=darst ./fargate/list_subnet.sh`. Note `VpcId` for newly created subnet (it has `"CidrBlock": "10.0.128.0/17"`). Put that value in `helm-charts/sds-helm/sds-helm/secrets/SDS_SUBNET_ID.secret` file.
- Create VPC security group via: `AWS_PROFILE=darst ./fargate/create_security_group.sh`.
- List security groups via: `AWS_PROFILE=darst ./fargate/list_security_group.sh`.
- Create service via: `[PUB=1] [SDS_VPC_ID=...] AWS_PROFILE=darst ./fargate/create_service.sh test sds-cluster sds-projname sds-projname-service`.
- List services via: `AWS_PROFILE=darst ./fargate/list_services.sh test sds-cluster`.
- Describe service via: `AWS_PROFILE=darst ./fargate/describe_service.sh test sds-cluster sds-projname-service`.
- Eventually delete security group via: `AWS_PROFILE=darst ./fargate/delete_security_group.sh`.
- Eventually delete subnet via: `AWS_PROFILE=darst ./fargate/delete_subnet.sh`.
- Eventually delete vpc via: `AWS_PROFILE=darst ./fargate/delete_vpc.sh`.
- Eventually delete role via: `AWS_PROFILE=darst ./fargate/delete_role.sh`.
- Eventually delete task via: `AWS_PROFILE=darst fargate/delete_task.sh test sds-projname`
- Eventually delete service via: `AWS_PROFILE=darst ./fargate/delete_service.sh test sds-cluster sds-projname-service`.
- Eventually delete cluster via: `AWS_PROFILE=darst ./fargate/delete_cluster.sh test sds-cluster`.
