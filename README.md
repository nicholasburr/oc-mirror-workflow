# `oc mirror` and Quay Enterprise workflow guide for Disconnected Openshift4 Clusters

## Introduction
  This is an oppinionated guide for  guide assumes an `ImageSetConfiguration` has already been generated and populated with content by an Administator or other 3rd party. 

  As of the writing of this document, `oc-mirror` is a Red Hat supported content mirroring plugin for the opencshift cli.  `oc mirror` can be used to sync Openshift Releases, Operators, Helm chats, and additional images to a container regestry using a configuartion file, and using a backend storage mechanism allows `oc mirror` to manage the content in a declaritive manor.


## Documentation 

[Opensource Project](https://github.com/openshift/oc-mirror)

[Red Hat Downloads:](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/) locate the latest release, and download `oc-mirror.ta.gz`

[Installation](https://docs.openshift.com/container-platform/4.12/installing/disconnected_install/installing-mirroring-disconnected.html#installation-oc-mirror-installing-plugin_installing-mirroring-disconnected)

[Mirroring for Disconnected Enviornments](https://docs.openshift.com/container-platform/4.12/installing/disconnected_install/installing-mirroring-disconnected.html)

## Prerequisits
1. A Git Repository for the `ImageSetConfiguration` files used by `oc mirror`.
2. A Quay Enterprise repositories for `oc mirror` to store the backend configuration (metadata).
3. A host or workstation with at least 5gb of `/tmp` space.
4. The `oc-mirror` plugin installed on your workstation enviornment using the Installation guide in the Documentation section.  Additionally, verify that you have [configured credentials](https://docs.openshift.com/container-platform/4.12/installing/disconnected_install/installing-mirroring-disconnected.html#installation-adding-registry-pull-secret_installing-mirroring-disconnected) to communicate with Quay Enterprise and the Red Hat Registry.

## Configuring an `ImageSetConfiguration`
  `oc mirror` works by defining an `ImageSetConfiguration` which it will use to declare image sets needs to be pulled from an external source, and mirrored to, or pruned from the existing mirrored content in Qauy Enterprise.  An `ImageSetConfiguration` can mirror 4 types of images; `platform`, `operators`, `additionalImages`, and `helm` charts. Each of these image types has thier own [configuration parameters](https://docs.openshift.com/container-platform/4.12/installing/disconnected_install/installing-mirroring-disconnected.html#oc-mirror-imageset-config-params_installing-mirroring-disconnected).

  In the provided example we will mirror the Openshift `4.12.0` and `4.12.1` releases from the `fast-4.12` channel, for the `amd64` architechure and the backend will be in the `quay-enterpise.example.com/my_organization/release-metadata` repository, with the `latest` tag. 

```
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2                                                    
storageConfig:                                                      
  registry:
    imageURL: quay-enterpise.example.com/my_organization/release-metadata:latest                 
    skipTLS: false
mirror:
  platform:
    graph: true                                                     
    architectures:
      - amd64
    channels:
      - name: fast-4.12                                            
        minVerison: 4.12.0
        maxVersion: 4.12.1
        shortestPath: true
        type: ocp
```

## Mirroring to Quay Enterprise
Heading to the command line where `oc mirror` is installed, we initalize the mirror by specifying our configuration file, and the destination organization in Quay Enterprise.
```
oc mirror --config /path/to/my/ImageSetConfigutation.yaml docker://quay-enterpise.example.com/my_organization
```
A successful mirror can take only a few minutes, to multiple hours, and once complete will write important files for your Openshift Deployment to the local disk.

```
...
info: Mirroring completed in 21m32.61s (19.90MB/s)
Writing image mapping to oc-mirror-workspace/results-1675877766/mapping.txt
Writing UpdateService manifests to oc-mirror-workspace/results-1675877766
Writing ICSP manifests to oc-mirror-workspace/results-1675877766

[user@hostname ~]$ tree oc-mirror-workspace/results-1675877766
oc-mirror-workspace/results-1675877766
├── charts
├── imageContentSourcePolicy.yaml
├── mapping.txt
├── release-signatures
│   ├── signature-sha256-abcd123.json
│   ├── signature-sha256-efgh456.json
│   └── signature-sha256-ijkl7890.json
└── updateService.yaml

2 directories, 6 files
```

## `oc mirror` generated content
After successfully mirroring, `oc mirror` will write specific Openshift configuration files to the local disk.  Each *type* of `mirror` defined  Below is a list of the key files written to the local disk in the `oc-mirror-workspace/results-*` directory.

- **platform:**
  - imageContentSourcePolicy.yaml
  - release-signatures/signature-sha256-xyz
  - updateService.yaml
- **operators**
  - imageContentSourcePolicy.yaml
  - catalogSource-redhat-operator-index.yaml
    - Note: When syncing multiple Operator catalogs, there will be multiple `catalogSource-redhat-operator-index.yaml` files.
- **additonalImages**
  - imageContentSourcePolicy.yaml
- **helm**
  - TBD

## Managing `oc mirror` configuration files

Because `oc mirror` generates Openshift configuration files, I recommend breaking up the  `ImageSetConfigutation` into multiple files for organizations that are trying to manage multiple Disconnected, or Content Restricted Openshift Clusters.



