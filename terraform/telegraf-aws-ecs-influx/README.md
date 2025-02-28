# telegraf

## overview

This terraform code configure AWS ECS containers that log TC data to InfluxDB.

Most of the work is done via `aws_ecs_service` and `aws_ecs_task_definition` instances. search for those.

This data is used in grafana dashboard. see https://github.com/mozilla-platform-ops/dashboards/tree/master/dashboards_2.

### details


####  map of tf files to `aws_ecs_service` names

```
ecs-cluster.tf:
  52: resource "aws_ecs_task_definition" "app" {
  53    family                   = "telegraf"

ecs-queues.tf:
  17: resource "aws_ecs_task_definition" "app_queues" {
  38:       { "name" : "TELEGRAF_CONFIG", "value" : "telegraf_queues.conf" },

ecs-vcs.tf:
  16
  17: resource "aws_ecs_task_definition" "app_vcs" {
  38:       { "name" : "TELEGRAF_CONFIG", "value" : "telegraf_vcs.conf" },

ecs-workers.tf:
  17: resource "aws_ecs_task_definition" "app_workers" {
  38:       { "name" : "TELEGRAF_CONFIG", "value" : "telegraf_workers.conf" },
```

#### docker image ecr versions

```
telegraf: 1.9
telegraf_queues: 1.9
telegraf_vcs: 1.9
telegraf_worker: 1.16
```

#### what they do

```bash
# format: conf file used, what it does/runs, instances (default is 1)
telegraf: telgraf.conf, listens for GH and papertrail webhooks, 2 instances
telegraf_queues: telegraf_queues.conf, provides tc worker pool info,
            runs queue.sh and tc-web.sh,
            queue.sh example output:
            {
              "provisionerId": "proj-autophone",
              "workerType": "gecko-t-bitbar-gw-perf-a55",
              "taskQueueId": "proj-autophone/gecko-t-bitbar-gw-perf-a55",
              "workers":73,"quarantinedWorkers":0,"pendingTasks": 0
            }
            tc-web.sh example output:
                {
                "provisionerId": "releng-hardware",
                "workerType": "win11-64-2009-hw-ref",
                "taskQueueId": "releng-hardware/win11-64-2009-hw-ref",
                "workers": 1,
                "runningWorkers": 0,
                "idleWorkers": 0,
                "quarantinedWorkers": 1,
                "pendingTasks": 0
                }
telegraf_vcs: telegraf_vcs.conf, runs:
            release_cal.sh
            google_chrome_releases.sh
            check_vcs.sh
            treestatus2.sh
telegraf_worker: telegraf_workers.conf, provides tc worker success info,
            runs queue2.sh which provides:
                exec,provisionerId=proj-autophone,workerType=gecko-t-bitbar-gw-perf-a55 idle=70,pendingTasks=0,quarantined=0,running=3,workers=73 1733350927000000000

```

#### what each config does

