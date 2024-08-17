# GSoC 2024 Final Report

## Overview

Organization: KubeVirt
Project: Rewrite KubeVirtCI into Go. [Issue](https://github.com/kubevirt/community/issues/257)
Contributor: Ammar Yasser
Mentors: 
- Luboslav Pivarc. [github](https://github.com/xpivarc)
- Antonio Cardace. [github](https://github.com/acardace)

## Introduction

KubevirtCI is a codebase used to spin up, build and provision the container images on which Kubevirt spins up virtual machines, And sits at the heart of the entire CI process of Kubevirt. Testing new Kubevirt code involves using this project in creating the base cluster that comes with all the dependencies that Kubevirt will need pre-installed.
The project has several parts:

### gocli

A small CLI written in golang using the Cobra framework, It's the core engine for spinning up k8s clusters and provisioning kubevirt providers, it is used to run containers that contain inside them scripts that create qemu virtual machines from qcow2 disk images, and then connects to those virtual machines to install optional configurations or create kubernetes deployments of Istio, prometheus, rook.. etc.

### cluster-provision

A collection of bash scripts used to create the container used by the gocli and create a proper disk image in it, these scripts are called to install all the necessary dependencies on the VM and then commit the end container that contains the modified disk image, they leverage the gocli and provide additional wrapper functionality.

### cluster-up

A collection of bash scripts used to create a KubevirtCI cluster, it collects the required parameters and uses them in calling the gocli for cluster creation. This is also the part that gets shipped to Kubevirt proper, to be consumed during testing

## Problem statement

The project contains a lot of duplicated code, the cluster-up and cluster-provision act based on directories, where each valid cluster provider has its own directory in the project, the providers are different versions of kubernetes, 1.28, 1.29.. etc. as well as multiple flavors of kind. kind normal cluster, kind with SR-IOV.. etc.

Each directory for a kubernetes version provider contains essentially the same bash scripts. adding a new kubernetes version is done by creating a new folder, e.g: 1.31, and copying the same scripts to it. this structure has multiple side effects:

- It is tedious to introduce a change to any of these scripts, as you have to update the same script in each directory
- The code base is bloated more than it needs to be, as it contains the same code multiple times
- Adding a new version involves a lot of work, since there needs to exist an entire directory for it
- Large parts of the code base are just wrappers around the gocli, and provide no functionality of their own
- No consistency in the codebase, some options in the cluster are created by the gocli, some through the bash code, and some use a mix of both
- No way of using proper unit testing or any form of best practices due to the nature of bash as a scripting language

In addition to problems in the gocli, such as:

- Most areas have no test coverage
- The code doesn't follow any of the idiomatic go principles
- The code isn't split into reusable modules and is hard to navigate, expand or edit

And so the goal of this project was to consolidate this bash code into the gocli to highly enhance reusability, expandability, to overall reduce duplication and to make it a lot easier to create new versions, features.. etc.

## Achieved goals

### Main targets:

- Removal of ssh.sh, which is a script inside each container and is the way to reach the VM and introduces an unnecessary extra hop. A library for direct SSH connection to the VM from the gocli was introduced. [PR](https://github.com/kubevirt/kubevirtci/pull/1209)
- Moving all the bash scripts into a go package called `opts` and move all options that the cluster can have to the gocli, leverage that package in favor of the bash scripts. [PR](https://github.com/kubevirt/kubevirtci/pull/1217)
- Reorganize the code for VM based cluster into a module and reimplement all the method under a type called `KubevirtProvider` [PR](https://github.com/kubevirt/kubevirtci/pull/1230)
- Move all the code for kind cluster provisioning into the gocli, this code was previously pure bash [PR](https://github.com/kubevirt/kubevirtci/pull/1232)

### Extras:

- Create a new way for provisioning clusters through building container images into VM's through bootc [PR](https://github.com/kubevirt/kubevirtci/pull/1247)

Some of these PR's are under review to prove their compatibility with the Kubevirt release cycles and existing practices

## Challenges

The project wasn't free of hurdles. some of the obstacles faced:

- The project introduced a massive amount of dependencies and the repo uses vendoring, which makes PR organization a necessity to allow any sort of reviewing
- KubevirtCI is a relatively small yet critical project, this means that the changes introduced heavily impacted many existing practices. this had to always be put into consideration
- The project changed how kubernetes deployments are being created, from kubectl to `client-go`, which required the schemas for all the dependencies to be in the kubernetes client and changes to the used manifests, as well as creating a way for testing. for which the kubernetes fake client + reactors was used
- Some aspects of the refactor required specific knowledge of technologies with little to no documentation. such as SR-IOV emulation using QEMU
- Avoiding bloated method signatures while refactoring, as the cluster parameters are well over 20. and this problem appeared only when moving the code from bash
- When using bootc for image provisioning the resulting operating system had many problems that prevents the provisioning kubernetes clusters on it, such as a read-only file system, lack of ipv6 connectivity, which caused ipv4 assignments with DHCP to fail

## Conclusion

Kubevirt is a very special piece of software that requires equally special considerations in how its built and tested, through this rewrite of KubevirtCI i got to appreciate the amount of detail that kubevirt requires.
As well as getting to learn many concepts about operating systems and the open source community at large. Many thanks to the amazing maintainers of this project and the members of kubevirt SIG-CI
