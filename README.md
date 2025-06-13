# Boundless Prover Guide

## Boundless Prover market
First, you need to know how **Boundless Prover market** actually works to realize what you are doing.
* **Requester Submits Ask**: A requester (e.g. developer) creates a task or a computation request as an `order` on Boundless, offering funds in ETH or ERC-20 to incentivize participation.
* **Prover Stakes USDC**: The Boundless market requires funds (USDC) deposited as stake before a prover can `bid` on requests.
* **Prover Places Bid**: A prover detects an `order`, submits a `bid`, stating their offered price or resources, which may be lower than the request’s locked funds or other provers’ `bid`s.
* **Prover Locks Order**: If their `bid` is accepted among other provers (e.g., lower `bid`, sufficient stake, or meeting specific criteria), the prover locks the order committing to prove it within a set deadline (`lock-timeout`) using previously staked `USDC`, so other provers can't touch it until it perform computational power.
* **Prover Generates Proof**: The prover completes the task and submits the `proof` to the network.
* **Slashing**: If the `proof` is invalid, incomplete, or the prover fails to deliver (e.g., due to low computational power, malicious behavior or timeout), the slashing mechanism activates, penalizing the prover by forfeiting a part of their staked `USDC` funds.
* **Order Fulfillment**: If the `proof` is valid, the prover receives the locked funds as a reward, and the requester receives the verified result, completing the process.
* `bid ` are actually `mcycle_price` (the price of each 1 million cycles the prover proves). I'll tell you more about this later in the guide.

---

## Requirements
### Hardware
* CPU - 16 threads, reasonable single core boost performance (>3Ghz)
* Memory - 32 GB
* Disk - 100 GB NVME/SSD
* GPU
  * Minimum: one 8GB vRAM GPU
  * Recommended to be competetive: 10x GPU with min 8GB vRAM
  * Recomended GPU models are 4090, 5090 and L4.

