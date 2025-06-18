# 🚀 Deploy Hello World Solana Program on Solayer Devnet (No Anchor)

This tutorial walks you through deploying a basic "Hello, Solayer!" smart contract to the **Solayer Devnet**, using **only the Solana CLI**.

---

## ✅ Prerequisites

1. **Install Rust + Cargo**
   ```bash
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   ```

2. **Install Solana CLI v1.18.0+**
   ```bash
   sh -c "$(curl -sSfL https://release.solana.com/v1.18.0/install)"
   ```

3. **Install BPF Toolchain**
   ```bash
   rustup component add rust-src
   ```

4. **Create or Import a Solana Wallet**
   ```bash
   solana-keygen new -o ./your-wallet-keypair.json
   ```

   > Ensure the wallet is funded on Solayer Devnet. Ask on [Solayer Discord](https://discord.gg/solana) for testnet SOL if needed.

---

## 🏗️ Step 1: Create Project Structure

```bash
mkdir solayer-hello
cd solayer-hello
cargo new --lib hello
```

---

## ⚙️ Step 2: Configure `Cargo.toml`

Edit `hello/Cargo.toml`:

```toml
[package]
name = "hello"
version = "0.1.0"
edition = "2021"

[dependencies]
solana-program = "1.18.0"

[lib]
crate-type = ["cdylib", "lib"]
```

---

## ✍️ Step 3: Write the Hello World Program

Replace `hello/src/lib.rs` with:

```rust
use solana_program::{
    account_info::AccountInfo, entrypoint, entrypoint::ProgramResult,
    pubkey::Pubkey, msg,
};

entrypoint!(process_instruction);

fn process_instruction(
    _program_id: &Pubkey,
    _accounts: &[AccountInfo],
    _instruction_data: &[u8],
) -> ProgramResult {
    msg!("Hello, Solayer!");
    Ok(())
}
```

---

## 🧱 Step 4: Build the Program

```bash
cargo build-bpf --manifest-path=hello/Cargo.toml --bpf-out-dir=dist
```

> ✅ This creates the binary at `dist/hello.so`

---

## 🔑 Step 5: Generate Program Keypair

```bash
solana-keygen new -o hello_program-keypair.json
solana address -k hello_program-keypair.json
```

> ⚠️ Save the **Program ID** (public key) — you’ll need it for deployment.

---

## 🌐 Step 6: Configure Solayer Devnet

```bash
solana config set --url https://devnet-rpc.solayer.org
solana config set -k ./your-wallet-keypair.json
```

---

## 🚀 Step 7: Deploy the Program

```bash
solana program deploy \
  -u https://devnet-rpc.solayer.org \
  --use-rpc \
  --program-id ./hello_program-keypair.json \
  --upgrade-authority ./your-wallet-keypair.json \
  ./dist/hello.so \
  -k ./your-wallet-keypair.json
```

---

## ✅ Step 8: Verify Deployment

```bash
solana program show <PROGRAM_ID> -u https://devnet-rpc.solayer.org
```

---

## 📝 Expected Results

- ✅ **Deployment Cost**: ~0.16 SOL
- ✅ **Program Logs**: “Hello, Solayer!”
- ✅ **Status**: Successfully deployed and callable

---

## ✅ Step 9: Test the Program

```bash
solana transfer --allow-unfunded-recipient --fund-recipient <PROGRAM_ID> 0.001
```

## 📁 Key Files Created

| File                           | Purpose                          |
|--------------------------------|----------------------------------|
| `hello_program-keypair.json`  | Program ID (keep secure)         |
| `dist/hello.so`               | Compiled program binary          |
| `hello/src/lib.rs`            | Source code                      |
| `hello/Cargo.toml`            | Rust config for BPF compilation  |

---

## 🛠️ Troubleshooting

- **Build Fails**: Ensure correct Rust + Solana CLI versions.
- **Insufficient Funds**: Request SOL from Solayer faucet/Discord.
- **RPC Errors**: Double-check URL and use `--use-rpc` flag.
- **Program Show Issues**: Minor formatting bugs — safe to ignore.

---

## 🔍 What the Program Does

- Accepts any instruction
- Logs `"Hello, Solayer!"` to Solana logs
- Returns success
- Callable by any client/wallet

---
