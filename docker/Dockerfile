FROM bitcoin/bitcoin:30.0 AS bitcoin

FROM ubuntu:24.04

# Install dependencies for sv2-tp
RUN apt-get update && apt-get install -y \
    cmake \
    g++ \
    libboost-dev \
    capnproto \
    libcapnp-dev \
    pkg-config \
    python3 \
    supervisor \
    && rm -rf /var/lib/apt/lists/*

# Copy Bitcoin Core binaries from official image
COPY --from=bitcoin /opt/bitcoin-30.0/bin/bitcoin /usr/local/bin/
COPY --from=bitcoin /opt/bitcoin-30.0/libexec/bitcoin-node /usr/local/libexec/

# Build sv2-tp
WORKDIR /build
COPY . .
RUN rm -rf build && \
    CMAKE_PREFIX_PATH=/usr/lib/$(uname -m)-linux-gnu/cmake cmake -B build && \
    cmake --build build -j$(nproc) && \
    cp build/bin/sv2-tp /usr/local/bin/

# Create config files in /etc (since /root/.bitcoin will be a volume mount)
RUN mkdir -p /etc/sv2-tp && \
    echo "chain=testnet4" > /etc/sv2-tp/sv2-tp.conf && \
    echo "ipcconnect=unix" >> /etc/sv2-tp/sv2-tp.conf && \
    echo "sv2bind=0.0.0.0" >> /etc/sv2-tp/sv2-tp.conf && \
    echo "sv2port=8442" >> /etc/sv2-tp/sv2-tp.conf && \
    echo "sv2interval=30" >> /etc/sv2-tp/sv2-tp.conf && \
    echo "sv2feedelta=1000" >> /etc/sv2-tp/sv2-tp.conf && \
    echo "debug=sv2" >> /etc/sv2-tp/sv2-tp.conf && \
    echo "loglevel=sv2:trace" >> /etc/sv2-tp/sv2-tp.conf && \
    echo "debug=ipc" >> /etc/sv2-tp/sv2-tp.conf && \
    echo "printtoconsole=1" >> /etc/sv2-tp/sv2-tp.conf

# Create supervisor config to run both processes
RUN echo "[supervisord]" > /etc/supervisor/conf.d/supervisord.conf && \
    echo "nodaemon=true" >> /etc/supervisor/conf.d/supervisord.conf && \
    echo "user=root" >> /etc/supervisor/conf.d/supervisord.conf && \
    echo "" >> /etc/supervisor/conf.d/supervisord.conf && \
    echo "[program:bitcoind]" >> /etc/supervisor/conf.d/supervisord.conf && \
    echo "command=/usr/local/bin/bitcoin -m node -testnet4 -ipcbind=unix -datadir=/root/.bitcoin -server=1 -rpcuser=user -rpcpassword=password -rpcbind=0.0.0.0 -rpcallowip=172.30.0.0/16" >> /etc/supervisor/conf.d/supervisord.conf && \
    echo "autostart=true" >> /etc/supervisor/conf.d/supervisord.conf && \
    echo "autorestart=true" >> /etc/supervisor/conf.d/supervisord.conf && \
    echo "stdout_logfile=/dev/stdout" >> /etc/supervisor/conf.d/supervisord.conf && \
    echo "stdout_logfile_maxbytes=0" >> /etc/supervisor/conf.d/supervisord.conf && \
    echo "stderr_logfile=/dev/stderr" >> /etc/supervisor/conf.d/supervisord.conf && \
    echo "stderr_logfile_maxbytes=0" >> /etc/supervisor/conf.d/supervisord.conf && \
    echo "" >> /etc/supervisor/conf.d/supervisord.conf && \
    echo "[program:sv2-tp]" >> /etc/supervisor/conf.d/supervisord.conf && \
    echo "command=/usr/local/bin/sv2-tp -conf=/etc/sv2-tp/sv2-tp.conf" >> /etc/supervisor/conf.d/supervisord.conf && \
    echo "autostart=true" >> /etc/supervisor/conf.d/supervisord.conf && \
    echo "autorestart=true" >> /etc/supervisor/conf.d/supervisord.conf && \
    echo "stdout_logfile=/dev/stdout" >> /etc/supervisor/conf.d/supervisord.conf && \
    echo "stdout_logfile_maxbytes=0" >> /etc/supervisor/conf.d/supervisord.conf && \
    echo "stderr_logfile=/dev/stderr" >> /etc/supervisor/conf.d/supervisord.conf && \
    echo "stderr_logfile_maxbytes=0" >> /etc/supervisor/conf.d/supervisord.conf && \
    echo "startsecs=15" >> /etc/supervisor/conf.d/supervisord.conf

WORKDIR /root

EXPOSE 48332 48333 8442

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
