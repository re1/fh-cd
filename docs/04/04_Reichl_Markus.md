# Docker

The application from the [previous exercises](https://github.com/re1/fh-cd) was used instead of the [example project]
(https://github.com/mrckurz/cd2020-ex04).

## Dockerfile

First, create a Dockerfile based on the instructions found in the example project repository.

```dockerfile
FROM golang:1.20.3-alpine

# Set maintainer label: maintainer=[YOUR-EMAIL]
LABEL maintainer="markus@re1.at"

# Set working directory: `/src`
WORKDIR /src

# Copy local source files `main.go` to the working directory
COPY . .

# List items in the working directory (ls)
RUN ls -a

# Build the GO app as myapp binary and move it to /usr/
RUN go build -o myapp
RUN mv myapp /usr/

# Expose port 8888
EXPOSE 8888

# Run the service myapp when a container of this image is launched
CMD ["/usr/myapp"]
```

## Image Build

To build a Docker image based on the Dockerfile the following command was used:

```bash
docker buildx build -f Dockerfile -t re1coy/fh-cd:0.0.1 ./
```

Note that this command requires the [buildx](https://docs.docker.com/buildx/working-with-buildx/) plugin to be installed.
The plugin is used, as the `docker build` command is deprecated and will not be available in future versions of Docker.

To check if the image was built correctly, list all images using the following command:

```bash
docker images
```

## Docker Hub

First, login to Docker Hub using the following command:

```bash
docker login
```

Then, push the image to Docker Hub:

```bash
docker image push re1coy/fh-cd:0.0.1
```

The image should then be listed on [Docker Hub](https://hub.docker.com/).

## Run Container

The deployed image can be used to run a container using the following command:

```bash
docker container run -p 9090:8888 re1coy/fh-cd:0.0.1
```

To check if the container is running, list all running containers using the following command:

```bash
docker container ls
```

The container can be stopped again with

```bash
docker stop <CONTAINER-ID>
```

## GitHub Actions

Based on the instructions in the [GitHub documentation](https://docs.github.com/en/actions/publishing-packages/publishing-docker-images), the following build job was added to the workflow file:

```yaml
jobs:
  hub:
    name: Push Docker image to Docker Hub
    runs-on: ubuntu-latest
    steps:
      - name: Check out the repo
        uses: actions/checkout@v3

      - name: Log in to Docker Hub
        uses: docker/login-action@65b78e6e13532edd9afa3aa52ac7964289d1a9c1
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Extract metadata (tags, labels) for Docker
        id: meta
        uses: docker/metadata-action@9ec57ed1fcdbf14dcef7dfbe97b2010124a938b7
        with:
          images: re1coy/fh-cd
          tags: type=sha,format=long

      - name: Build and push Docker image
        uses: docker/build-push-action@f2a1d5e99d037542a71f64918e516c093c6f3fc4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

Additionally, the docker username and password have to be added as secrets to the repository.

### Tag image with commit hash

To tag the image with the commit hash, the following line was added to metadata step in the hub job of the workflow file:

```yaml
- name: Extract metadata (tags, labels) for Docker
  id: meta
  uses: docker/metadata-action@9ec57ed1fcdbf14dcef7dfbe97b2010124a938b7
  with:
    images: re1coy/fh-cd
    tags: type=sha
```

## Trivy

Based on the instructions found in the [Trivy repository](https://github.com/aquasecurity/trivy-action), the following steps were added to the build job in the workflow file:

### Docker Image Scan

```yaml
- name: Run Trivy vulnerability scanner for in docker mode
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: "docker.io/re1coy/fh-cd:sha-${{ github.sha }}"
    format: "table"
    exit-code: "1"
    ignore-unfixed: true
    vuln-type: "os,library"
    severity: "CRITICAL"
```

As this

### Repository Scan

```yaml
- name: Run Trivy vulnerability scanner in repo mode
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: "fs"
    scan-ref: "."
    format: "sarif"
    output: "trivy-results.sarif"
    severity: "CRITICAL,HIGH"
```

Note that the image scan will need to run after the image has been built and pushed to Docker Hub.
If the scan is defined in its own job, it will need to define this dependency:

```yaml
jobs:
  trivy:
    needs: build
```
