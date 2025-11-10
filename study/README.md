# libuv Study Guide

## Docker Environment

### Start Docker Container
```bash
docker-compose up -d
```

### Access Docker Container
```bash
docker-compose exec libuv-dev /bin/bash
```

### Stop Docker Container
```bash
docker-compose down
```

## Build Instructions

### Build Example Code 

```bash
cd /workspace/docs/code
cmake -B build
cmake --build build
```

### Run Examples
```bash
# Basic examples
./build/helloworld
./build/default-loop
./build/idle-basic

# Network examples
./build/tcp-echo-server
./build/udp-dhcp

# Other examples
./build/timer
./build/signal
```
