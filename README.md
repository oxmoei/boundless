# Boundless Prover Guide

## Boundless Prover market
First, you need to know how **Boundless Prover market** actually works to realize what you are doing.
* **Requester Submits Ask**: A requester (e.g. developer) creates a task or computation `order` and submits an `ask` on Boundless, locking funds to incentivize participation.
* **Prover Places Bid**: A prover selects an `order`, submits a `bid`, stating their offered price or resources, which may be lower than the `ask`’s locked funds or other provers’ `bid`s.
* **Prover Locks Order**: If their `bid` is accepted among other provers (e.g., lower `bid`, sufficient stake, or meeting specific criteria), the prover locks the `order`, committing to perform the computational work.
* **Prover Generates Proof**: The prover completes the task and submits the `proof` to the network.
* **Verifier Checks Proof**: Verifiers validate the `proof` to ensure it meets the `orders`’s requirements and protocol standards.
* **Slashing**: If the `proof` is invalid, incomplete, or the prover fails to deliver (e.g., due to low computational power, malicious behavior or timeout), the slashing mechanism activates, penalizing the prover by forfeiting a part of their staked funds.
* **Order Fulfillment**: If the `proof` is valid, the prover receives the locked funds as a reward, and the requester receives the verified result, completing the process.




## Dependecies:
### Install & Update Packages
```bash
apt update && sudo apt upgrade -y
### Install & Update Packages
```bash
sudo apt install curl iptables build-essential git wget lz4 jq make gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev tar clang bsdmainutils ncdu unzip libleveldb-dev libclang-dev ninja-build  -y
```

### Clone Boundless Repo
```bash
git clone https://github.com/boundless-xyz/boundless
cd boundless
git checkout release-0.9
```

### Install Dependecies
To run a Boundless prover, you'll need the following dependencies:
* Docker compose
* GPU Drivers
* Docker Nvidia Support
* Rust programming language
* `Just` command runner
* CUDA Tollkit

For a quick set up of Boundless dependencies on Ubuntu 22.04 LTS, you can run:
```bash
sudo ./scripts/setup.sh
```
However, we need to install some dependecies manually:

```console
\\ Execute command lines one by one

# Install rustup:
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
. "$HOME/.cargo/env"

# Update rustup:
rustup update

# Install the Rust Toolchain:
apt update
apt install cargo

# Verify Cargo:
cargo --version

# Install rzup:
curl -L https://risczero.com/install | bash
source ~/.bashrc

# Verify rzup:
rzup --version

# Install RISC Zero Rust Toolchain:
rzup install rust

# Install cargo-risczero:
cargo install cargo-risczero
rzup install cargo-risczero

# Update rustup:
rustup update

# Install Bento-client:
cargo install --git https://github.com/risc0/risc0 --bin bento_cli
echo 'export PATH="$HOME/.cargo/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
bento_cli --version

# Install Boundless CLI:
cargo install --locked boundless-cli
export PATH=$PATH:/root/.cargo/bin
source ~/.bashrc

# Verify boundless-cli:
boundless-cli -h
```


compose.yml:
```yml
name: bento

# Anchors:
x-base-environment: &base-environment
  DATABASE_URL: postgresql://${POSTGRES_USER:-worker}:${POSTGRES_PASSWORD:-password}@${POSTGRES_HOST:-postgres}:${POSTGRES_PORT:-5432}/${POSTGRES_DB:-taskdb}
  REDIS_URL: redis://${REDIS_HOST:-redis}:6379
  S3_URL: http://${MINIO_HOST:-minio}:9000
  S3_BUCKET: ${MINIO_BUCKET:-workflow}
  S3_ACCESS_KEY: ${MINIO_ROOT_USER:-admin}
  S3_SECRET_KEY: ${MINIO_ROOT_PASS:-password}
  RUST_LOG: ${RUST_LOG:-info}
  RUST_BACKTRACE: 1

x-agent-common: &agent-common
  image: risczero/risc0-bento-agent:stable
  restart: always
  depends_on:
    - postgres
    - redis
    - minio
  environment:
    <<: *base-environment

