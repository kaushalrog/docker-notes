# Docker Context Guide

Docker Contexts allow you to manage multiple Docker environments and easily switch between them. This is especially useful when you are managing local Docker installations alongside remote Docker endpoints, or multiple Kubernetes clusters.

## Basic Commands

### List Available Contexts
```bash
docker context ls
```

### Create a New Context
You can create a context that connects to a remote daemon over SSH:
```bash
docker context create remote-server --docker "host=ssh://user@remote-server"
```

### Use a Specific Context
Switch your docker CLI to use the newly created context:
```bash
docker context use remote-server
```

### Inspect a Context
```bash
docker context inspect remote-server
```

### Update a Context
```bash
docker context update remote-server --description "My Remote Server Environment"
```

### Remove a Context
```bash
docker context rm remote-server
```

### Environment Variable
You can also use the `DOCKER_CONTEXT` environment variable to override the current context temporarily:
```bash
DOCKER_CONTEXT=remote-server docker ps
```
