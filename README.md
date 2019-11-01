# Geneva Transformation

## Authors

* Ernesto Ojeda
* Lisa Rashidi-Ranjbar
* James Gregg
* Emilio Reyes

## Background

* What is a Jenkinsfile? (James)
* Difference/benefits between JJB freestyle and Jenkins pipeline (Ernesto)
  * Job types:
      * Verify Job - What it do?
      * Merge Job - What it do?
      * Stage Job - What it do?
  * Plugins:
      * Before: GHPRB, trigger phrase (recheck)
      * After: GitHub Branch Source, automatic scanning of repo, automatic builds via webhooks, untrusted builds would be triggered manually.
  * What changes with Jenkins pipeline?
      * What happens the ci-management JJB template?
      * Easily debug where job fails
      * Job simplification, only one job per branch/PR
      * ...
* How Pipeline Simplifies the release management process (Lisa)
* git-semver (Lisa)
  * what is git-semver?
  * how does it work?
  * What happens to the `VERSION` file? Delete and add to .gitignore

## New Developer Requirements

* Jenkinsfile at the root of the repository.
* git-semver manages versions wherever possible.
* Repository must contain a Makefile
* Standard Makefile targets:
  * **test**: Run unit tests
  * **build**: Compile code
  * **version**: `cat VERSION 2>/dev/null || echo 0.0.0`. In this case the VERSION file is only used in the Jenkins pipeline and git-semver automatically writes the file to the workspace. When running locally a developer's version would be 0.0.0. This could be overridden by the developer if they so care to do so.

## Docker as the build runtime (Ernesto)

### Benefits

* Speed up builds
* Give us better granularity in Jenkins stages allowing us to pinpoint more easily where errors are occurring.

### Implementation

* Dockerfile.build: containing dependencies needed for code build. This image is used to run `make test` and/or `make build`.
* Dockerfile: multi-stage build. Parameterized `FROM` would allow reusing the image generated from Dockerfile.build as the builder image. Example:

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

## New Pre-Built Jenkins Pipelines (Ernesto)

* **edgeXBuildGoApp**: New pipeline that will be used for Go microservices
  * pipeline image here

* **edgeXBuildDocker**: A generic pipeline that builds and scans docker images.
  * pipeline image here

* **edgeXGeneric**: Easily convert existing JJB templates to pipelines
  * pipeline image here

* Before & After Pipelines
  * Before Jenkinsfile
      * [app-service-configurable](samples/app-service-configurable/Jenkinsfile.before)
      * [device-bacnet-c](samples/device-bacnet-c/Jenkinsfile.before)
  * Show after Jenkinsfile
      * [app-service-configurable](samples/app-service-configurable/Jenkinsfile.after)
      * [device-bacnet-c](samples/device-bacnet-c/Jenkinsfile.after)
