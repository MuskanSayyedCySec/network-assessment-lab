FROM debian:13-slim
RUN apt-get update \
    && apt-get install -y --no-install-recommends nmap curl redis-tools \
    && rm -rf /var/lib/apt/lists/*