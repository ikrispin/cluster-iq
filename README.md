# Cluster IQ

[![Go Report Card](https://goreportcard.com/badge/github.com/RHEcosystemAppEng/cluster-iq)](https://goreportcard.com/report/github.com/RHEcosystemAppEng/cluster-iq)
[![Go Reference](https://pkg.go.dev/badge/github.com/RHEcosystemAppEng/cluster-iq.svg)](https://pkg.go.dev/github.com/RHEcosystemAppEng/cluster-iq)

Cluster IQ is a tool for making stock of the Openshift Clusters and its
resources running on the most common cloud providers and collects relevant
information about the compute resources, access routes and billing.

Metrics and monitoring is not part of the scope of this project, the main
purpose is to maintain and updated inventory of the clusters and offer a easier
way to identify, manage, and estimate costs.

## Supported cloud providers

The scope of the project is to cover make stock on the most common public cloud
providers, but as the component dedicated to scrape data is decoupled, more
providers could be included in the future.

The following table shows the compatibility matrix and which features are
available for every cloud provider:

| Cloud Provider | Compute Resources | Billing | Managing |
|----------------|-------------------|---------|----------|
| AWS            | Yes               | Yes     | No       |
| Azure          | No                | No      | No       |
| GCP            | No                | No      | No       |


## Architecture

The following graph shows the architecture of this project:
![ClusterIQ architecture diagram](./doc/arch.png)


## Installation
This section explains how to deploy ClusterIQ and ClusterIQ Console.


### Prerequisites:
#### Accounts Configuration
1. Create a folder called `secrets` for saving the cloud credentials. This folder is ignored on this repo to keep your
   credentials safe.
    ```text
    mkdir secrets
    export CLUSTER_IQ_CREDENTIALS_FILE="./secrets/credentials"
    ```
    :warning: Please take care and don't include them on the repo.

2. Create your credentials file with the AWS credentials of the accounts you
   want to scrape. The file must follow the following format:
    ```text
    echo "
    [ACCOUNT_NAME]
    provider = {aws/gcp/azure}
    user = XXXXXXX
    key = YYYYYYY
    billing_enabled = {true/false}
    " >> $CLUSTER_IQ_CREDENTIALS_FILE
    ```
    :warning: The values for `provider` are: `aws`, `gcp` and `azure`, but the
    scraping is only supported for `aws` by the moment.  The credentials file
    should be placed on the path `secrets/*` to work with
    `docker/podman-compose`.

    :exclamation: This file structure was design to be generic, but it works
    differently depending on the cloud provider. For AWS, `user` refers to the
    `ACCESS_KEY`, and `key` refers to `SECRET_ACCESS_KEY`.

    :exclamation: Some Cloud Providers has extra costs when querying the Billing
    APIs (like AWS Cost Explorer). Be careful when enable this module. Check your
    account before enabling it.

### Openshift Deployment
1. Prepare your cluster and CLI
    ```sh
    oc login ...

    export NAMESPACE="cluster-iq"
    oc new-project $NAMESPACE
    ```

2. Create a secret containing this information is needed. To create the secret,
   use the following command:
    ```shell
    oc create secret generic credentials -n $NAMESPACE \
      --from-file=credentials=$CLUSTER_IQ_CREDENTIALS_FILE
    ```

3. Configure your cluster-iq deployment using
   `./deployments/openshift/00_config.yaml` file. For more information about the
   supported parameters, check the [Configuration Section](#configuration).
    ```sh
    oc apply -n $NAMESPACE -f ./deployments/openshift/00_config.yaml
    ```

4. Create the Service Account for Cluster-IQ, and bind it with the `anyuid` SCC.
    ```sh
    oc apply -n $NAMESPACE -f ./deployments/openshift/01_service_account.yaml
    oc adm policy add-scc-to-user anyuid -z cluster-iq
    ```

5. Deploy and configure the Database:
    ```sh
    oc create configmap -n $NAMESPACE pgsql-init --from-file=init.sql=./db/sql/init.sql
    oc apply -n $NAMESPACE -f ./deployments/openshift/02_database.yaml
    ```

6. Deploy API:
    ```sh
    oc apply -n $NAMESPACE -f ./deployments/openshift/03_api.yaml
    ```

7. Reconfigure ConfigMap with API's route hostname.
    ```sh
    ROUTE_HOSTNAME=$(oc get route api -o jsonpath='{.spec.host}')
    oc get cm config -o yaml | sed 's/REACT_APP_CIQ_API_URL: .*/REACT_APP_CIQ_API_URL: https:\/\/'$ROUTE_HOSTNAME'\/api\/v1/
    ```

7. Deploy Scanner:
    ```sh
    oc apply -n $NAMESPACE -f ./deployments/openshift/04_scanner.yaml
    ```

8. Deploy Console:
    ```sh
    oc apply -n $NAMESPACE -f ./deployments/openshift/05_console.yaml
    ```


## Local Deployment (for development)
For deploying ClusterIQ in local for development purposes, check the following
[document](./doc/development-setup.md)



### Configuration
Available configuration via Env Vars:
| Key                  | Value                         | Description                               |
|----------------------|-------------------------------|-------------------------------------------|
| CIQ_API_HOST         | string (Default: "127.0.0.1") | Inventory API listen host                 |
| CIQ_API_PORT         | string (Default: "6379")      | Inventory API listen port                 |
| CIQ_API_PUBLIC_HOST  | string (Default: "")          | Inventory API public endpoint             |
| CIQ_DB_HOST          | string (Default: "127.0.0.1") | Inventory database listen host            |
| CIQ_DB_PORT          | string (Default: "6379")      | Inventory database listen port            |
| CIQ_DB_PASS          | string (Default: "")          | Inventory database password               |
| CIQ_CREDS_FILE       | string (Default: "")          | Cloud providers accounts credentials file |

These variables are defined in `./<PROJECT_FOLDER>/.env` to be used on Makefile
and on `./<PROJECT_FOLDER>/deploy/openshift/config.yaml` to deploy it on Openshift.


### Scanners
[![Docker Repository on Quay](https://quay.io/repository/ecosystem-appeng/cluster-iq-scanner/status "Docker Repository on Quay")](https://quay.io/repository/ecosystem-appeng/cluster-iq-aws-scanner)

As each cloud provider has a different API and because of this, a specific
scanner adapted to the provider is required.

To build every available scanner, use the following makefile rules:

```shell
make build-scanners
```

By default, every build rule will be performed using the Dockerfile for each
specific scanner

#### AWS Scanner
The scanner should run periodically to keep the inventory up to date.

```shell
# Building
make build-aws-scanner
```



## API Server
[![Docker Repository on Quay](https://quay.io/repository/ecosystem-appeng/cluster-iq-api/status "Docker Repository on Quay")](https://quay.io/repository/ecosystem-appeng/cluster-iq-api)

The API server interacts between the UI and the DB.

```shell
# Building
make build-api

# Run
make start-api
```
