# Boundless Prover Guide

## Boundless Prover market
First, you need to know how **Boundless Prover market** actually works to realize what you are doing.
* **Requester Submits Ask**: A requester (e.g. developer) creates a task or computation `order` and submits an `ask` on Boundless, locking funds to incentivize participation.
* **Prover Stakes USDC**: The Boundless market requires funds (USDC) deposited as stake before a prover can `bid` on requests.
* **Prover Places Bid**: A prover selects an `order`, submits a `bid`, stating their offered price or resources, which may be lower than the `ask`’s locked funds or other provers’ `bid`s.
* **Prover Locks Order**: If their `bid` is accepted among other provers (e.g., lower `bid`, sufficient stake, or meeting specific criteria), the prover locks the `order`, committing to perform the computational work.
* **Prover Generates Proof**: The prover completes the task and submits the `proof` to the network.
* **Verifier Checks Proof**: Verifiers validate the `proof` to ensure it meets the `orders`’s requirements and protocol standards.
* **Slashing**: If the `proof` is invalid, incomplete, or the prover fails to deliver (e.g., due to low computational power, malicious behavior or timeout), the slashing mechanism activates, penalizing the prover by forfeiting a part of their staked funds.
* **Order Fulfillment**: If the `proof` is valid, the prover receives the locked funds as a reward, and the requester receives the verified result, completing the process.




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
boundless-cli -h
```

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
```
htop
```

### GPU Check (Realtime):
The best for real-time monitoring your GPUs in a seprated terminal while your prover is proving.
```
nvtop
```

## Configure Prover
The `compose.yml` file defines all services within Prover.
* Default `compose.yml` only supporting single-GPU and default CPU, RAM utilization.
* Edit `compose.yml` by this command:
  ```bash
  nano compose.yml
  ```

To add GPUs or modify CPU,RAM to maximize utilization, replace the current compose file with my [custom compose.yml](https://github.com/0xmoei/boundless/blob/main/compose.yml) with 4 custom GPUs
* You see that on `gpu_prove_agent0`, then GPU 0 is listed with the device ID of `0`, memory limited to `4G`, and CPU set as `4` cores for each GPU instance:
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
* You can modify them based on your hardware but don't maximize and keep always keep some for other jobs.
* You can add/remove `gpu_prove_agentX` for more or less than 4 GPUs



## Running Prover
Boundless is comprised of two major components:
* `Bento` is the local proving infrastructure. Bento will take requests, prove them and return the result.
* `Broker` interacts with the Boundless market. `Broker` can submit or request proves from the market.

## Run Bento
To get started with a test proof on a new proving machine, let's run `Bento` to benchmark our GPUs:
```
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


## Configure Network
### Sepolia:
Configure `.env.testnet` file:
```bash
nano .env.testnet
```
Add the following variables to the `.env.testnet`.
* `RPC_URL=""`: To get Ethereum Sepolia rpc url, Use third-parties .e.g Alchemy or run your own [Geth Node](https://github.com/0xmoei/geth-prysm-node/)
* RPC has to be between `""`
* `PRIVATE_KEY=`: Add your EVM wallet private key

![image](https://github.com/user-attachments/assets/3b41c3b7-8f79-4067-9117-41ac68b41946)

Inject changes:
```bash
source <(just env testnet)
```
* After each terminal close, you have to run this to inject the network before running `broker` or doing `Deposit` commands (both in next steps).

## Deposit Stake
```
source ~/.bashrc
```

**Deposit ETH:**
```
boundless-cli account deposit ETH_AMOUNT
```
* Deposit Balance: `boundless-cli account balance`

**Deposit Stake:**
```
boundless-cli account deposit-stake STAKE_AMOUNT
```
* Deposit Stake Balance: `boundless-cli account stake-balance`

##  Run Broker
You can now start `broker` (which runs both `bento` + `broker` i.e. the full proving stack!):
```bash
just broker
```
Check the total proving logs:
```bash
just broker logs
```
Check the `broker` logs, which has the most important logs of your order fulfillments:
```
docker compose logs -f broker
```

## Broker Configuration and Optimize
* Broker is not the prover, it's for onchain activities like locking orders or setting amount of stake bids, etc.
* `broker.toml` has the settings to configure how your broker interact on-chain and compete with other provers.
```
nano broker.toml
```
* You can see an example of the official `broker.toml` [here](https://github.com/boundless-xyz/boundless/blob/main/broker-template.toml)





```
sudo apt update
sudo apt install postgresql postgresql-client
psql --version
```
