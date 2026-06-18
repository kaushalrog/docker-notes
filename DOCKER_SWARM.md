# Docker Swarm Guide

Docker Swarm is native clustering for Docker. It turns a pool of Docker hosts into a single, virtual Docker host.

## Key Concepts
- **Nodes**: A node is an instance of the Docker engine participating in the swarm.
- **Manager Nodes**: Dispatch tasks, maintain swarm state.
- **Worker Nodes**: Receive and execute tasks.
- **Services**: The definition of the tasks to execute on the manager or worker nodes.

## Basic Commands

### Initialize a Swarm
```bash
docker swarm init --advertise-addr <MANAGER-IP>
```

### Join a Swarm as a Worker
```bash
docker swarm join --token <WORKER-TOKEN> <MANAGER-IP>:2377
```

### List Nodes in the Swarm
```bash
docker node ls
```

### Create a Service
```bash
docker service create --replicas 3 --name helloworld alpine ping docker.com
```

### List Services
```bash
docker service ls
```

### Inspect a Service
```bash
docker service ps helloworld
```

### Scale a Service
```bash
docker service scale helloworld=5
```
