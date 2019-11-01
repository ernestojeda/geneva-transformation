# Geneva Transformation

## Authors

* Ernesto Ojeda
* Lisa Rashidi-Ranjbar
* James Gregg
* Emilio Reyes

## Background

* What is a Jenkinsfile?
* Difference/benefits between JJB freestyle and Jenkins Pipeline.
  * **JJB Freestyle**: Yaml based, managed separate from source code that it builds. Hard to trace through all the templates and scripts. Relies on old paradigms to manage Jenkins jobs.
  * **Jenkins Pipeline**: Suite of plugins that enables 'Pipeline-As-Code'. It allows users to define a build descriptor (Jenkinsfile) right next to the code to more easily manage code builds. More industry adoption/support. Officially supported by Cloudbees (makers of Jenkins).

## Plugins
  * Before: [GitHub Pull Request Builder](https://github.com/jenkinsci/ghprb-plugin) aka. GHPRB. This plugin is currently responsible for building all the pull requests. Plugin is no longer being actively developed and is up for adoption. Comments direct users to use the GitHub Branch Source plugin. Automatic builds on code push and allows triggering via GitHub comments (i.e. recheck).
  * After: [GitHub Branch Source](https://docs.cloudbees.com/docs/admin-resources/latest/plugins/github-branch-source). Automatic scanning of GitHub at the organization level. Any branch or PR where a Jenkinsfile is found will automatically be built. Builds are triggered via GitHub webhooks, untrusted code, i.e. code that is committed by users without committer privileges, would require a manual trigger via the Jenkins UI.

## Job types

### Verify Job
  * **Summary**: This is a Jenkins freestyle job created per branch/release stream (master, edinburgh, fuji, etc). This job typically does unit tests and builds the code and docker images.
  * **Trigger**: [GitHub Pull Request Builder](https://github.com/jenkinsci/ghprb-plugin) plugin. Triggered when a user opens a pull request or recheck.
  * **Management**: JJB template in the ci-management repository.
  * **Example**: ([edgex-go-master-verify-go](https://jenkins.edgexfoundry.org/job/edgex-go-master-verify-go/)). 

### Merge Job
  * **Summary**: This is a Jenkins freestyle job created per branch/release stream (master, edinburgh, fuji, etc). This job typically does the same steps as the **verify** job and will push any generated docker images/artifacts to the **snapshots** nexus3 and nexus2 repositories.
  * **Trigger**: [GitHub Webhook](https://help.github.com/en/github/extending-github/about-webhooks). Triggered when a pull request is merged into **master/release stream branch** (master, edinburgh, fuji, etc).
  * **Management**: JJB template in ci-management.
  * **Example**: [edgex-go-master-merge-go](https://jenkins.edgexfoundry.org/job/edgex-go-master-merge-go/). 

### Stage Job
  * **Summary**: This is a Jenkins freestyle job created per branch/release stream (master, edinburgh, fuji, etc). Similar to the merge job, this job typically does the same steps as the **verify** job and will push any generated docker images/artifacts to the **staging** nexus3 and nexus2 repositories.
  * **Trigger**: Nightly cron.
  * **Management**: JJB template in ci-management.
  * **Example**: [edgex-go-master-stage-go](https://jenkins.edgexfoundry.org/job/edgex-go-master-stage-go/). 

## What changes with Jenkins pipeline
* **Q: For Geneva, will we still need have verify, merge and stage jobs?**
* A: No. All details related to build verification, pushing docker images will be moved into the Jenkinsfile. Also, stage jobs will no longer be needed as we will be pushing docker images to nexus staging on merges into master.
* **Q: Will there still be a need for any JJB templates?**
* A: Yes. Some JJB templates will remain for older releases, i.e. Edinburgh, Fuji. In addition, there will be one JJB template for the edgexfoundry GitHub organization as a whole which will discover branches to build.
* **Q: Will it be easier to find build failures?**
* A: Yes, Jenkins pipeline defines the concept of named stages which can help pinpoint where failures are happening more easily.
* **Q: Will it be simpler to navigate jobs?**
* A: Yes, there will be only one high level job and you will be able to easily navigate to the branch/PR  you are concerned about.
* **Q: Can I still trigger the build with a 'recheck'?**
* A: No, just pushing your branch will trigger the build. If you require a new build and the code has not changed, you can push and empty commit.


## How Pipeline Simplifies the release management process
Summary...

* git-semver (Lisa)
  * what is git-semver?
  * how does it work?
  * What happens to the `VERSION` file? Delete and add to .gitignore


## New Geneva Pre-Built Jenkins Pipelines

Geneva introduces a set of pre-built Jenkins pipelines for use by the EdgeX community. These pipelines 

### edgeXBuildGoApp

New pipeline that will be used for Go microservices.

![edgeXBuildGoApp.png](images/edgeXBuildGoApp.png)

### edgeXBuildDocker

A generic pipeline that builds and scans docker images.

![edgeXBuildDocker.png](images/edgeXBuildDocker.png)

### edgeXGeneric

Easily convert existing JJB templates to pipelines. Will be used for more complicated jobs as a temporary stop gap until a more custom pipeline can be created.

![edgeXGeneric.png](images/edgeXGeneric.png)


## Migrating Existing Pipelines

### Examples

* Before Jenkinsfile
  * [app-service-configurable](samples/app-service-configurable/Jenkinsfile.before)
  * [device-bacnet-c](samples/device-bacnet-c/Jenkinsfile.before)
* After Jenkinsfile
  * [app-service-configurable](samples/app-service-configurable/Jenkinsfile.after)
  * [device-bacnet-c](samples/device-bacnet-c/Jenkinsfile.after)

## New CI Requirements

### All Repositories

* Jenkinsfile at the root of the repository.
* git-semver manages versions wherever possible.
* Repository must contain a Makefile
* Standard Makefile targets:
  * **test**: Run unit tests
  * **build**: Compile code
  * **version**: `cat VERSION 2>/dev/null || echo 0.0.0`. In this case the VERSION file is only used in the Jenkins pipeline and git-semver automatically writes the file to the workspace. When running locally a developer's version would be 0.0.0. This could be overridden by the developer if they so care to do so.

### Microservice Type Repositories

### Leverage Docker as the build runtime, rather than underlying host.

#### Benefits

* Optimization of builds due to caching of docker image layers.
* Installation of custom tooling can be done inside of docker rather than Packer...allowing for easier re-use.

### Implementation

* **Dockerfile.build**: containing dependencies needed for code build. This image is used to run `make test` and/or `make build`.
* **Dockerfile**: multi-stage build. Parameterized `FROM` would allow reusing the image generated from Dockerfile.build as the builder image. Example:

#### Dockerfile.build

```Dockerfile
FROM golang:1.12-alpine
RUN apk add --update make git zmq-dev

COPY . .

RUN go mod download
```

### Building the base image

The CI process now creates the build image and subsequent make commands are executed inside a running container.

`docker build -t base-build -f Dockerfile.build .`

Then run make commands inside the base image

`docker run --rm -t -v /path/to/workspace:/path/to/workspace base-build make test && make ...`

### Building the final image

#### Dockerfile

```Dockerfile
ARG BASE=golang:1.12-alpine
FROM ${BASE} as builder
RUN apk add --update make git zmq-dev
COPY . .
RUN make build

FROM alpine
COPY --from=builder /go-binary /go-binary
ENTRYPOINT ["/go-binary"]
```

### Build base image Dockerfile.build

`docker build -t base-build -f Dockerfile.build .`

### Build final docker image

`docker build -t docker-edgex-binary -f Dockerfile --build-arg BASE=base-build .`
