---
icon: material/docker
---

#### Docker Runtime Vulnerability Scanning

#### CIS benchmark
```bash
docker run --rm --net host --pid host --userns host --cap-add audit_control \
    -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
    -v /etc:/etc:ro \
    -v /usr/bin/containerd:/usr/bin/containerd:ro \
    -v /usr/bin/runc:/usr/bin/runc:ro \
    -v /usr/lib/systemd:/usr/lib/systemd:ro \
    -v /var/lib:/var/lib:ro \
    -v /var/run/docker.sock:/var/run/docker.sock:ro \
    --label docker_bench_security \
    docker/docker-bench-security
```

#### Secured Docker daemon
```bash
vi /etc/docker/daemon.json
{
    "icc": false,
    "storage-driver": "overlay2",
    "default-ulimit": true,
    "userns-remap": "default",
    "log-driver": "syslog",
    "live-restore": true,
    "userland-proxy": false,
    "no-new-privileges": true,
    "hosts": ["fd://", "unix:///var/run/docker.sock", "tcp://127.0.0.1:2376"],
    "tls": true,
    "tlsverify": true,
    "tlscacert": "/etc/docker/tls/cacert.pem",
    "tlscert": "/etc/docker/tls/server-cert.pem",
    "tlskey": "/etc/docker/tls/server-key.pem",
    "graph": "/var/lib/docker"
}
```