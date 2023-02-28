---
layout: post
title:  "Build a docker image with Argo Workflows"
---

# Building a docker image with Argo Workflows

In this post, we will learn how to build a docker image using Argo Workflows, Kaniko, scan it with Trivy, and push it to an OCI Image repository.  

## Using Argo Workflows for Continuous Integration

One of the common responsibilities of a DevOps Engineer is maintaining CI/CD for their company.  There are many tools with various tradeoffs.  Some are free, others are paid, some are fully integrated with your VCS (hopefully git). [Jenkins](https://www.jenkins.io/) is the venerable beast most have used at some point.  Many of the newer tools are using YAML to abstract away much of what would have been done in groovy with Jenkins.  Some examples of these tools in no particular order are [Gitlab Runners](https://docs.gitlab.com/runner/), [Github Actions](https://github.com/features/actions), and [Drone CI](https://www.drone.io/).  But what if you want to do CI in a less-proprietary cloud-agnostic Kubernetes native way?  Enter [Argo Workflows](https://argoproj.github.io/argo-workflows/).  

In the words of the [Argo Project](https://argoproj.github.io), Argo Workflows is an open source container-native workflow engine for orchestrating parallel jobs on Kubernetes. Argo Workflows can be deployed the same, no matter where you run it.  Whether you are in an airgapped on-prem Kubernetes cluster, or a managed Kubernetes cluster with a  cloud provider, you can use Argo Workflows the same way.  It also recently graduated 🎓🎉 as a CNCF Project.

### Key features of Argo Workflows

* Vendor and Cloud Agnostic
* Directed Acyclic Graphs
* Kubernetes Native
* Implemented as CRDs
* Can be used for many things, like Data Science or Infrastructure as Code implementation

### WTF is a Directed Acyclic Graph?

A Directed Acyclic Graph or DAG, is a way to define steps or tasks inside of a workflow by declaring its dependencies, as opposed to declaring an explicit order.  By doing this, you enable the workflow scheduler to achieve maximum parallelism without having to worry about factors such as the execution time of each individual step.

### Prerequisites

This tutorial starts off by assuming you have a working install of Argo Workflows and can access the UI. If you don't have that, you can follow the [official documentation](https://argoproj.github.io/argo-workflows/quick-start/). For this tutorial you do not need to do anything like configuring artifact repositories.  You will also need a working Kubeconfig for the cluster that you will be developing, as well as have configured the [Argo CLI](https://argoproj.github.io/argo-workflows/quick-start/#install-the-argo-workflows-cli). 

### A typical CI Workflow

A typical Continuous Integration workflow looks something like this:

    1. Checkout Code
    2. Build Code
    3. Run Tests
    4. Run Security and Vulnerability Scanning
    5. Publish/Release Code

### How can we achieve this in Argo Workflows?

    ---
    apiVersion: argoproj.io/v1alpha1
    kind: WorkflowTemplate
    metadata:
      name: ci-template
    spec:
      entrypoint: ci
      artifactGC:
        strategy: OnWorkflowDeletion
      volumeClaimTemplates:
        - metadata:
            name: workdir
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 500Mi
      arguments:
        parameters:
          - name: repo
            value: https://github.com/argoproj/argo-workflows.git
          - name: revision
            value: v2.1.1"
          - name: oci-registry
            value: docker.io
          - name: oci-image
            value: templates/workflow
          - name: oci-tag
            value: v1.1.1
          - name: push-image
            value: false
      templates:
        - name: ci
          dag:
            tasks:
              - name: git-clone
                template: git-clone
              - name: ls
                template: ls
                dependencies:
                  - git-clone
              - name: build
                template: build
                dependencies:
                  - git-clone
                  - ls
              - name: trivy-image-scan
                template: trivy-image-scan
                dependencies:
                  - build
              - name: trivy-filesystem-scan
                template: trivy-filesystem-scan
                dependencies:
                  - build
              - name: push-image
                template: push-image
                when: "{{workflow.parameters.push-image}} == true"
                dependencies:
                  - trivy-filesystem-scan
                  - trivy-image-scan
        - name: git-clone
          inputs:
            parameters:
              - name: repo
                value: "{{workflow.parameters.repo}}"
              - name: revision
                value: "{{workflow.parameters.revision}}"
            artifacts:
              - name: argo-source
                path: /src
                git:
                  repo: "{{inputs.parameters.repo}}"
                  revision: "{{inputs.parameters.revision}}"
          container:
            image: alpine:3.17
            command:
              - sh
              - -c
            args:
              - cp -r /src/* .
            workingDir: /workdir
            volumeMounts:
              - name: workdir
                mountPath: /workdir
        - name: ls
          container:
            image: alpine:3.17
            command:
              - sh
              - -c
            args:
              - ls /
            workingDir: /workdir
            volumeMounts:
              - name: workdir
                mountPath: /workdir
        - name: build
          inputs:
            parameters:
              - name: oci-image
                value: "{{workflow.parameters.oci-image}}"
              - name: oci-tag
                value: "{{workflow.parameters.oci-tag}}"
          container:
            image: gcr.io/kaniko-project/executor:latest
            args:
              - --context=/workdir
              - --destination={{inputs.parameters.oci-image}}:{{inputs.parameters.oci-tag}}
              - --no-push
              - --tar-path=/workdir/{{inputs.parameters.oci-tag}}.tar
            workingDir: /workdir
            volumeMounts:
              - name: workdir
                mountPath: /workdir
        - name: trivy-image-scan
          inputs:
            parameters:
              - name: oci-tag
                value: "{{workflow.parameters.oci-tag}}"
          container:
            image: aquasec/trivy
            args:
              - image
              - --input=/workdir/{{inputs.parameters.oci-tag}}.tar
            env:
              - name: DOCKER_HOST
                value: tcp://127.0.0.1:2375
            volumeMounts:
              - name: workdir
                mountPath: /workdir
          sidecars:
            - name: dind
              image: docker:23.0.1-dind
              command:
                - dockerd-entrypoint.sh
              env:
                - name: DOCKER_TLS_CERTDIR
                  value: ""
              securityContext:
                privileged: true
              mirrorVolumeMounts: true
        - name: trivy-filesystem-scan
          inputs:
            parameters:
              - name: oci-tag
                value: "{{workflow.parameters.oci-tag}}"
          container:
            image: aquasec/trivy
            args:
              - filesystem
              - /workdir
              - --ignorefile=/workdir/{{inputs.parameters.oci-tag}}.tar
            volumeMounts:
              - name: workdir
                mountPath: /workdir
        - name: push-image
          inputs:
            parameters:
              - name: oci-tag
                value: "{{workflow.parameters.oci-tag}}"
              - name: oci-image
                value: "{{workflow.parameters.oci-image}}"
              - name: oci-registry
                value: "{{workflow.parameters.oci-registry}}"
          script:
            image: gcr.io/go-containerregistry/crane:debug
            env:
              - name: OCI_REGISTRY_USER
                valueFrom:
                  secretKeyRef:
                    name: registry-credentials
                    key: username
              - name: OCI_REGISTRY_PASSWORD
                valueFrom:
                  secretKeyRef:
                    name: registry-credentials
                    key: password
            command:
              - sh
            source: >
              crane auth login -u $OCI_REGISTRY_USER -p $OCI_REGISTRY_PASSWORD {{inputs.    parameters.oci-registry}}
              crane push /workdir/{{inputs.parameters.oci-tag}}.tar {{inputs.parameters.    oci-registry}}/{{inputs.parameters.oci-image}}:{{inputs.parameters.oci-tag}}
            volumeMounts:
              - name: workdir
                mountPath: /workdir
    

### The Breakdown


    apiVersion: argoproj.io/v1alpha1
    kind: WorkflowTemplate
    metadata:
      name: ci-template
    spec:
      entrypoint: ci #Specifies what task the workflow will start on by default
      artifactGC:
        strategy: OnWorkflowDeletion # tells argo to cleanup after itself
      volumeClaimTemplates:
        - metadata:
            name: workdir
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 500Mi
      arguments:
        parameters:
          - name: repo
            value: https://github.com/argoproj/argo-workflows.git
          - name: revision
            value: v2.1.1"
          - name: oci-registry
            value: docker.io
          - name: oci-image
            value: templates/workflow
          - name: oci-tag
            value: v1.1.1
          - name: push-image
            value: false

The VolumeClaimTemplate here is the notable part of this workflow.  Instead of configuring an artifact repository, Argo Workflows has a facility to allow for seamlessly mounting a volume across individual steps or tasks in the workflow.  This is extremely useful, because each task runs in its own ephemeral container.  It also allows the re-use of assets between tasks without having to upload or download from somewhere else, as this is my biggest gripe with artifact caching in Github Actions.

Below that, I specify workflow level parameters and their default values.  These will be the defaults shown in the UI, and can be overridden there or in the on the command line.


      templates:
        - name: ci
          dag:
            tasks:
              - name: git-clone
                template: git-clone
              - name: ls
                template: ls
                dependencies:
                  - git-clone
              - name: build
                template: build
                dependencies:
                  - git-clone
                  - ls
              - name: trivy-image-scan
                template: trivy-image-scan
                dependencies:
                  - build
              - name: trivy-filesystem-scan
                template: trivy-filesystem-scan
                dependencies:
                  - build
              - name: push-image
                template: push-image
                when: "{{workflow.parameters.push-image}} == true"
                dependencies:
                  - trivy-filesystem-scan
                  - trivy-image-scan

Above, under `templates:` is where the actual tasks are defined.  You can see here that the first template is named `ci`, which matches the `entrypoint` specified earlier.  That means that unless overridden, this step `ci` will be executed when you run the workflow.  

You can also see that this is a `dag`.  I have listed out all the other templates, or tasks to be performed as well as their dependencies.  When any task has all of its dependencies complete, it will immediately begin execution.  In this workflow, it looks a bit like this:  ![argo-workflows-dag](/assets/images/argo-workflows-dag.png)

You can see that `git-clone` is the first step, followed by `ls`. `build` needs to wait for both of those steps, next up are the two trivy scans.  `trivy-image-scan` depends on the build stage, as it is scanning the actual docker image.  On the other hand, `trivy-filesystem-scan` can begin executing immediately after the git clone.  Once both are complete `push-image` can begin execution, provided that `workflow.parameters.push-image` is set to `true`.

        - name: git-clone
          inputs:
            parameters:
              - name: repo
                value: "{{workflow.parameters.repo}}"
              - name: revision
                value: "{{workflow.parameters.revision}}"
            artifacts:
              - name: argo-source
                path: /src
                git:
                  repo: "{{inputs.parameters.repo}}"
                  revision: "{{inputs.parameters.revision}}"
          container:
            image: alpine:3.17
            command:
              - sh
              - -c
            args:
              - cp -r /src/* .
            workingDir: /workdir
            volumeMounts:
              - name: workdir
                mountPath: /workdir

This clones the repo specified by the parameters above, as well as the branch or ref specified in `revision`.  It then copies the downloaded source onto a volume mount so that it can be seamlessly passed between other tasks in this workflow.

There is a step called `ls` that is mainly included just for log output and to show how simple tasks can be performed. 

        - name: build
          inputs:
            parameters:
              - name: oci-image
                value: "{{workflow.parameters.oci-image}}"
              - name: oci-tag
                value: "{{workflow.parameters.oci-tag}}"
          container:
            image: gcr.io/kaniko-project/executor:latest
            args:
              - --context=/workdir
              - --destination={{inputs.parameters.oci-image}}:{{inputs.parameters.oci-tag}}
              - --no-push
              - --tar-path=/workdir/{{inputs.parameters.oci-tag}}.tar
            workingDir: /workdir
            volumeMounts:
              - name: workdir
                mountPath: /workdir

In this step, I use Kaniko to build the image.  Kaniko is a tool for building OCI/Docker images inside of Kubernetes.  It avoids many of the pitfalls of running Docker in Docker (or in this case Docker in containerd).   As you can see, it also  doesn't require `root` privilege.  I also important specify output to the volume mount as a tar file, so that it can be used in downstream tasks. 

        - name: trivy-image-scan
          inputs:
            parameters:
              - name: oci-tag
                value: "{{workflow.parameters.oci-tag}}"
          container:
            image: aquasec/trivy
            args:
              - image
              - --input=/workdir/{{inputs.parameters.oci-tag}}.tar
            env:
              - name: DOCKER_HOST
                value: tcp://127.0.0.1:2375
            volumeMounts:
              - name: workdir
                mountPath: /workdir
          sidecars:
            - name: dind
              image: docker:23.0.1-dind
              command:
                - dockerd-entrypoint.sh
              env:
                - name: DOCKER_TLS_CERTDIR
                  value: ""
              securityContext:
                privileged: true
              mirrorVolumeMounts: true

In this step, I use Trivy to scan the previously built docker image.  Trivy requires Docker, so how are we going to do that inside of this workflow?  You can see that in the main Trivy container I specify the standard docker environment variable of `DOCKER_HOST` I point this to `localhost` which refers to the sidecar running the official `docker:23.0.1-dind` image.  You can see that this requires `privileged` access to execute successfully, one of the many issues with running Docker in Docker.

Next, I run a Trivy filesystem scan of the cloned source repo.  I will skip explaining this step. 

        - name: push-image
          inputs:
            parameters:
              - name: oci-tag
                value: "{{workflow.parameters.oci-tag}}"
              - name: oci-image
                value: "{{workflow.parameters.oci-image}}"
              - name: oci-registry
                value: "{{workflow.parameters.oci-registry}}"
          script:
            image: gcr.io/go-containerregistry/crane:debug
            env:
              - name: OCI_REGISTRY_USER
                valueFrom:
                  secretKeyRef:
                    name: registry-credentials
                    key: username
              - name: OCI_REGISTRY_PASSWORD
                valueFrom:
                  secretKeyRef:
                    name: registry-credentials
                    key: password
            command:
              - sh
            source: >
              crane auth login -u $OCI_REGISTRY_USER -p $OCI_REGISTRY_PASSWORD {{inputs.    parameters.oci-registry}}
              crane push /workdir/{{inputs.parameters.oci-tag}}.tar {{inputs.parameters.    oci-registry}}/{{inputs.parameters.oci-image}}:{{inputs.parameters.oci-tag}}
            volumeMounts:
              - name: workdir
                mountPath: /workdir

Finally, I use `crane` to push the newly built and scanned image to an OCI or Docker registry.  In this step, I use the `source` command to specify that I am executing an inline script instead of a command and arguments.  I mount a secret into the environment and then reference those credentials as environment variables.  You will see that it is also possible to reference inputs and parameters directly inside of an inline script.  

### Submitting the workflow:

From the UI, it's fairly straight forward.  From the command line, it looks a bit like this: `argo submit --from workflowtemplate/ci-template -p repo="https://github.com/mikenabhan/proxmox-kubernetes-cluster-autoscaler.git" -p revision="main" -p oci-image="mikenabhan/proxmox-kubernetes-cluster-autoscaler" -p oci-tag="test-tag" -p push-image="false" --watch`


That's it, you've now built a docker image using Argo Workflows, Kaniko, and scanned it for security vulnerabilities using Trivy.  All in a cloud-agnostic manner. 
