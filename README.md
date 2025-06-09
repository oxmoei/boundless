# boundless


Dependecies:
```
apt install build-essential libclang-dev cmake pkg-config libssl-dev ninja-build -y

```
Install rustup:

```
rustup update
```

Install the Rust Toolchain:
```
rustup default stable

OR

apt update
apt install cargo
```

Verify Installation:
Check that cargo is installed:
```bash
cargo --version
```


Install Bento-client
```
cargo install --git https://github.com/risc0/risc0 --bin bento_cli
```
```
echo 'export PATH="$HOME/.cargo/bin:$PATH"' >> ~/.bashrc
```
```
source ~/.bashrc
```
```
bento_cli --version
```


Step 1: Install the RISC Zero Toolchain
Install `rzup`:
```bash
curl -L https://risczero.com/install | bash
```
```
source ~/.bashrc
```
Verify `rzup` is installed:
```
rzup --version
```
Install the RISC Zero Rust Toolchain:
```bash
rzup install rust
```

Install `cargo-risczero`:

```bash
cargo install cargo-risczero
rzup install cargo-risczero
```


