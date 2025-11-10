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

## Build Documentation

### Setup Python Virtual Environment
```bash
# Create virtual environment
python3 -m venv /workspace/.venv

# Activate virtual environment
source /workspace/.venv/bin/activate
```

### Install Dependencies and Build
```bash
# Install Sphinx and dependencies
cd /workspace/docs
pip install -r requirements.txt

# Build HTML documentation
make html
```

### View Documentation
```bash
# Built documentation location
# /workspace/docs/build/html/index.html

# View with web server (optional)
cd /workspace/docs/build/html
python3 -m http.server 8000
# Access http://localhost:8000 in your browser
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
