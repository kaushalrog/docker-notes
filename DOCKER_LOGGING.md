# Docker Logging Guide

Docker includes multiple logging mechanisms to help you get information from running containers and services.

## Logging Drivers
Docker configures the `json-file` logging driver by default, but it supports others:
- `json-file`: The logs are formatted as JSON. (Default)
- `syslog`: Writes logging messages to the syslog facility.
- `journald`: Writes log messages to journald.
- `gelf`: Writes log messages to a Graylog Extended Log Format (GELF) endpoint.
- `fluentd`: Writes log messages to fluentd.
- `awslogs`: Writes log messages to Amazon CloudWatch Logs.
- `splunk`: Writes log messages to Splunk using the HTTP Event Collector.
- `local`: Custom local logging driver.

## Configuring Default Logging Driver
You can change the default logging driver in `daemon.json` (usually `/etc/docker/daemon.json`):

```json
{
  "log-driver": "syslog"
}
```

## Configuring Driver per Container
You can set a different driver for a specific container:

```bash
docker run --log-driver syslog --name my-syslog-container alpine echo "Hello Syslog"
```

## Viewing Logs
To view logs for a container:
```bash
docker logs <CONTAINER_ID_OR_NAME>
```

Follow logs continuously:
```bash
docker logs -f <CONTAINER_ID_OR_NAME>
```

Show timestamps:
```bash
docker logs -t <CONTAINER_ID_OR_NAME>
```

Tail the last N lines:
```bash
docker logs --tail 100 <CONTAINER_ID_OR_NAME>
```

## Log Rotation
To prevent logs from consuming all disk space, configure log rotation:
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```
