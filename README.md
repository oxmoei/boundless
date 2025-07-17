# Boundless Prover Guide
Boundless Prover node is a computational proof system participating in the Boundless decentralized proof market. Provers need to stake USDC, bid for computational tasks, use GPU acceleration to generate zero-knowledge proofs, and receive rewards upon successful proof generation.

This guide covers the **automated** installation method on Ubuntu 20.04/22.04 systems.

## Table of Contents
- [Official Video Tutorial](https://youtu.be/MZqU-J-fW2M)
- [Boundless Prover Market](#boundless-prover-market)
- [Notes](#notes)
- [Hardware and Software Requirements](#hardware-and-software-requirements)
- [Automated Installation](#automated-installation)
- [Bento (Prover) and Broker Optimization](#bento-prover--broker-optimization)
  - [Segment Size (Prover)](#segment-sizeprover)
  - [Bento Benchmark](#bento-benchmark)
  - [Broker Optimization](#broker-optimization)
  - [Multiple Brokers](#multiple-brokers)
- [Safely Update or Stop Prover](#safely-update-or-stop-prover)
- [Debugging](#debugging)

---

## Boundless Prover Market
First, you need to understand how the **Boundless Prover Market** actually works so you know what you are doing.

1. **Request Submission**: Developers submit computational tasks (orders) on Boundless and provide ETH/ERC-20 rewards
2. **Prover Stakes USDC**: Provers must stake `USDC` before bidding on orders
3. **Bidding Process**: Provers discover orders and submit competitive bids (`mcycle_price`)
4. **Order Locking**: The winning prover uses the staked USDC to lock the order, committing to complete the proof before the deadline
5. **Proof Generation**: Prover uses GPU acceleration to compute and submit the proof
6. **Rewards/Penalties**: Valid proofs receive rewards; invalid/late proofs will result in forfeiture of the stake

---

## Notes
- The Prover is currently in the testing phase. Although I believe this guide is very comprehensive, you may still encounter some issues during operation. You can wait until the official incentivized testnet is more stable and the guide is updated before participating, or you can start trying now.
- It is recommended to start with the testnet to avoid loss of staked funds.
- I will continuously update this GitHub guide, so please check back regularly.

---

## Hardware and Software Requirements
### Hardware
* CPU - 16 threads, good single-core turbo performance (>3Ghz)
* Memory - 32 GB
* Storage - 100 GB NVME/SSD
* GPU
  * Minimum: One GPU with 8GB VRAM
  * Recommended: 10 GPUs with 8GB VRAM each
  * Recommended GPU models: 4090, 5090, and L4.
> * You can start testing with a single card and adjust later according to your setup. See details below.

### System
* Supported: Ubuntu 20.04/22.04
* Not supported: Ubuntu 24.04
* If running locally on Windows, use this [guide](https://github.com/0xmoei/Install-Linux-on-Windows) to install Ubuntu 22 WSL

---

# Automated Installation
For automated installation and Prover management, you can use this script to automatically handle all dependencies, configuration, installation, and Prover management.

## Download and Run the Installation Script:
```bash
# Clone the repository
git clone https://github.com/oxmoei/boundless.git && cd boundless

# Run the installation script
chmod +x install_prover.sh && sudo ./install_prover.sh
```

* The installation process may take a while as it needs to install drivers and build large files. Please be patient.

### During Installation:
* The script will automatically detect your GPU configuration
* You will be prompted to enter:
  * Network selection (Mainnet/Testnet)
  * RPC URL: See [Get RPC](#get-rpc)
  * Private key (input is hidden)
  * Broker configuration parameters: See [Broker Optimization](#broker-optimization) for details


### Post-Installation Management Script:
After installation, to run or configure the Prover, enter the installation directory and run the management script `prover.sh`:

```bash
cd ~/boundless
./prover.sh
```
The management script menu includes:
- **Service Management**: Start/stop broker, view logs, health check
- **Configuration Management**: [Switch network](#get-rpc), update private key, edit [broker configuration](#broker-optimization)
- **Staking Management**: Stake USDC, check balance
- **Performance Testing**: Run benchmark with order ID
- **Monitoring**: Real-time GPU monitoring


### Modify CPU/RAM for x-exec-agent-common & gpu-prove-agent
The `prover.sh` script manages all broker configurations (such as `broker.toml`), but if you need to optimize and increase RAM and CPU for `compose.yml`, refer to the [x-exec-agent-common](#modify-gpu_prove_agent-cpu-ram) and [gpu-prove-agent](#modify-gpu_prove_agent-cpu-ram) sections.
* After modifying `compose.yml`, restart the broker.

---

# Bento (Prover) and Broker Optimization
There are many optimization factors to improve your winning rate in the prover competition. See the [official broker guide](https://docs.beboundless.xyz/provers/broker) or [prover guide](https://docs.beboundless.xyz/provers/performance-optimization) for details.

## Improve Preflight Efficiency
* The `exec_agent` service in `compose.yml` is responsible for preflight of orders to evaluate whether the prover can bid.
* The more concurrent preflights, the faster you can lock orders and the more competitive you are.
  * Increase the number of `exec_agent` to preflight more orders concurrently.
  * Increasing CPU/RAM for a single `exec_agent` can improve preflight speed.
* The default value is `2`, adjust as needed.
* Related services: `x-exec-agent-common` and `exec_agent`
  * `x-exec-agent-common`: Main settings for all `exec_agent` services, including CPU and memory
  * `exec_agentX`: Specific agents, X is the number. To add more agents, just increment the number.

Example of `x-exec-agent-common`:
```yml
x-exec-agent-common: &exec-agent-common
  <<: *agent-common
  mem_limit: 4G
  cpus: 2
  environment:
    <<: *base-environment
    RISC0_KECCAK_PO2: ${RISC0_KECCAK_PO2:-17}
  entrypoint: /app/agent -t exec --segment-po2 ${SEGMENT_SIZE:-21}
```
* You can increase `cpus` and `mem_limit`

Example of `exec_agent`:
```yaml
  exec_agent0:
    <<: *exec-agent-common

  exec_agent1:
    <<: *exec-agent-common
```
* To add more agents, just add more lines with incremented numbers

## Improve GPU Proof Efficiency
* The `gpu_prove_agent` service in `compose.yml` is responsible for using the GPU to prove orders.
* For single GPU setups, you can improve performance by increasing CPU/RAM for each GPU agent.
* The default CPU/RAM is sufficient, but you can increase it if you have better hardware.
* You will see the following `gpu_prove_agentX` service configuration, where you can increase memory and CPU for each GPU agent.
   ```yml
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
   ```
* Although the default CPU/RAM is sufficient, you can increase it for single GPU setups, but do not use all resources—leave some for other tasks.

## Bento Benchmark
Install psql:
```bash
apt update
apt install postgresql-client
psql --version
```

**1. Recommended: Simulate benchmark with order ID (make sure Bento is running):**
```bash
boundless proving benchmark --request-ids <IDS>
```
* Get order IDs from [here](https://explorer.beboundless.xyz/orders)
* Separate multiple IDs with commas
* It is recommended to select orders of different sizes and programs, favoring large orders for more representative benchmarks

* As shown below, the prover estimates it can handle about 430,000 cycles/s (about 430 kHz).
* Set a value slightly lower than the recommended value in `peak_prove_khz` in `broker.toml` (explained later)

> You can use the `nvtop` command in a new terminal to monitor GPU usage

**2. Benchmark with Harness Test**
* You can also use ITERATION_COUNT to benchmark the GPU:
```
RUST_LOG=info bento_cli -c <ITERATION_COUNT>
```
`<ITERATION_COUNT>` is the number of synthetic task executions. Start with `4096`, use `2048` or `1024` for lower performance, and `32` for functional testing.

* Check the `khz` and `cycles` of the harness test
```
bash scripts/job_status.sh JOB_ID
```
* `JOB_ID` is the number prompted during the test.
* The obtained `hz` divided by 1000 is `khz`, and the number of cycles proved.
* If you get a `not_found` error, it means `.env.broker` was not created. The script uses the `SEGMENT_SIZE` in `.env.broker` to query the segment size. You can fix it with `cp .env.broker-template .env.broker`.

---

## Broker Optimization

* The broker is a container of the prover, responsible for on-chain interaction, order locking, setting staking bids, etc.
* The `broker.toml` configures how the broker interacts on-chain and competes with other provers.

Copy the template as the main configuration file:
```bash
cp broker-template.toml broker.toml
```

Edit broker.toml:
```bash
nano broker.toml
```
* Official `broker.toml` example is [here](https://github.com/boundless-xyz/boundless/blob/main/broker-template.toml)

### Increase Lock Rate
After the broker is running, before GPU proving, it needs to compete with other provers to lock orders. Optimization methods:

1. Lower `mcycle_price` to let the broker bid at a lower price.
* After an order is detected, the broker preflights to estimate the required `cycles`. As shown, the prover proves orders of millions/thousands of cycles.
* `mcycle_price` is the price per million cycles. Final price = `mcycle_price` x `cycles`
* Setting a lower `mcycle_price` increases the chance of winning the bid.

* You can check other provers' `mcycle_price` on the [explorer](https://explorer.beboundless.xyz/orders/0xc2db89b2bd434ceac6c74fbc0b2ad3a280e66db024d22ad3) in the order details page under `ETH per Megacycle`

2. Increase `lockin_priority_gas` to spend more gas to bid first. Remove the `#` comment and set the gas in Gwei.

### Other `broker.toml` Settings
See [official documentation](https://docs.beboundless.xyz/provers/broker#settings-in-brokertoml)
* `peak_prove_khz`: Maximum cycles per second (kHz) for the proof backend.
  * Set according to the [Bento Benchmark](https://github.com/0xmoei/boundless/tree/main#benchmarking-bento) above

* `max_concurrent_proofs`: Number of orders that can be locked simultaneously. Increasing this value allows more orders to be locked, but if you cannot complete the proofs before the deadline, your stake will be forfeited.
  * When the limit is reached, the system will pause new orders and wait for existing proofs to complete.
  * Default is `2`, depending on GPU and configuration. Test and adjust accordingly.

* `min_deadline`: Minimum remaining seconds for an order when bidding.
  * Orders have a deadline. If the prover cannot complete the proof before this time, the stake will be forfeited.
  * After setting the minimum deadline, the prover will not accept orders with less than this time remaining.
  * As shown, if the order is completed after the deadline, the prover is penalized for being late.

---

## Multiple Brokers
You can run multiple brokers with a single Bento client to generate proofs on different networks.
* Your configuration may differ from mine. You can ask AI chat for help. Here is my configuration example:
* Files to modify: `compose.yml`, `broker.toml`, `.env` files (such as `.env.base-sepolia`)

### Modify `compose.yml`

**Step 1: Add `broker2` service**:

In the services section, add `broker2` after the existing `broker` service. Its configuration is similar to the original `broker`, but uses a different database and config file.
* Changes needed:
 * Name changed to `broker2`
 * `source: ./broker2.toml`
 * `broker2-data:/db/`
 * `--db-url` changed to `'sqlite:///db/broker2.db'`

**Step 2: Multiple broker environment variables (.env files):**

Originally, the `.env` file (such as `.env.base`) was used to set the network. Now, you need to specify the corresponding `.env` file for each broker (such as `broker`, `broker1`, `broker3`) in `compose.yml`.
* After the `volumes` section of each broker service, add:
```
    env_file:
      - .env.base
```

**Step 3: Add `broker2-data` volume**:
* At the end of `compose.yml`, add a new volume in the `volumes` section:

For example, the configuration for `broker` and `broker2` services supporting Base and ETH Sepolia networks is as follows:

```yaml
  broker:
    restart: always
    depends_on:
      - rest_api
      - gpu_prove_agent0
      - exec_agent0
      - exec_agent1
      - aux_agent
      - snark_agent
      - redis
      - postgres
    profiles: [broker]
    build:
      context: .
      dockerfile: dockerfiles/broker.dockerfile
    mem_limit: 2G
    cpus: 2
    stop_grace_period: 3h
    volumes:
      - type: bind
        source: ./broker.toml
        target: /app/broker.toml
      - broker-data:/db/
    network_mode: host
    env_file:
      - .env.base
    environment:
      RUST_LOG: ${RUST_LOG:-info,broker=debug,boundless_market=debug}
    entrypoint: /app/broker --db-url 'sqlite:///db/broker.db' --set-verifier-address ${SET_VERIFIER_ADDRESS} --boundless-market-address ${BOUNDLESS_MARKET_ADDRESS} --config-file /app/broker.toml --bento-api-url http://localhost:8081
    ulimits:
      nofile:
        soft: 65535
        hard: 65535

  broker2:
    restart: always
    depends_on:
      - rest_api
      - gpu_prove_agent0
      - exec_agent0
      - exec_agent1
      - aux_agent
      - snark_agent
      - redis
      - postgres
    profiles: [broker]
    build:
      context: .
      dockerfile: dockerfiles/broker.dockerfile
    mem_limit: 2G
    cpus: 2
    stop_grace_period: 3h
    volumes:
      - type: bind
        source: ./broker2.toml
        target: /app/broker.toml
      - broker2-data:/db/
    network_mode: host
    env_file:
      - .env.eth-sepolia
    environment:
      RUST_LOG: ${RUST_LOG:-info,broker=debug,boundless_market=debug}
    entrypoint: /app/broker --db-url 'sqlite:///db/broker2.db' --set-verifier-address ${SET_VERIFIER_ADDRESS} --boundless-market-address ${BOUNDLESS_MARKET_ADDRESS} --config-file /app/broker.toml --bento-api-url http://localhost:8081
    ulimits:
      nofile:
        soft: 65535
        hard: 65535

volumes:
  redis-data:
  postgres-data:
  minio-data:
  grafana-data:
  broker-data:
  broker2-data:
```

### Modify `broker.toml`
Each broker instance needs a separate `broker.toml` file (such as `broker.toml`, `broker2.toml`, etc.)

To create a new config file for the second broker:
```bash
# Copy from existing broker config
cp broker.toml broker2.toml
 
# Or create from template
cp broker-template.toml broker2.toml
```
Then modify the configuration for each network. Note:

* `peak_prove_khz` is shared among all brokers.
  * For example, if the benchmark is `500kHz`, the total sum in all configs must not exceed `500kHz`.
  * Example: `broker.toml`: `peak_prove_khz = 250`, `broker2.toml`: `peak_prove_khz = 250`

* `max_concurrent_preflights` limits the number of pricing tasks a broker can run concurrently. The total for all brokers must not exceed the number of `exec_agent` services in `compose.yml`.
  * If you have two `exec_agent` (e.g., `exec_agent0` and `exec_agent1`), the total `max_concurrent_preflights` for all brokers must not exceed 2.

* `max_concurrent_proofs`
  * Unlike `peak_prove_khz`, `max_concurrent_proofs` is set independently for each broker and controls the number of proof tasks a single broker can handle simultaneously.
  * If you have only one GPU, you can usually only handle one proof at a time. Set `max_concurrent_proofs = 1`.

 * `lockin_priority_gas`: Set an appropriate gwei for each network

---

# Safely Update or Stop Prover
### 1. Check Locked Orders
Check via `broker` logs or the [prover's indexer page](https://explorer.beboundless.xyz/provers/) to ensure the broker has no unfinished locked orders. Otherwise, you may be penalized when stopping or updating.

* To temporarily stop accepting new orders, set `max_concurrent_proofs` to `0`. After all locked orders are completed, stop the node.

### 2. Stop broker and optionally clean database
```bash
# Optional, skip if not upgrading the node repo
just broker clean
 
# Or just stop the broker without cleaning the data volume
just broker down
```

### 3. Update to new version
Latest tag is in [releases](https://github.com/boundless-xyz/boundless/releases)
```bash
git checkout <new_version_tag>
# Example: git checkout v0.10.0
```

### 4. Start new version broker
```bash
just broker
```

---

### Network Configuration Method 2: .env File
**Recommended** to use Method 1, skip this step and go directly to [Stake USDC](#stake-usdc). For Method 2, see below.

* The official configuration has three `.env` files for each network (`.env.base`, `.env.base-sepolia`, `.env.eth-sepolia`).

### Base Mainnet
* Here, `.env.base` is used as an example. For other networks, modify the corresponding file.
* Currently, Base mainnet order demand is low. You can participate in Base Sepolia or ETH Sepolia by modifying `.env.base-sepolia` or `.env.eth-sepolia`.

* Configure the `.env.base` file:
```bash
nano .env.base
```
Add the following variables:
* `export RPC_URL=""`:
  * The RPC address must be in double quotes
* `export PRIVATE_KEY=`: Fill in your EVM wallet private key

* Inject `.env.base` into the prover:
```bash
source .env.base
```
* Each time you close the terminal or start the prover, you need to re-inject the network configuration.

### Optional: Custom Environment `.env.broker`
`.env.broker` is similar to the above `.env` files but allows more options. When using, refer to the [deployment page](https://docs.beboundless.xyz/developers/smart-contracts/deployments) to replace contract addresses for each network.
* Not recommended, as switching networks is easier by changing the above `.env` files directly.

* Create `.env.broker`:
```bash
cp .env.broker-template .env.broker
```

* Configure the `.env.broker` file:
```bash
nano .env.broker
```
Add the following variables:
* `export RPC_URL=""`: Base network rpc url, recommended to use third-party services like Alchemy
  * The RPC address must be in double quotes
* `export PRIVATE_KEY=`: Fill in your EVM wallet private key
* Other variables see [here](https://docs.beboundless.xyz/developers/smart-contracts/deployments):
  * `export BOUNDLESS_MARKET_ADDRESS=`
  * `export SET_VERIFIER_ADDRESS=`
  * `export VERIFIER_ADDRESS=` (add manually)
  * `export ORDER_STREAM_URL=`
 
* Inject `.env.broker` into the prover:
```
source .env.broker
```
  * After closing the terminal, you need to re-inject the network configuration.

---

# Debugging
## Error: Too many open files (os error 24)
You may encounter the `Too many open files (os error 24)` error during `just broker` build.

### Solution:
```
nano /etc/security/limits.conf
```
* Add:
```
* soft nofile 65535
* hard nofile 65535
```

```
nano /lib/systemd/system/docker.service
```
* In the `[Service]` section, add or modify:
```
LimitNOFILE=65535
```

```
systemctl daemon-reload
systemctl restart docker
```

* Restart the terminal, re-inject the network configuration, and run `just broker` again


## Many `Locked` Orders on Prover [explorer](https://explorer.beboundless.xyz/)
* Mostly due to RPC issues, please check the logs.
* In the `broker.toml` file, set `txn_timeout = 45` to increase the transaction confirmation timeout.



