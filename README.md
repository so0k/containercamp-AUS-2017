# Distributed Command Execution using Containers and Cog

Overview of Operable Cog (a ChatOps bot) and how it uses containers and Docker hosts to execute command across a distributed set of servers. A 30 minute talk about ChatOps and the power of linux command pipelines when leveraging technologies such as Docker. I hope to ignite an interest to attract new FaaS oriented frameworks on integrating with the user interface provided by Cog.

[slides](https://docs.google.com/presentation/d/1qw1a8o8FnALbE7kBS_JiZoQh0H3Fs-mJBJqmWOvLQtc/edit?usp=sharing)

## Demo Script

*Note*: to automatically have route53 records created,
I should install the `route53-mapper` service and annotate my pods...

Let's do it manually using Cog r53 and kubectl bundle:

Inspect kubectl bundle
```
!help kubectl
```

Inspect kubectl run command
```
!help kubectl:run
```

Create Inspector deployment
```
!kubectl:run inspector --image=so0k/kuar-inspector:1.0.0 --port=80
```

Review deployment status:
```
!kubectl:get deploy inspector
```

Expose inspector through a service of type loadBalancer
```
!kubectl:expose deploy inspector --type=LoadBalancer
```
Notice the ExternalIP is pending

Revisit the service status
```
!kubectl:get svc inspector
```

Manually create a CNAME record for this load balancer
```
!r53:record-create -z Z37AJ8CHRUEZC6 -t CNAME inspector.spawnhub.com <elb-here>
```

Verify the DNS records have been updated
```
!dig --short inspector.spawnhub.com
```

Scale out the number of inspector pods
```
!kubectl:scale deploy inspector --replicas=3
```

Watch the round robin over the Pod ips
```
while true;do curl -s http://inspector.spawnhub.com/net | egrep -o '100\.[0-9]+\.[0-9]+\.[0-9]+' | head -n 1; sleep .5;done
```

Review the deployment after scaling up
```
!kubectl:get deploy inspector
```

Watch the version
```
while true;do curl -s http://inspector.spawnhub.com | egrep -o 'Version: Inspector [0-9].[0-9].[0-9]'; sleep .5;done
```

Start a rolling update
```
!kubectl:set image deploy inspector inspector=so0k/kuar-inspector:2.0.0
```


A proper pipeline
```
!instance-search 'tag:KubernetesCluster=staging.spawnhub.com' | jq '.public_ip_address | values | "ssh://admin@\(.)"'
```

*Note*:

- Seems there is a bug creating Alias records with r53, using CNAME
- Piping into kubectl:describe doesn't seem to work

## Preparations:

### Kubernetes clusters on AWS

Create an AWS account and get a domain name
follow the excelent [kops on aws](https://github.com/kubernetes/kops/blob/master/docs/aws.md) guides

(for Sydney use `ap-southeast-2`)

easy DNS validation with [Gsuite dig](https://toolbox.googleapps.com/apps/dig/#NS/)

```
export CLUSTER=staging
export DOMAIN=spawnhub
export NAME=${CLUSTER}.${DOMAIN}.com
export KOPS_STATE_STORE=s3://kubernetes-${DOMAIN}-com-state-store
export AWS_REGION=ap-southeast-2
export KEY_NAME=k8s-${CLUSTER}-${DOMAIN}

aws s3api create-bucket --bucket kubernetes-${DOMAIN}-com-state-store --region ${AWS_REGION}
aws s3api put-bucket-versioning --bucket kubernetes-${DOMAIN}-com-state-store  --versioning-configuration Status=Enabled


# aws ec2 describe-availability-zones --region ap-southeast-2

mkdir -p ~/.ssh/
aws --region ${AWS_REGION} ec2 create-key-pair --key-name ${KEY_NAME} --query 'KeyMaterial' --output text > ~/.ssh/${KEY_NAME}.pem
chmod 400 ~/.ssh/${KEY_NAME}.pem
ssh-add ~/.ssh/${KEY_NAME}.pem
ssh-keygen -y -f ~/.ssh/${KEY_NAME}.pem > ~/.ssh/${KEY_NAME}.pub


kops create cluster \
    --zones ap-southeast-2a \
    --ssh-public-key ~/.ssh/${KEY_NAME}.pub \
    ${NAME}

kops update cluster ${NAME} --yes # to actually create the cluster

# takes max 5 min..

kops validate cluster ${NAME}

# get dashboard - https://github.com/kubernetes/dashboard/releases
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.6.1/src/deploy/kubernetes-dashboard.yaml

# alternatively - add DNS mapper? (not applied for this talk)
https://github.com/kubernetes/kops/blob/75718857b69756ac8374a1001363eadfa6f09a4f/addons/route53-mapper/README.md

# Ideally use something like kube-lego to use https with cog...
```

### Installing Cog

Overview:

- Installing into Staging k8s cluster
- Using postgres instance managed by RDS (could also just helm install postgres in k8s, but going with RDS for now).
  note: need to add a subnet to the k8s VPC to provision RDS (kops doesn't have multi subnets for the default cluster config)
- Using the excelent cog helm chart from [ohaiwalt](https://github.com/ohaiwalt/cog-helm/)

Once Postgres is provisioned, Follow [these instructions to set up postgres](https://gist.github.com/so0k/f4308160a9a2e749aa0b90715288e08b)

(Ensure RDS security group allows ingress from the k8s cluster)
```
$ kubectl exec -it alpine-shell --image=alpine:3.5 -- sh
> apk update
> apk add postgres-client
> psql -U $RDS_USERNAME -h $RDS_HOSTNAME -p $RDS_PORT -d postgres
postgres=> \l
postgres=> CREATE DATABASE cog;
postgres=> CREATE USER cog WITH PASSWORD 'mysupersecret';
postgres=> GRANT ALL PRIVILEGES ON DATABASE cog TO cog;
postgres=> \connect cog
cog=> CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
cog=> \q
```

Install Tiller to cluster
```
helm init
```

[Get slack token](https://my.slack.com/services/new/bot)

Get relay-id: using `uuid` on OSX:
```
uuidgen | tr '[:upper:]' '[:lower:]'
```

Get relayCogToken (using python)
```
python -c 'import sys,os,binascii; sys.stdout.write(binascii.hexlify(os.urandom(16)).decode("utf-8"))'
```

Review `cog/.secrets.yaml`:
```
cog:
  config:
    cog-api-url-base: http://cog.spawnhub.com
    ## Cog usually represents these endpoints as separate ports
    ## Configuring the url-base allows them on the same domain
    cog-service-url-base: http://cog.spawnhub.com/service
    cog-trigger-url-base: http://cog.spawnhub.com/trigger

  secrets:
    databaseURL: ecto://cog:mysupersecret@spawnhub-db.clhwa0v5ueuv.ap-southeast-2.rds.amazonaws.com:5432/cogdb
    slackAPIToken: xoxb-...
    cogBootstrapPassword: mybootstrappassword
relay:
  config:
    relay-id: ...
  secrets:
    relayCogToken: ...
```


Once Tiller is ready:
```
helm install --name ccamp -f .secrets.yaml pod
```

You should see Cog connect to your slack channel... say hello to auto-create an account..


Get a shell into the cog pod and add yourself to the admin group
```
kubectl exec -it ccamp-cog-2654845993-2h2b5 -- bash
cogctl group add cog-admin <your-slack-handle>
```

Going forward, you will be able to install bundles from slack:

Installing and enabling the ec2,r53,s3 and jq bundles
```
@cog bundle install ec2
@cog bundle enable ec2

@cog bundle install s3
@cog bundle enable s3

@cog bundle install r53
@cog bundle enable r53

@cog bundle install jq
@cog bundle enable jq
@cog relay-group member assign default ec2 s3 r53 jq
```

similar, repeat for each bundle:
```
@cog permission grant ec2:read cog-admin
@cog permission grant ec2:write cog-admin
@cog permission grant ec2:admin cog-admin
```

You still need to provide the AWS credentials.

For Demo purposes, give cog :
```
aws iam create-group --group-name cog

export arns="
arn:aws:iam::aws:policy/AmazonEC2FullAccess
arn:aws:iam::aws:policy/AmazonRoute53FullAccess
arn:aws:iam::aws:policy/AmazonS3FullAccess"

for arn in $arns; do aws iam attach-group-policy --policy-arn "$arn" --group-name cog; done

aws iam create-user --user-name cog

aws iam add-user-to-group --user-name cog --group-name cog

aws iam create-access-key --user-name cog
```

Create config, upload and apply to all bundles:
```
echo 'AWS_ACCESS_KEY_ID: "AKI.."' >> config.yaml
echo 'AWS_SECRET_ACCESS_KEY: "ncO..."' >> config.yaml
echo 'AWS_REGION: "ap-southeast-2"' >> config.yaml
kubectl cp ./config.yaml ccamp-cog-2654845993-2h2b5:/home/operable/
kubectl exec -it ccamp-cog-2654845993-2h2b5 -- bash

cogctl bundle config create ec2 ~/config.yaml
cogctl bundle config create s3 ~/cog-config.yaml
cogctl bundle config create r53 ~/cog-config.yaml
```

Create k8s credentials
```
kubectl create sa cog
c=`kubectl config current-context`
secret=$(kubectl get sa cog -o json | jq -r .secrets[].name)
KUBERNETES_SERVER=`kubectl config view -o jsonpath="{.clusters[?(@.name == \"$name\")].cluster.server}"`
KUBERNETES_TOKEN=$(kubectl get sa cog -o json | jq -r .secrets[0].name | xargs kubectl get secret -o json | jq -r .data.token | base64 -D)
KUBERNETES_CERT=$(kubectl get sa cog -o json | jq -r .secrets[0].name | xargs kubectl get secret -o json | jq -r .data[\"ca.crt\"])


echo "\"KUBERNETES_TOKEN\": ${KUBERNETES_TOKEN}" >> k8s-staging-config.yaml
echo "\"KUBERNETES_SERVER\": ${KUBERNETES_SERVER}" >> k8s-staging-config.yaml
echo "\"KUBERNETES_CERT\": ${KUBERNETES_CERT}" >> k8s-staging-config.yaml
```

Install kubectl bundle
```
@cog bundle install kubectl
@cog bundle enable kubectl
@cog bundle relay-group member assign default kubectl

@cog permission grant kubectl:read cog-admin
@cog permission grant kubectl:write cog-admin
@cog permission grant kubectl:admin cog-admin
```

Configure kubectl bundle
```
kubectl cp ./k8s-staging-config.yaml ccamp-cog-2654845993-2h2b5:/home/operable/
kubectl exec -it ccamp-cog-2654845993-2h2b5 -- bash
cogctl bundle config create kubectl ~/k8s-staging-config.yaml --layer=room/cluster-staging
```

