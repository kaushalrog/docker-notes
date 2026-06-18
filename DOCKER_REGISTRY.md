# Docker Registry Guide

A Docker Registry is a storage and content delivery system, holding named Docker images, available in different tagged versions.

## Public vs Private Registry
- **Public Registry**: Docker Hub is the default public registry.
- **Private Registry**: You can host your own registry to keep images private.

## Running a Local Registry

You can run a local registry for testing or internal network use.

### Start a local registry
```bash
docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

### Tag an image for the local registry
```bash
docker tag my-image localhost:5000/my-image
```

### Push the image to the local registry
```bash
docker push localhost:5000/my-image
```

### Pull the image from the local registry
```bash
docker pull localhost:5000/my-image
```

### Stop and remove the registry
```bash
docker container stop registry && docker container rm -v registry
```
