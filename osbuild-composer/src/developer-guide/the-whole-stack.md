# The Whole Stack

## Setting up from scratch

### 1. Clone repositories

### 2. Configuration

#### osbuild-composer

`/etc/osbuild-composer/osbuild-composer.toml`
```toml
[koji]
ca = "/etc/osbuild-composer/ca-crt.pem"
enable_tls = true

[worker]
allowed_domains = ["worker", "172.31.0.20"]
ca = "/etc/osbuild-composer/ca-crt.pem"
enable_tls = true
```

*Note: The `koji` section of the config file is used for general composer configuration as well.*

The `worker.allowed_domains` option controls which domains are allowed to connect to the osbuild-composer service and take on a job as a worker.
In this case, the hostname and IP address of the worker service inside the container network are defined.

#### osbuild-worker

`/etc/osbuild-worker/osbuild-worker.toml`
```toml
[aws]
credentials = "/etc/osbuild-worker/aws-credentials"
[gcp]
credentials = "/etc/osbuild-worker/gcp-credentials"
[azure]
credentials = "/etc/osbuild-worker/azure-credentials"
```

The worker requires cloud service credentials for uploading images and other artifacts to the public cloud.
The files should have the following content:

`aws-credentials` has the same content as the aws-cli credentials configuration file.
If it's not specified, the worker will try to load it from the default location `$HOME/.aws/credentials`
```
[default]
aws_acces_key_id = "<ID>"
aws_secret_access_key = "<SECRET KEY>"
```

`gcp-credentials`
*TODO*

`azure-credentials`
```
client_id     = "<ID>"
client_secret = "<SECRET>"
```

#### image-builder

##### Environment

The following values are set in the `docker-compose.yml` file for the `backend` service.
The URLs and paths are all set in order to work within the container network.

General configuration settings for the service.
- `LISTEN_ADDRESS=backend:8086` address and port that the image-builder backend will listen on.
- `LOG_LEVEL=DEBUG`
- `ALLOWED_ORG_IDS=*` controls which org IDs can send requests to the Image Builder API.  `*` means any ID is allowed.
- `DISTRIBUTIONS_DIR=/app/distributions` is a directory that contains package sources (repositories) and package lists for each repository.  Production versions of these files exist in the [`image-builder` source][github/image-builder] in the `distributions/` directory.  They will be copied into the container during the build.

Database configuration.  This depends on the configuration for the database service, named `postgres`, in the same file.
- `PGHOST=postgres`
- `PGPORT=5432`
- `PGDATABASE=postgres`
- `PGUSER=postgres`
- `PGPASSWORD=postgres`

OSBuild configuration.  This depends on the configuration for the osbuild-composer service, named `composer`, in the same file.
- `OSBUILD_URL=https://composer:8080`
- `COMPOSER_TOKEN_URL=https://composer:8080`
- `COMPOSER_OFFLINE_TOKEN=token`

Cloud secrets and configuration are required for uploading images and artifacts to public cloud services.
The real values are set in the `.env` file
- `OSBUILD_AWS_REGION=$OSBUILD_AWS_REGION`
- `OSBUILD_AWS_ACCESS_KEY_ID=$OSBUILD_AWS_ACCESS_KEY_ID`
- `OSBUILD_AWS_SECRET_ACCESS_KEY=$OSBUILD_AWS_SECRET_ACCESS_KEY`
- `OSBUILD_AWS_S3_BUCKET=$OSBUILD_AWS_S3_BUCKET`
- `OSBUILD_GCP_REGION=$OSBUILD_GCP_REGION`
- `OSBUILD_GCP_BUCKET=$OSBULID_GCP_BUCKET`
- `OSBUILD_AZURE_LOCATION=$OSBUILD_AZURE_LOCATION`

The certificates and keys are generated before starting the containers and are shared between all containers for convenience.
- `OSBUILD_CERT_PATH=/etc/osbuild-composer/client-crt.pem`
- `OSBUILD_KEY_PATH=/etc/osbuild-composer/client-key.pem`
- `OSBUILD_CA_PATH=/etc/osbuild-composer/ca-crt.pem`
- `COMPOSER_CA_PATH=/etc/osbuild-composer/`



### 3. Generate certificates





This document describes how all the components of the entire OSBuild + Image Builder stack fit together.
This is not meant as a guide for setting up or deploying the stack, but instead a description of the requirements for such a setup.
The goal is provide an understanding of the setup and leave the specifics of any deployment (containers, virtual machines, etc) up to the reader.

## Components

A bottom up list of the relevant components:
1. [OSBuild][guides/osbuild] ([GitHub/osbuild/osbuild][github/osbuild])
2. [OSBuild Composer + Worker][guides/osbuild-composer] ([GitHub/osbuild/osbuild-composer][github/osbuild-composer])

Interfaces/APIs
1. [Image Builder](#image-builder) [GitHub/osbuild/image-builder](https://github.com/osbuild/image-builder)
   1. [Image Builder Frontend](#image-builder-frontend) [GitHub/RedHatInsights/image-builder-frontend](https://github.com/RedHatInsights/image-builder-frontend/issues)
2. [Cockpit Composer](#cockpit-composer) [GitHub/osbuild/cockpit-composer](https://github.com/osbuild/cockpit-composer)
3. [Composer CLI](#composer-cli)
4. [Koji]


[OSBuild Composer](#osbuild-composer) exposes three different APIs, each one with different subsets of the overall functionality provided by the service.
The details of each API are discussed in the section for each interface.

### OSBulid

OSBuild is the core worker of the whole stack.
It reads a *manifest* which describes a series of *pipelines*, each of which has one or more *stages*, and produces an OS artifact.
The type of artifact it produces depends on the manifest, typically defined by the stages in the final pipeline.

See the [relevant page][guides/osbuild] for more details.

OSBuild is a command line tool.
Build jobs are started by calling `osbuild` with JSON manifest file as the main argument.
A typical call to osbuild looks like the following:

```bash
osbuild --export <finalstage> --output-directory osbuild-output manifest.json
```

TODO: Describe the other flags


### OSBuild Composer

OSBuild Composer is a service for managing OSBuild jobs.
One of the main tasks of composer is to define image definitions for producing manifests for OSBuild to consume.


See the [relevant page][guides/osbuild-composer] for more details.

When OSBuild Composer receives a request to build an image, it produces the appropriate manifest for the request and calls OSBuild with:
```bash
osbuild --export <export> --output-directory <outputdir> --store <storedir> --json -
```
where:
- `<export>` is the name of a pipeline the result of which will be exported as the final artifact,
- `<outputdir>` is the root directory in which the artifact will be exported (with a subdirectory named after the export pipeline),
- `<storedir>` is a cache directory where intermediate stages and cached objects are stored, and
- `--json` defines the log and status output format.

The manifest is piped to OSBuild through `stdin`.

-----

#### OSBuild Composer APIs

OSBuild Composer supports requests through three APIs:
1. Weldr: the weldr API is based on 

<!--

## NOTES

- Image Builder database: Requests to image builder are assigned to a user ID.  Users can only see their own org's composes.
- Composer stores output of osbuild alongside manifest, success flag, and start and end timestamps.

-->

<!-- links -->

[guides/osbuild]: osbuild.html
[github/osbuild]: https://github.com/osbuild/osbuild
[guides/osbuild-composer]: osbuild-composer.html
[github/osbuild-composer]: https://github.com/osbuild/osbuild-composer
[github/image-builder]: https://github.com/osbuild/image-builder