### Software
* Supported: Ubuntu 20.04/22.04
* Experimental: Ubuntu 24.04
* If you are running on Windows os locally, install Ubuntu 22 WSL using this [Guide](https://github.com/0xmoei/Install-Linux-on-Windows)

---

## Rent GPU
* Beginners in renting GPU can use this [guide](https://github.com/0xmoei/Rent-and-Config-GPU)
* You have to rent a `Ubuntu VM` template (and NOT `CUDA` or `Pytorch`) GPU
* As my research, [Vast.ai](https://cloud.vast.ai/?ref_id=228875) was the only cheap GPU provider supporting `Ubuntu VM` templates with crypto payments.
* Prover installation is using `Docker`, so `CUDA` or `Pytorch` templates for cloud GPUs is not possible because they also run your GPU instance in a Docker and you can't run Prover Docker inside your GPU instance Docker.

---
# Setup
Here is the step by step guide to Install and run your Prover smoothly, but please pay attention to these notes:
* Read every single word of this guide, if you really want to know what you are doing.
* There is an #Optimize Prover section where you need to ready after Setup.

## Dependecies
### Install & Update Packages
```bash
apt update && sudo apt upgrade -y
apt install curl iptables build-essential git wget lz4 jq make gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev tar clang bsdmainutils ncdu unzip libleveldb-dev libclang-dev ninja-build -y
```

### Clone Boundless Repo
```bash
git clone https://github.com/boundless-xyz/boundless
cd boundless
git checkout release-0.10
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
bash ./scripts/setup.sh
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
cargo install --git https://github.com/risc0/risc0 bento-client --bin bento_cli
echo 'export PATH="$HOME/.cargo/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Verify Bento-client:
bento_cli --version

# Install Boundless CLI:
cargo install --locked boundless-cli
export PATH=$PATH:/root/.cargo/bin
source ~/.bashrc

# Verify boundless-cli:
boundless -h
```

---

## System Hardware Check
* In the beginning, to configure your Prover, You need to know what's your GPUs IDs (if multiple GPUs), CPU cores and RAM.
* Also the following tools are best to monitor your hardware during proving.

### GPU Check:
* If your Nvidia driver and Cuda tools are installed succesfully, run the following command to see your GPUs status:
```
nvidia-smi
```
* You can now monitor Nvidia driver & Cuda Version, GPU utilization & memory usage.
* In the image below, there are four GPUs with **0-3** IDs, you'll need it when adding GPU to your configuration.

![image](https://github.com/user-attachments/assets/26c57f43-0fbf-4068-949c-b2ea31273998)

* Check your system GPUs IDs (e.g. 0 through X):
```bash
nvidia-smi -L
```

### CPU & RAM check (Realtime):
To see the status of your CPU and RAM.
```bash
htop
```

### GPU Check (Realtime):
The best for real-time monitoring your GPUs in a seprated terminal while your prover is proving.
```bash
nvtop
```

---

## Configure Prover
### Single GPU:
The `compose.yml` file defines all services within Prover.
* Default `compose.yml` only supporting single-GPU and default CPU, RAM utilization.
* Edit `compose.yml` by this command:
  ```bash
  nano compose.yml
  ```
* The current `compose.yml` is set for `1` GPU by default.

### Multiple GPUs
* 4 GPUs:
To add more GPUs or modify CPU and RAM sepcified to each GPU (default numbs are fine), replace the current compose file with my [custom compose.yml](https://github.com/0xmoei/boundless/blob/main/compose.yml) with 4 custom GPUs

* More/Less than 4 GPUs
 Follow this [detailes step by step guide](https://github.com/0xmoei/boundless/blob/main/add-remove-gpu.md) to add or remove the number of 4 GPUs in my custom compose.yml file

### Modify CPU/RAM of each GPU
* You see that on `gpu_prove_agent0`, GPU 0 is listed with the device ID of `0`, memory limited to `4G`, and CPU set as `4` cores for each GPU instance:
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
* While the default CPU/RAM for each GPU is enough, for single GPUs, you can increase them to increase efiiciency, but don't maximize and always keep some CPU/RAML for other jobs.

**Note**: `SEGMENT_SIZE` of the prover is set to `21` by default for each GPU, which is compatible only with `>20GB vRAM` GPUs, if you have less vRAM, you have to modify it, you can read the Segment Size section of the guide.

---

## Running Prover
Boundless is comprised of two major components:
* `Bento` is the local proving infrastructure. Bento will take requests, prove them and return the result.
* `Broker` interacts with the Boundless market. `Broker` can submit or request proves from the market.

---

## Run Bento
To get started with a test proof on a new proving machine, let's run `Bento` to benchmark our GPUs:
```bash
just bento
```
* This will spin up `bento` without the `broker`.

Check the logs :
```bash
just bento logs
```

Run a test proof:
```bash
RUST_LOG=info bento_cli -c 32
```
* If everything works well, you should see something like the following as `Job Done!`:

![image](https://github.com/user-attachments/assets/a67fdfb0-3d22-4a4a-b47a-247567df0d45)

* To check if all your GPUs are utilizing:
  *  Increase `32` to `2048`/`4096`
  *  Open new terminal with `nvtop` command
  *  Run the test proof and monitor your GPUs utilization.

---

## Configure Network
Boundless is available on `Base Mainnet`, `Base Sepolia` and `Ethereum Sepolia`, marking the first time real funds will be used in Boundless.


Provers should ensure they configure their Proving Nodes to reduce the risk of accepting unprofitable requests or being slashed due to overcommitting proving resources. This typically involves configuring `mcycle_price`, `mcycle_price_stake_token`, `peak_prove_khz`, `max_concurrent_proofs` and other configuration values according to your pricing preferences and cluster size. Refer to broker-

### Set Networks
There are three `.env` files with the official configurations of each network (`.env.base`, `.env.base-sepolia`, `.env.eth-sepolia`).

### Base Mainnet
In this step I modify `.env.base`, you can replace any of above with it.

* Configure `.env.base` file:
```bash
nano .env.base
```
Add the following variables to the `.env.base`.
* `RPC_URL=""`: To get Base Mainnet rpc url, Use third-parties .e.g free [Alchemy](https://dashboard.alchemy.com/apps), [Quicknode](https://quicknode.com/signup?via=moei) or paid [Ankr](https://www.ankr.com/rpc/?utm_referral=LqL9Sv86Te). haven't tested third-party platforms myself yet, they might have some issue, so we might need to run our own node. I'll update this section soon.
* RPC has to be between `""`
* `PRIVATE_KEY=`: Add your EVM wallet private key

![image](https://github.com/user-attachments/assets/3b41c3b7-8f79-4067-9117-41ac68b41946)

* Inject `.env.base` changes to prover:
```bash
source .env.base
```
* After each terminal close, you have to run this to inject the network before running `broker` or doing `Deposit` commands (both in next steps).

### Optional: `.env.broker` with custom enviroment
`.env.broker` is a custom environment file same as previous `.env` files but with more options to configure, you can also use it but you have to refer to [Deployments](https://docs.beboundless.xyz/developers/smart-contracts/deployments) page to replace contract addresses of each network.

* Create `.env.broker`:
```bash
cp .env.broker-template .env.broker
```

* Configure `.env.broker` file:
```bash
nano .env.broker
```
Add the following variables to the `.env.broker`.
* `RPC_URL=""`: To get Base network rpc url, Use third-parties .e.g Alchemy or paid ones.
  * RPC has to be between `""`
* `PRIVATE_KEY=`: Add your EVM wallet private key
* Find the value of following variables [here](https://docs.beboundless.xyz/developers/smart-contracts/deployments):
  * `BOUNDLESS_MARKET_ADDRESS=`
  * `SET_VERIFIER_ADDRESS=`
  * `VERIFIER_ADDRESS=` (add it to .env manually)
  * `ORDER_STREAM_URL=`
 
* Inject `.env.broker` changes to prover:
```
source .env.broker
```
  * After each terminal close, you have to run this to inject the network before running `broker` or doing `Deposit` commands (both in next steps).

---

## Deposit Stake
During this phase `USDC` will be used as the staking token across all networks. HP tokens are now deprecated. Provers will need to deposit` USDC` to the Boundless Market contract to use as stake when locking orders.

Note that `USDC` has a different address on each network. Refer to the [Deployments page](https://docs.beboundless.xyz/developers/smart-contracts/deployments) for the addresses. USDC can be obtained on testnets from the [Circle Faucet](https://faucet.circle.com/).

**Add `boundless` CLI to bash:**
```
source ~/.bashrc
```

**Deposit ETH:**
```
boundless account deposit ETH_AMOUNT
```
* Deposit Balance: `boundless account balance`

**Deposit Stake:**
```
boundless account deposit-stake STAKE_AMOUNT
```
* Deposit Stake Balance: `boundless account stake-balance`

---

##  Run Broker
You can now start `broker` (which runs both `bento` + `broker` i.e. the full proving stack!):
```bash
just broker
```
Check the total proving logs:
```bash
just broker logs
```
Check the `broker` logs, which has the most important logs of your `order` lockings and fulfillments:
```
docker compose logs -f broker

# For last 100 logs
docker compose logs -fn 100
```

---

# Bento (Prover) & Broker Optimizations
There are many factors to be optimized to win in provers competetion where you can read the official guide for [broker](https://docs.beboundless.xyz/provers/broker) or [prover](https://docs.beboundless.xyz/provers/performance-optimization)

Here I simplified everything with detailed steps:

## Segment Size (Prover)
Larger segment sizes more proving (bento) performance, but require more GPU VRAM. To pick the right `SEGMENT_SIZE` for your GPU VRAM, see the [official performance optimization page](https://docs.beboundless.xyz/provers/performance-optimization#finding-the-maximum-segment_size-for-gpu-vram).

![image](https://github.com/user-attachments/assets/ef566e27-ce69-4563-a035-87733827126d)

### Setting SEGMENT_SIZE
* `SEGMENT_SIZE` in `compose.yml` under the `exec_agent` service is `21`by default.
* Also you can change the value of `SEGMENT_SIZE` in `.env.broker` before running the prover.
* Note, when you set a number for `SEGMENT_SIZE` in env or default yml files, it sets that number for each GPU identically.


### Benchmarking Bento
Install psql:
```bash
sudo apt update
apt install postgresql postgresql-client
psql --version
```

Benchmark by simulating an order id: (make shure Bento is running)
```bash
boundless-cli proving benchmark --request-ids <IDS>
```
* You can use the order IDs listed [here](https://explorer.beboundless.xyz/orders)
* You can add multiples by adding comma-seprated ones.
* Recommended to pick a few requests of varying sizes and programs, biased towards larger proofs for a more representative benchmark.

![image](https://github.com/user-attachments/assets/04ca61f7-a658-4cb8-b09b-928bbe4694d4)

* As in the image above, the prover is estimated to handle ~430,000 cycles per second (~430 khz). 
* Use a lower amount of the recommented `peak_prove_khz` in your `broker.toml` (I explain it more in the next step)

---

## Broker Optimization
* Broker is a container of the whole prover, it's not proving itself, it's for onchain activities, and initializing with orders like locking orders or setting amount of stake bids, etc.
* `broker.toml` has the settings to configure how your broker interact on-chain and compete with other provers.
```bash
nano broker.toml
```
* You can see an example of the official `broker.toml` [here](https://github.com/boundless-xyz/boundless/blob/main/broker-template.toml)

### Increasing Lock-in Rate
Once your broker is running, before the gpu-based prover gets into work, broker must compete with other provers to lock-in the orders. Here is how to optimize broker to lock-in faster than other provers:

1. Decreasing the `mcycle_price` would tune your Broker to `bid` at lower prices for proofs.
* Once an order detected, the broker runs a preflight execution to estimate how many `cycles` the request needs. As you see in the image, a prover proved orders with millions or thousands of cycles.
* `mcycle_price` is actually price of a prover for proving each 1 million cycles. Final price = `mcycle_price` x `cycles`
* The less you set `mcycle_price`, the higher chance you outpace other provers.

![image](https://github.com/user-attachments/assets/fab9cc79-362f-4a43-a461-258ffe0bfc1a)

2. Increasing `lockin_priority_gas` to consume more gas to outrun other bidders. You might need to first remove `#` to uncomment it's line, then set the gas. It's based on Gwei.

### Other settings in `broker.toml`
Read the more about them in [official doc](https://docs.beboundless.xyz/provers/broker#settings-in-brokertoml)
* `peak_prove_khz`: Maximum number of cycles per second (in kHz) your proving backend can operate.
  * You can set the `peak_prove_khz` by following the previous step (Benchmarking Bento)

---

# Safe Update or Stop Prover
### 1. Check locked orders
Ensure either through the `broker` logs or [through indexer page of your prover](https://explorer.beboundless.xyz/provers/) that your broker does not have any incomplete locked orders before stopping or update, othervise you might get slashed for your staked assets.

### 2. Stop the broker and optionally clean the database
```bash
just broker clean
 
# Or stop the broker without cleaning volumes
just broker down
```

### 3. Update to the new version
See [releases](https://github.com/boundless-xyz/boundless/releases) for latest tag to use.
```bash
git checkout <new_version_tag>
# Example: git checkout v0.10.0
```

### 4. Start the broker with the new version
```bash
just broker
```



