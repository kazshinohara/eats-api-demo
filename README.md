# Eats API demo 

A demo application inspired by *ber Eats with the following Google Cloud Products.

- [Cloud Run](https://cloud.google.com/run)
- [Cloud SQL](https://cloud.google.com/sql)
- [Cloud Pub/Sub](https://cloud.google.com/pubsub)
- [Artifact Registry](https://cloud.google.com/artifact-registry)
- [Cloud Build](https://cloud.google.com/build)

Technologies which this demo application uses.

- [Golang](https://golang.org/)
- [Restful API](https://cloud.google.com/apis/design/resources)
- [gRPC](https://grpc.io/) (Server-side streaming)
- [MySQL](https://www.mysql.com)
- [Messaging Queue](https://en.wikipedia.org/wiki/Message_queue)
- [Container](https://github.com/opencontainers/image-spec) (a.k.a Docker)

## Architecture
### Diagram
![architecture_diagram](https://storage.googleapis.com/handson-images/eats-architecture-overview-en.png)

### Database schema
![db_schema](https://storage.googleapis.com/handson-images/eats-db-schema.png)


## How to use
### 1. Preparation

Set your preferred Google Cloud region name.
```shell
export REGION_NAME={{REGION_NAME}}
```

Set your Google Cloud Project ID
```shell
export PROJECT_ID={{PROJECT_ID}}
```

Set your Artifact Registry repository name
```shell
export REPO_NAME={{REPO_NAME}}
```

Set your Service Account name
```shell
export SA_NAME={{SERVICE_ACCOUNT_NAME}}
```

Set your DB instance name
```shell
export DB_INSTANCE_NAME={{DB_INSTANCE_NAME}}
```

Set your schema id
```shell
export SCHEMA_ID={{SCHEMA_ID}}
```

Set your Topic id
```shell
export TOPIC_ID={{TOPIC_ID}}
```

Set your Subscription id
```shell
export SUB_ID={{SUB_ID}}
```

Set your Artifact Registry repo name
```shell
export REPO_NAME={{REPO_NAME}}
```

Enable Google Cloud APIs
```shell
gcloud services enable \
  run.googleapis.com \
  sql-component.googleapis.com \
  sqladmin.googleapis.com \
  compute.googleapis.com \
  pubsub.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com
```

Set the project id into gcloud.
```shell
gcloud config set project $PROJECT_ID
```

Create a Service Account and give necessary roles to it.
```shell
gcloud iam service-accounts create ${SA_NAME}
```
```shell
gcloud projects add-iam-policy-binding --member "serviceAccount:${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" --role "roles/pubsub.publisher"
```
```shell
gcloud projects add-iam-policy-binding --member "serviceAccount:${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" --role "roles/pubsub.subscriber"
```

### 2. Create a Cloud SQL instance

Create a MySQL DB Instance.
```shell
gcloud sql instances create ${DB_INSTANCE_NAME} --tier=db-custom-1-3840 --region=${REGION_NAME}
```

Confirm if you could create it.
```shell
gcloud sql instances list
```

Change the root user's password.
```text
gcloud sql users set-password root \
--host="%" \
--instance=${DB_INSTANCE_NAME} \
--prompt-for-password
```

Connect to the DB instance with the root user.
```shell
gcloud sql connect ${DB_INSTANCE_NAME} --user=root
```

Create your database, the name must be "handson"
```sql
CREATE DATABASE handson;
```

Leave from the DB
```sql
QUIT;
```

Set DB related parameters as env vars.

```shell
export DB_USER=root
```
```shell
export DB_PWD={{DB_PASSWORD}}
```
```shell
export DB_INSTANCE=$(gcloud sql instances describe handson-db --format json | jq -r .connectionName)
```
```bash
export DB_CONNECTION="/cloudsql/"$(gcloud sql instances describe handson-db --format json | jq -r .connectionName)
```

### 3. Create Cloud Pub/Sub's schema, topic and subscription.

Create the message schema as Protocol buffer type.
```shell
gcloud beta pubsub schemas create ${SCHEMA_ID} \
--type=PROTOCOL_BUFFER \
--definition='syntax = "proto3";message ProtocolBuffer {string event_name = 1;string purchaser = 2;int64 order_id = 3;int64 item_id = 4;}'

```

Verify your schema.
```shell
gcloud beta pubsub schemas validate-message \
--message-encoding=JSON \
--message='{"event_name": "Order received", "purchaser": "Taro Yamada", "order_id": 1, "item_id": 1 }'  \
--schema-name=${SCHEMA_ID}                 
```

Create a topic with the schema.
```shell
gcloud beta pubsub topics create ${TOPIC_ID} \
--message-encoding=JSON \
--schema=${SCHEMA_ID}
```

Create a subscription.
```shell
gcloud pubsub subscriptions create ${SUB_ID} \
--topic=${TOPIC_ID}
```

### 4. Build container images
Note: please make your own [Artifact Registry repo](https://cloud.google.com/artifact-registry/docs/docker/quickstart) in advance, if you don't have it yet.

Create an Artifact Registry's repo.
```shell
gcloud artifacts repositories create ${REPO_NAME} --repository-format=docker --location=${REGION_NAME}
```
Git clone this repo to your local.
```shell
git clone git@github.com:kazshinohara/eats-api-demo.git
```

Build eats service image & Push it to Artifact Registry's repo by Cloud Build.
```shell
cd eats-api-demo/eats
```
```shell
gcloud builds submit --tag ${REGION_NAME}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/eats:v1
```

Build notification server image & Push it to Artifact Registry's repo by Cloud Build.
```shell
cd ../notification-server
```
```shell
gcloud builds submit --tag ${REGION_NAME}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/notification-server:v1
```

Build notification client image in your local.
```shell
cd ../notification-client
```
```bash
docker build -t ${REGION_NAME}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/notification-client:v1 .
```

### 5. Deploy containers to Cloud Run (fully managed)
Set Cloud Run's base configuration.
```shell
gcloud config set run/region ${REGION_NAME}
gcloud config set run/platform managed
```

Deploy Eats.
```bash
gcloud run deploy eats \
--image=${REGION_NAME}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/eats:v1 \
--allow-unauthenticated \
--set-env-vars=DB_PWD=${DB_PWD},DB_USER=${DB_USER},DB_CONNECTION=${DB_CONNECTION},PROJECT_ID=${PROJECT_ID},TOPIC_ID=${TOPIC_ID} \
--set-cloudsql-instances=${DB_INSTANCE} \
--service-account="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
```

Get Eats service's url for test access later.
```bash
export EATS_URL=$(gcloud run services describe eats --format json | jq -r '.status.address.url')
```

Deploy Notification server.
```shell
gcloud beta run deploy notification-server \
--image=${REGION_NAME}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/notification-server:v1 \
--allow-unauthenticated \
--use-http2 \
--timeout=3600 \
--set-env-vars=PROJECT_ID=${PROJECT_ID},SUB_ID=${SUB_ID} \
--service-account="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
```

Get domain name of Notification server for the client's access.
```bash
export NOTIFICATION_DOMAIN=$(gcloud run services describe notification-server --format json | jq -r '.status.address.url' | sed 's|https://||g')
```

### 6. Check behavior
Confirm if Eats service is alive.
```bash
curl -X GET ${EATS_URL}/
```

```terminal
{"version":"v1","message":"This is Eats service API"}
```

Get items list, these items were automatically created in the db when eats service starts up.
```bash
curl -X GET ${EATS_URL}/items
```

```terminal
[
    {
        "ID":1,
        "CreatedAt":"2021-06-11T16:48:44.379Z",
        "UpdatedAt":"2021-06-11T16:48:44.379Z",
        "DeletedAt":null,
        "name":"Simple Pizza",
        "price":1000,
        "currency":"JPY"
    },
    {
        "ID":2,
        "CreatedAt":"2021-06-11T16:48:44.385Z",
        "UpdatedAt":"2021-06-11T16:48:44.385Z",
        "DeletedAt":null,
        "name":"Normal Pizza",
        "price":2000,
        "currency":"JPY"
    },
    {
        "ID":3,
        "CreatedAt":"2021-06-11T16:48:44.393Z",
        "UpdatedAt":"2021-06-11T16:48:44.393Z",
        "DeletedAt":null,
        "name":"Luxury Pizza",
        "price":3000,
        "currency":"JPY"
    }
]
```
Create an order.
```shell
curl -X POST -d '{"purchaser":"Taro Yamada","item_id":1}' ${EATS_URL}/orders
```

```terminal
{
    "ID":1,
    "CreatedAt":"2021-06-13T16:34:30.286Z",
    "UpdatedAt":"2021-06-13T16:34:30.286Z",
    "DeletedAt":null,"item_id":1,
    "purchaser":"Taro Yamada",
    "item_completed":false,
    "delivery_completed":false,
    "delivery_completed_at":null
}
```

Get orders list, you could see what you created in the earlier step.
```bash
curl -X GET ${EATS_URL}/orders
```

```terminal
{
    "ID":1,
    "CreatedAt":"2021-06-13T16:34:30.286Z",
    "UpdatedAt":"2021-06-13T16:34:30.286Z",
    "DeletedAt":null,"item_id":1,
    "purchaser":"Taro Yamada",
    "item_completed":false,
    "delivery_completed":false,
    "delivery_completed_at":null
}
```

Update your order changing item_completed flag to true.
```bash
curl -X PUT -d '{"purchaser":"Taro Yamada","item_id":1,"item_completed":true}' ${EATS_URL}/orders/1
```

```terminal
{
    "ID":1,
    "CreatedAt":"2021-06-13T16:34:30.286Z",
    "UpdatedAt":"2021-06-13T17:10:53.908Z",
    "DeletedAt":null,
    "item_id":1,
    "purchaser":"Taro Yamada",
    "item_completed":true,
    "delivery_completed":false,
    "delivery_completed_at":null
}
```

Delete your order.
```bash
curl -X DELETE ${EATS_URL}/orders/1
```
```terminal
{"id":"1","message":"deleted"}
```

Run Notification client in your local.
```bash
docker run --name notification-client -e INSECURE=false -e DOMAIN=${NOTIFICATION_DOMAIN} -e PORT=443 ${REGION_NAME}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/notification-client:v1
```

Create orders, recommend you to try creating 5 ~ 6 orders to get the notification quickly.
```bash
curl -X POST -d '{"purchaser":"Taro Yamada","item_id":1}' ${EATS_URL}/orders
```

In your Notification client, you could see the following messages from Notification server.
```terminal
2021/06/12 14:40:10 Subscribing...
2021/06/12 14:40:34 {"event_name":"Order received","purchaser":"Taro Yamada","order_id":2,"item_id":1}
2021/06/12 14:40:34 {"event_name":"Order received","purchaser":"Taro Yamada","order_id":3,"item_id":2}
2021/06/12 14:40:34 {"event_name":"Order received","purchaser":"Taro Yamada","order_id":4,"item_id":3}
```

Let's try Updating your order and confirm if the notification comes.
```bash
curl -X PUT -d '{"purchaser":"Taro Yamada","item_id":1,"item_completed":true}' ${EATS_URL}/orders/2
```

In your Notification client.
```terminal
2021/06/13 17:10:47 {"event_name":"Order updated","purchaser":"Taro Yamada","order_id":2,"item_id":3}
```