services:
  redis:
    hostname: ${REDIS_HOST:-redis}
    image: ${REDIS_IMG:-redis:7.2.5-alpine3.19}
    restart: always
    ports:
      - 6379:6379
    volumes:
      - redis-data:/data

  postgres:
    hostname: ${POSTGRES_HOST:-postgres}
    image: ${POSTGRES_IMG:-postgres:16.3-bullseye}
    restart: always
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-taskdb}
      POSTGRES_USER: ${POSTGRES_USER:-worker}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-password}
    expose:
      - '${POSTGRES_PORT:-5432}'
    ports:
      - '${POSTGRES_PORT:-5432}:${POSTGRES_PORT:-5432}'
    volumes:
      - postgres-data:/var/lib/postgresql/data
    command: -p ${POSTGRES_PORT:-5432}

  minio:
    hostname: ${MINIO_HOST:-minio}
    image: ${MINIO_IMG:-minio/minio:RELEASE.2024-05-28T17-19-04Z}
    ports:
      - '9000:9000'
      - '9001:9001'
    volumes:
      - minio-data:/data
    command: server /data --console-address ":9001"
    environment:
      - MINIO_ROOT_USER=${MINIO_ROOT_USER:-admin}
      - MINIO_ROOT_PASSWORD=${MINIO_ROOT_PASS:-password}
      - MINIO_DEFAULT_BUCKETS=${MINIO_BUCKET:-workflow}
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 5s
      timeout: 5s
      retries: 5

  grafana:
    image: ${GRAFANA_IMG:-grafana/grafana:11.0.0}
    restart: unless-stopped
    ports:
     - '3000:3000'
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_LOG_LEVEL=WARN
      - POSTGRES_HOST=${POSTGRES_HOST:-postgres}
      - POSTGRES_DB=${POSTGRES_DB:-taskdb}
      - POSTGRES_PORT=${POSTGRES_PORT:-5432}
      - POSTGRES_USER=${POSTGRES_USER:-worker}
      - POSTGRES_PASS=${POSTGRES_PASSWORD:-password}
      - GF_INSTALL_PLUGINS=frser-sqlite-datasource
    volumes:
      - ./dockerfiles/grafana:/etc/grafana/provisioning/
      - grafana-data:/var/lib/grafana
      - broker-data:/db
    depends_on:
      - postgres
      - redis
      - minio

  exec_agent:
    <<: *agent-common
    runtime: nvidia
    mem_limit: 4G
    cpus: 4
    environment:
      <<: *base-environment
      RISC0_KECCAK_PO2: ${RISC0_KECCAK_PO2:-17}
      # JOIN_STREAM: 1
      # COPROC_STREAM: 1
    entrypoint: /app/agent -t exec --segment-po2 ${SEGMENT_SIZE:-21}

  aux_agent:
    <<: *agent-common
    runtime: nvidia
    mem_limit: 256M
    cpus: 1
    entrypoint: /app/agent -t aux --monitor-requeue

  gpu_prove_agent0:
    <<: *agent-common
    runtime: nvidia
    mem_limit: 4G
    cpus: 4
    entrypoint: /app/agent -t prove
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['0']
              capabilities: [gpu]

  gpu_prove_agent1:
    <<: *agent-common
    runtime: nvidia
    mem_limit: 4G
    cpus: 4
    entrypoint: /app/agent -t prove
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['1']
              capabilities: [gpu]

  gpu_prove_agent2:
    <<: *agent-common
    runtime: nvidia
    mem_limit: 4G
    cpus: 4
    entrypoint: /app/agent -t prove
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['2']
              capabilities: [gpu]

  gpu_prove_agent3:
    <<: *agent-common
    runtime: nvidia
    mem_limit: 4G
    cpus: 4
    entrypoint: /app/agent -t prove
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['3']
              capabilities: [gpu]

  snark_agent:
    <<: *agent-common
    runtime: nvidia
    entrypoint: /app/agent -t snark
    ulimits:
      stack: 90000000

  rest_api:
    image: risczero/risc0-bento-rest-api:stable
    restart: always
    depends_on:
      - postgres
      - minio
    mem_limit: 1G
    cpus: 1
    environment:
      <<: *base-environment
    ports:
      - '8081:8081'
    entrypoint: /app/rest_api --bind-addr 0.0.0.0:8081

  broker:
    restart: always
    depends_on:
      - rest_api
    profiles: [broker]
    build:
      context: .
      dockerfile: dockerfiles/broker.dockerfile
    mem_limit: 2G
    cpus: 2
    volumes:
      - type: bind
        source: ./broker.toml
        target: /app/broker.toml
      - broker-data:/db/
    network_mode: host
    environment:
      RUST_LOG: ${RUST_LOG:-info,broker=debug,boundless_market=debug}
      PRIVATE_KEY: ${PRIVATE_KEY}
      RPC_URL: ${RPC_URL}
      ORDER_STREAM_URL:
    entrypoint: /app/broker --db-url 'sqlite:///db/broker.db' --set-verifier-address ${SET_VERIFIER_ADDRESS} --boundless-market-address ${BOUNDLESS_MARKET_ADDRESS} --config-file /app/broker.toml --bento-api-url http://localhost:8081

volumes:
  redis-data:
  postgres-data:
  minio-data:
  grafana-data:
  broker-data:
```


Configure Network
* Sepolia:
```bash
nano .env.testnet
```
Add the following variables to the `.env.testnet`.
* `RPC_URL=""`: To get Ethereum Sepolia rpc url, Use third-parties .e.g Alchemy or run your own [Geth Node](https://github.com/0xmoei/geth-prysm-node/)
* RPC has to be between `""`
* `PRIVATE_KEY=`: Add your EVM wallet private key

![image](https://github.com/user-attachments/assets/3b41c3b7-8f79-4067-9117-41ac68b41946)

```bash
source <(just env testnet)
```

Deposit
```
source ~/.bashrc
```
Deposit ETH
```
boundless account deposit ETH_AMOUNT
```
* Deposit Balance: `boundless account balance`

Deposit Stake
```
boundless account deposit-stake STAKE_AMOUNT
```
* Deposit Stake Balance: `boundless account stake-balance`


