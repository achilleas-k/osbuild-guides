# The Whole Stack

## Quick start

The [`devel/`][github/image-builder-frontend/devel] directory of the [image-builder-frontend][github/image-builder-frontend] repository contains scripts, docker configs, and documentation for how to set up the whole Image Builder / OSBuild stack for local development.
You can follow the instructions in the README to get started.

This guide's purpose is to describe how all the components interact.
Using the information in this guide, you should be able to deploy, test, and troubleshoot individual components and understand how the full deployment linked above works.

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

The [`gen-certs.sh`][gen-certs.sh] script in the `tools/` directory of the osbuild-composer repository can generate all the certificates and keys that we need.
Use it with the [`openssl.cnf`][openssl.cnf] configuration file in the `test/data/` directory of the same repository to generate test certificates.

Assuming the current working directory is the root of the osbuild-composer repository:
```bash
./tools/gen-certs.sh ./test/data/openssl.cnf <devel>/config <devel>/config/ca
```
The `<devel>/config` directory will be added to each container that needs the certificates, so make note of where the certs were created.

This document describes how all the components of the entire OSBuild + Image Builder stack fit together.
This is not meant as a guide for setting up or deploying the stack, but instead a description of the requirements for such a setup.
The goal is provide an understanding of the setup and leave the specifics of any deployment (containers, virtual machines, etc) up to the reader.

### 4. The Dockerfiles

## Components

A bottom up list of the relevant components:
1. [OSBuild][guides/osbuild] ([GitHub/osbuild/osbuild][github/osbuild])
2. [OSBuild Composer + Worker][guides/osbuild-composer] ([GitHub/osbuild/osbuild-composer][github/osbuild-composer])

Interfaces/APIs
1. [Image Builder](#image-builder) [GitHub/osbuild/image-builder][github/image-buider]
   1. [Image Builder Frontend](#image-builder-frontend) [GitHub/RedHatInsights/image-builder-frontend][github/image-builder-frontend]
2. [Cockpit Composer](#cockpit-composer) [GitHub/osbuild/cockpit-composer][github/cockpit-composer]
3. [Composer CLI](#composer-cli)
4. [Koji]


[OSBuild Composer](#osbuild-composer) exposes three different APIs, each one with different subsets of the overall functionality provided by the service.
The details of each API are discussed in the section for each interface.

### OSBuild

OSBuild is the core worker of the whole stack.
It reads a *manifest* which describes a series of *pipelines*, each of which has one or more *stages*, and produces an OS artifact.
The type of artifact it produces depends on the manifest, typically defined by the stages in the final pipeline.

See the [relevant guide][guides/osbuild] for more details.

OSBuild is a command line tool.
Build jobs are started by calling `osbuild` with a JSON file as the main argument (the manifest).
A typical call to osbuild looks like the following:

```bash
osbuild --export <pipeline> --output-directory <output directory> manifest.json
```

The `--export` flag controls which *pipeline* will produce the final output.
The artifact produced by the named pipeline will be copied to the directory named by `--output-directory`.

### OSBuild Composer

OSBuild Composer is a service for managing OSBuild jobs.
One of the main tasks of composer is to define image definitions for producing manifests for OSBuild to consume.
It also maintains a queue of build jobs that have been requested, handles image uploads to public cloud

OSBuild Composer runs as two separate services:
- `osbuild-composer`: Responsible for receiving requests, generating the appropriate manifest, and maintaining the job queue.
- `osbuild-worker`: Responsible for calling `osbuild` to create an artifact and uploading it to cloud providers (e.g., AWS, MS Azure, GCP).

See the [relevant page][guides/osbuild-composer] for more details.

When `osbuild-composer` receives a request to build an image, the following sequence of actions occurs:
1. Depsolves the packages for the image
    - Depending on the API that receives the request, this is either handled by the `osbuild-worker` or `osbuild-composer` itself.
      More on this later.
    - This consists of multiple sets of packages, so multiple depsolve calls are made for each depsolve job.
2. Generates a manifest based on the requested image type, the sets of packages that were generated by the depsolve job, and the customizations requested by the user.
3. Enqueues the build job with the generated manifest.

Meanwhile, `osbuild-worker` continuously checks for jobs from `osbuild-composer` through a **worker API** (which we wont discuss in detail here).


it produces the appropriate manifest for the request and calls OSBuild with:
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
- Repositories are different depending on API:
  - Weldr API uses repositories defined in composer configurations: /usr/lib/osbuild-composer/repositories and /etc/osbuild-composer/repositories
  - Cloud API requires specifying repositories on the request
  - Image builder uses repositories defined in $DISTRIBUTIONS_DIR (/app/distributions by default)
- AWS credentials and config
  - region is defined in image builder env
  - credentials are configured in the worker
  - bucket is defined in the Koji configuration (koji.aws_config.bucket)


-->

<!-- links -->

[guides/osbuild]: osbuild.html
[github/osbuild]: https://github.com/osbuild/osbuild
[guides/osbuild-composer]: osbuild-composer.html
[github/osbuild-composer]: https://github.com/osbuild/osbuild-composer
[github/image-builder]: https://github.com/osbuild/image-builder
[github/image-builder-frontend]: https://github.com/RedHatInsights/image-builder-frontend/
[github/image-builder-frontend/devel]: https://github.com/RedHatInsights/image-builder-frontend/blob/main/devel/
[github/cockpit-composer]: https://github.com/osbuild/cockpit-composer
[gen-certs.sh]: https://github.com/osbuild/osbuild-composer/blob/main/tools/gen-certs.sh
[openssl.cnf]: https://github.com/osbuild/osbuild-composer/blob/main/test/data/x509/openssl.cnf
