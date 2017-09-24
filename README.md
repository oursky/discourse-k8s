# Tutorial for Discourse Deployment on k8s

## Why I want to deploy discourse on k8s?

1. We have a cluster for some random tools / staging deployment already. So it is actually cheaper to deploy on the existing cluster for a internal discourse
2. My team use k8s a lot lately, although I haven't been doing any realy technical work for last few years, I still wanna learn something new.

## Why and what is this tutorial about?

* There are no complete tutorial online for deploy discourse
* This tutorial and sample config below, deploy single discourse web-server, connects to postgreSQL and redis server, and assume you're using Google Cloud Registry and gcePersistentDisk.

So here begin:

## Create Discourse App Docker Image

We will "mis-use" the launcher provided by discourse_docker to create the docker image we want for the discourse web server.

1. Clone from [https://github.com/discourse/discourse_docker](https://github.com/discourse/discourse_docker) to local environment
1. Setup temp Redis and Postgres in local environment
1. Create `containers/web_only.yml` as shown below
    1. The env var is not relevant to k8s, just for building the local image, fill in something works for your local environment
    1. Determine the plugins you want to install with your discourse setting here
1. Tips: in case you download redis, it might be in protected mode and doesn't allow docker guest to host connection, start it with `redis-server —protected-mode no`
1. Create the docker images and upload the images to your k8s docker registry. Let's say we are using Google Cloud Registry:
    1. Create docker image by discourse's launcher: `./launcher bootstrap web_only`
    1. Verify created: docker images
    1. Upload image to registry with the commands below.

```
docker tag local_discourse/web_only gcr.io/**my-cluster**/discourse:latest
gcloud docker -- push gcr.io/**my-cluster**/discourse:latest
```


web_only.yml here

```
templates:
  - "templates/web.template.yml"
  - "templates/web.ratelimited.template.yml"

env:
  LANG: en_US.UTF-8
  UNICORN_WORKERS: 2
  DISCOURSE_DB_USERNAME: chpapa
  DISCOURSE_DB_PASSWORD: ''
  DISCOURSE_DB_HOST: docker.for.mac.localhost
  DISCOURSE_DB_NAME: chpapa
  DISCOURSE_DEVELOPER_EMAILS: 'bencheng@oursky.com'
  DISCOURSE_HOSTNAME: 'localhost'
  DISCOURSE_REDIS_HOST: docker.for.mac.localhost

hooks:
  after_code:
    - exec:
        cd: $home/plugins
        cmd:
          - mkdir -p plugins
          - git clone https://github.com/discourse/docker_manager.git
          - git clone https://github.com/discourse/discourse-solved.git
          - git clone https://github.com/discourse/discourse-voting.git
          - git clone https://github.com/discourse/discourse-slack-official.git
          - git clone https://github.com/discourse/discourse-assign.git
run:
  - exec:
      cd: /var/www/discourse
      cmd:
        - sed -i 's/GlobalSetting.serve_static_assets/true/' config/environments/production.rb
        - bash -c "touch -a /shared/log/rails/{sidekiq,puma.err,puma}.log"
        - bash -c "ln -s /shared/log/rails/{sidekiq,puma.err,puma}.log log/"
        - sed -i 's/default \$scheme;/default https;/' /etc/nginx/conf.d/discourse.conf
```

## Deploy to k8s

### Prepare Persistence Volume

Assume you're using GCEPersistentDisk:

```
gcloud compute disks create --size=**10GB** --type=**pd-ssd** --zone=**us-east1-b** **discourse**
gcloud compute disks create --size=**10GB** --type=**pd-ssd** --zone=**us-east1-b** **discourse-pgsql**
```

### Prepare the secrets

Create a secret.yaml as below

```
[workspace] cat secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret
type: Opaque
data:
  dbUsername: [base64 encoded]
  dbPassword: [base64 encoded]
  smtpUsername: [base64 encoded]
  smtpPassword: [base64 encoded]

[workspace] kubectl apply -f secret.yaml
```

### Deploy to k8s

Customize the sample k8s file as follows, and the variables you probably want to tweak:

* volumes.yaml
    * For both PersistentVolume:
        * spec.capacity.storage
        * spec.gcePersistentDisk.pdName
    * The sample file here assume using gcePersistentDisk. This file need to change heavily depends on what type of persistent disk you plan to use
* redis.yaml
    * Deployment (redis)
        * spec.template.spec.containers.resources.* (CPU and Memory resources for cache server)
* pgsql.yaml
    * PersistentVolumeClaim (pgsql-pv-claim)
        * spec.resources.requests.storage (storage of DB server)
* discourse.yaml
    * PersistentVolumeClaim (discourse-pv-claim)
        * spec.resources.requests.storage (storage of web-server disk for logs and backups)
    * Deployment  (web-server)
        * spec.template.spec.containers.image (URL to your Docker image)
        * spec.template.spec.containers.env
            * DISCOURSE_DEVELOPER_EMAILS
            * DISCOURSE_HOSTNAME
            * DISCOURSE_SMTP_ADDRESS
            * DISCOURSE_SMTP_PORT
        * spec.template.spec.containers.resources.* (CPU and Memory resources for your web-server)
* ingress.yaml 
    * spec.rules.host
    * spec.tls.hosts

(Optional) Set the k8s context first to avoid --namespace in all commands below

```
kubectl config set-context $(kubectl config current-context) --namespace=**discourse-namespace**
```

Apply all config files

```
kubectl apply -f volumes.yaml
kubectl apply -f redis.yaml
kubectl apply -f pgsql
```

Before starting the app, run the following on pgsql instance to init the database properly (TODO, create a pgsql image for that). You can find your pod by running `kubectl get pods`

```
kubectl exec **pgsql-2981420674-hxx6h** -- su postgres -c '/opt/bitnami/postgresql/bin/psql template1 -c "create extension if not exists hstore;"'
kubectl exec **pgsql-2981420674-hxx6h** -- su postgres -c '/opt/bitnami/postgresql/bin/psql template1 -c "create extension if not exists pg_trgm;"'
kubectl exec **pgsql-2981420674-hxx6h** -- su postgres -c '/opt/bitnami/postgresql/bin/psql discourse -c "create extension if not exists hstore;"'
kubectl exec **pgsql-2981420674-hxx6h** -- su postgres -c '/opt/bitnami/postgresql/bin/psql **discourse** -c "create extension if not exists pg_trgm;"'
kubectl exec **pgsql-2981420674-hxx6h** -- su postgres -c "/opt/bitnami/postgresql/bin/psql **discourse** -c \"update pg_database set encoding = pg_char_to_encoding('UTF8') where datname = '**discourse**' AND encoding = pg_char_to_encoding('SQL_ASCII');\""
```

Create discourse deployment and ingress

```
kubectl apply -f discourse.yaml
kubectl apply -f ingress.yaml
```

From there, your discourse instance should be up and running, some useful command below in case things don't work and require your debugging:

```
# Check logs of k8s pod
kubectl logs —since=1h —tail=50 -lapp=web-server

# Open an interactive terminal into the web-server:
kubectl exec -it **web-server** — /bin/bash

# You might want to do the following:
# * Check logs under /shared/log/rails
# * Run bundle exec rake db:migrate / admin:create in case something went wrong.
# * You can also check the logs with an admin account at /admin/logs of your deployment
```

## Setup S3 Backup and Files Upload

Discourse can use S3 for backup and files upload, here are the steps to enable it:

1. Create two S3 bucket, one for backup, one for files upload. Set it as private.
2. Create an IAM user with API access only, attach the AWS inline policy below.
3. Fill in the Access Key and Key ID in Discourse Setting.

AWS inline policy for S3 bucket access

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Stmt1506240388000",
            "Effect": "Allow",
            "Action": [
                "s3:*"
            ],
            "Resource": [
                "arn:aws:s3:::**discourse-upload**",
                "arn:aws:s3:::**discourse-upload**/*"
            ]
        },
        {
            "Sid": "Stmt1506240479000",
            "Effect": "Allow",
            "Action": [
                "s3:*"
            ],
            "Resource": [
                "arn:aws:s3:::**discourse-backup**",
                "arn:aws:s3:::**discourse-backup**/*"
            ]
        }
    ]
}
```

## Potential improvement:

* Run multiple replicas for Discourse web-server for scale up. It should works actually but just haven't tried.
* Deploy Master-Slave PostgreSQL for scale up. We're using bitnami's postgreSQL docker image and the relevant instructions are here: https://github.com/bitnami/bitnami-docker-postgresql

