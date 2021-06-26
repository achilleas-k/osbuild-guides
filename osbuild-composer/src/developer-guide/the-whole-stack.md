# The Whole Stack

This document describes how all the components of the entire OSBuild + Image Builder stack fit together.
It is not a guide for setting up or deploying the stack per se, but instead a description of the requirements for such a setup.
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
osbuild --export assembler --output-directory osbuild-output manifest.json
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
