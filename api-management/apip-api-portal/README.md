# API Portal

Dockerfiles for the WSO2 API Portal images: `1.0.0/api-portal/alpine` (API Portal)
and `1.0.0/platform-api/ubuntu` (Platform API).

## Building an image

### Step 1: Retrieve a registry token

Generate a registry token as described in
[Registry Token Management](https://updates.docs.choreolabs.com/en/latest/updates/registry-token-management-customer/).

### Step 2: Set your credentials

```bash
export REGISTRY_USERNAME='<token-name>'
export REGISTRY_PASSWORD='<token-secret>'
```

### Step 3: Build the image

Pass the product version to build, and the directory that holds the Dockerfile
(for example `1.0.0/api-portal/alpine`):

```bash
docker build \
    --build-arg PRODUCT_VERSION=<product-version> \
    --secret id=REGISTRY_USERNAME \
    --secret id=REGISTRY_PASSWORD \
    -t <user-registry>/<user-image>:<user-image-version> <dockerfile-dir>
```

For example, to build both images of product version 1.0.0.7 from this directory:

```bash
docker build \
    --build-arg PRODUCT_VERSION=1.0.0.7 \
    --secret id=REGISTRY_USERNAME \
    --secret id=REGISTRY_PASSWORD \
    -t registry.example.com/api-portal:1.0.0.11 1.0.0/api-portal/alpine

docker build \
    --build-arg PRODUCT_VERSION=1.0.0.7 \
    --secret id=REGISTRY_USERNAME \
    --secret id=REGISTRY_PASSWORD \
    -t registry.example.com/platform-api:0.16.0.6 1.0.0/platform-api/ubuntu
```

### Both architectures

To build for `linux/amd64` and `linux/arm64` together, use `docker buildx build`
with the same arguments. Docker's default image store keeps one architecture per
image, so push the result straight to your registry:

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
    --build-arg PRODUCT_VERSION=<product-version> \
    --secret id=REGISTRY_USERNAME \
    --secret id=REGISTRY_PASSWORD \
    -t <user-registry>/<user-image>:<user-image-version> --push <dockerfile-dir>
```

## Changing the base image

Edit the `FROM` lines after the `binary` stage, and keep the image's runtime
contract (user, ports, paths, and entrypoint) unchanged.

- **API Portal:** the base must provide Node.js 24, and npm in the build stage.
- **Platform API:** the binary uses cgo, so the base must be glibc-based (not
  Alpine). The Dockerfile uses `apt-get`, `groupadd` and `useradd`; replace them
  with your base's equivalents, and keep UID 10001.

## Requirements

Docker with BuildKit, which is the default since Docker 23.0.