```bash
$ cd git/relops_infra_as_code
$ cd terraform/telegraf/docker_aws_influx/etc_v200
$ rg -A 1 commands | grep /etc/telegraf
telegraf_vcs.conf-    "/etc/telegraf/treestatus2.sh"
telegraf_vcs.conf-    "/etc/telegraf/check_vcs.sh"
telegraf_vcs.conf-    "/etc/telegraf/google_chrome_releases.sh"
telegraf_vcs.conf-    "/etc/telegraf/release_cal.sh"

telegraf_workers.conf-    "/etc/telegraf/queue2.sh releng-hardware"
telegraf_workers.conf-    "/etc/telegraf/queue2.sh aws-provisioner-v1"
telegraf_workers.conf-    "/etc/telegraf/queue2.sh gecko-t"
telegraf_workers.conf-    "/etc/telegraf/queue2.sh mobile-1 mobile-3 mobile-t"
telegraf_workers.conf-    "/etc/telegraf/queue2.sh gecko-1 gecko-b gecko-2 gecko-3"

telegraf_queues.conf-    "/etc/telegraf/queue.sh scriptworker-prov-v1"
telegraf_queues.conf-    "/etc/telegraf/queue.sh proj-servo"
telegraf_queues.conf-    "/etc/telegraf/queue.sh gce"
telegraf_queues.conf-    "/etc/telegraf/queue.sh bitbar"
telegraf_queues.conf-    "/etc/telegraf/queue.sh proj-autophone"
telegraf_queues.conf-    "/etc/telegraf/queue.sh terraform-packet"
telegraf_queues.conf-    "/etc/telegraf/queue.sh releng-hardware"
telegraf_queues.conf-    "/etc/telegraf/queue.sh gecko-t"
telegraf_queues.conf-    "/etc/telegraf/queue.sh gecko-b"
telegraf_queues.conf-    "/etc/telegraf/queue.sh gecko-1 gecko-2 gecko-3"
telegraf_queues.conf-    "/etc/telegraf/queue.sh mobile-1 mobile-3 mobile-t"
telegraf_queues.conf-    "/etc/telegraf/queue.sh bitbar proj-autophone proj-servo scriptworker-prov-v1"
telegraf_queues.conf-    "/etc/telegraf/queue.sh scriptworker-k8s"
telegraf_queues.conf-    "/etc/telegraf/queue.sh gce terraform-packet"
telegraf_queues.conf-    "/etc/telegraf/queue.sh aws-provisioner-v1"
telegraf_queues.conf-    "/etc/telegraf/tc-web.sh releng-hardware"
telegraf_queues.conf-    "/etc/telegraf/tc-web.sh aws-provisioner-v1"
telegraf_queues.conf-    "/etc/telegraf/tc-web.sh gce"
telegraf_queues.conf-    "/etc/telegraf/tc-web.sh proj-autophone terraform-packet"
telegraf_queues.conf-    "/etc/telegraf/tc-web.sh bitbar proj-servo scriptworker-prov-v1"
telegraf_queues.conf-    "/etc/telegraf/tc-web.sh gecko-t"
telegraf_queues.conf-    "/etc/telegraf/tc-web.sh gecko-1 gecko-2 gecko-3"
telegraf_queues.conf-    "/etc/telegraf/tc-web.sh gecko-b"
telegraf_queues.conf-    "/etc/telegraf/tc-web.sh mobile-1 mobile-3 mobile-t"

```

The scripts used across all docker image versions.

NOTES: Not all need to be migrated?

```bash
$ cd git/relops_infra_as_code/terraform/telegraf/docker_aws_influx
$ rg -A 1 commands | grep /etc/telegraf | cut -d ' ' -f 5 | sort | uniq | 's/\"//g'
/etc/telegraf/check_vcs.sh
/etc/telegraf/google_chrome_releases.sh
/etc/telegraf/quarantined.sh  # not run
/etc/telegraf/queue.sh
/etc/telegraf/queue2.sh
/etc/telegraf/release_cal.sh
/etc/telegraf/tc-web.sh
/etc/telegraf/treestatus.sh # not run
/etc/telegraf/treestatus2.sh

```

Theory: queue2.sh replaced tc-web.sh, but tc-web.sh was never decommissioned?
  - TODO: to confirm, dump grafana dashboards and audit their queries.

##### prom migration status

status of the migration of each of the telegraf configs to prometheus.

moved to https://mozilla-hub.atlassian.net/browse/RELOPS-1163

#### questions

- ANSWERED: what's the difference between queue2.sh and queue.sh
  - not much, slight key change (quarantinedWorkers vs quarantined)
  - queue2 is line format (vs json)
  - queue2.sh has idle and running workers
- ANSWERED: what's the difference between queue.sh and tc_web.sh?
  - tc_web.sh has idle (queue2 also has idle)
  - tc_web.sh uses GraphQL for most data (also REST API, like rest)
  - tc_web.sh seems broken (bad data produced, counts of -1, 0, or 1 for everything)
    - fixed in tc-web-prom.sh
- ANSWERED: tc-web.sh vs tc-web2.sh?
  - tc-web2 doesn't work. script in development that never got finished?
  - tc-web2 seems older strangely
  - both are graphql based
- treestatus.sh vs treestatus2.sh
  - treestatus2 is the only one used in prod.
  -

###### queue2.sh vs queue.sh

```bash
powderdry  etc git:(telegraf_work) ✗  ➜  ./queue.sh proj-autophone
[
{
  "provisionerId": "proj-autophone",
  "workerType": "gecko-t-bitbar-gw-perf-a55",
  "taskQueueId": "proj-autophone/gecko-t-bitbar-gw-perf-a55",
  "workers":75,"quarantinedWorkers":0,"pendingTasks": 82
},{
  "provisionerId": "proj-autophone",
  "workerType": "gecko-t-bitbar-gw-perf-p6",
  "taskQueueId": "proj-autophone/gecko-t-bitbar-gw-perf-p6",
  "workers":3,"quarantinedWorkers":0,"pendingTasks": 0
},{
  "provisionerId": "proj-autophone",
  "workerType": "gecko-t-bitbar-gw-perf-s24",
  "taskQueueId": "proj-autophone/gecko-t-bitbar-gw-perf-s24",
  "workers":3,"quarantinedWorkers":0,"pendingTasks": 0
},{
  "provisionerId": "proj-autophone",
  "workerType": "gecko-t-bitbar-gw-unit-p5",
  "taskQueueId": "proj-autophone/gecko-t-bitbar-gw-unit-p5",
  "workers":16,"quarantinedWorkers":0,"pendingTasks": 0
}
]
powderdry  etc git:(telegraf_work) ✗  ➜  ./queue2.sh proj-autophone
exec,provisionerId=proj-autophone,workerType=gecko-t-bitbar-gw-perf-a55 idle=11,pendingTasks=82,quarantined=0,running=64,workers=75 1733254240000000000
exec,provisionerId=proj-autophone,workerType=gecko-t-bitbar-gw-perf-p6 idle=3,pendingTasks=0,quarantined=0,running=0,workers=3 1733254241000000000
exec,provisionerId=proj-autophone,workerType=gecko-t-bitbar-gw-perf-s24 idle=3,pendingTasks=0,quarantined=0,running=0,workers=3 1733254242000000000
exec,provisionerId=proj-autophone,workerType=gecko-t-bitbar-gw-unit-p5 idle=16,pendingTasks=0,quarantined=0,running=0,workers=16 1733254243000000000

```

## reverse engineering the dockerfile

The dockerfile was missing. Recreated from the image running in AWS.

### pulling an image from ECR

see https://docs.aws.amazon.com/AmazonECR/latest/userguide/docker-pull-ecr-image.html

### login to ECR

```bash
# get these from the aws SSO portal
#
# project is:
#   relops-aws-prod
#     961225894672 | relops-aws-prod@mozilla.com
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...

 aws ecr get-login-password --region us-west-2  | docker login --username AWS --password-stdin 961225894672.dkr.ecr.us-west-2.amazonaws.com

#  docker login
```

### list ECR repos and images

```bash
aws ecr describe-repositories

# for help: `aws ecr describe-images help`

aws ecr describe-images --repository-name telegraf

aws ecr describe-images \
    --repository-name telegraf \
    --image-ids imageTag=1.7

# list tags
aws ecr describe-images --repository-name telegraf --query "sort_by(imageDetails,& imagePushedAt)[ * ].imageTags[ * ]"
```

### pulling a container

```bash
docker pull 961225894672.dkr.ecr.us-west-2.amazonaws.com/telegraf:latest

docker pull 961225894672.dkr.ecr.us-west-2.amazonaws.com/telegraf:'1.7'

# run it
docker run -it 961225894672.dkr.ecr.us-west-2.amazonaws.com/telegraf /bin/bash
```

### reverse-engineering an image to a Dockerfile

From https://stackoverflow.com/questions/19104847/how-to-generate-a-dockerfile-from-an-image.

Uses https://hub.docker.com/r/alpine/dfimage (that uses https://github.com/P3GLEG/Whaler).

```
alias dfimage="docker run -v /var/run/docker.sock:/var/run/docker.sock --rm alpine/dfimage"

# once image has been pulled, see above
dfimage -sV=1.36 961225894672.dkr.ecr.us-east-1.amazonaws.com/telegraf:latest
```

### exporting layers of a Docker image

https://github.com/micahyoung/docker-layer-extract

```bash
go install github.com/micahyoung/docker-layer-extract@latest
~/go/bin/docker-layer-extract -h

```

## reference links
- aws services
  - https://docs.aws.amazon.com/ecs/
  - https://docs.aws.amazon.com/ecr/
- prometheus
  - https://prometheus.io/docs/instrumenting/exposition_formats/#text-based-format
  - https://github.com/prometheus/docs/blob/main/content/docs/instrumenting/exposition_formats.md
  - https://github.com/influxdata/telegraf/tree/master/plugins/outputs/prometheus_client
  - https://github.com/influxdata/telegraf/blob/master/plugins/inputs/prometheus
    - used to scan other prometheus sources, not so useful in our case
- telegraf
  - https://github.com/influxdata/telegraf/blob/master/plugins/inputs/exec/README.md
