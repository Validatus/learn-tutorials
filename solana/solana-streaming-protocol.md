# Introduction

Real-time streaming can be understood as a constant flow of assets from one wallet to another over a period of time. It simplifies transactions and also enables a trustless environment. Subscription services or freelancers can use streaming transactions to maintain a trustless environment with their customers.
There are many uses of this concept. You can check out the [SuperFluid](https://www.superfluid.finance/home) protocol built on the Ethereum network as an example. This tutorial is comprised of three parts. First, we will write a Solana program to handle the streaming, then we will create a backend for our protocol. Finally, we will connect it all with our front-end.

# Prerequisites

A good understanding of the [Rust](https://www.rust-lang.org/) programming language and [React](https://reactjs.org/) and [Redux](https://redux.js.org/) is required to grasp the contents of this tutorial.

# Requirements

The following software is required to complete this tutorial:

- Git, install it from [HERE](https://git-scm.com/downloads).
- Solana CLI, install it from [HERE](https://docs.solana.com/cli/install-solana-cli-tools#use-solanas-install-tool).
- Solana wallet
- The Rust toolchain, install it from [HERE](https://www.rust-lang.org/tools/install).
- Node.js (v14.18.1+), install it from [HERE](https://nodejs.org/en/download/).
- Postgress, install it from [HERE](https://www.postgresql.org/)

We will write the tutorial in 3 parts. First, we will write our Solana program then we will create a backend for our protocol, and in the end, we will connect it all with our front-end.

# Solana Program

Before getting into the coding, let us briefly review what a **streaming protocol** means and what instructions we need to create in our Solana program.
A **streaming protocol** creates an escrow account that will keep track of the balance of both parties with the help of time. Initially, all the funds in the escrow account will be owned by the sender. As time passes, ownership of some funds will be transferred to the receiver which means the receiver can then withdraw those funds.

We will need 3 instructions in our Solana program

- **To Create a Stream**: This can be used the the sender and they will be funding our escrow account, give information about the receiver and define start and end time for the Stream.
- **Withdraw funds**: This can be used by the receiver to withdraw the funds they have earned.
- **Cancel Stream**: This may be used by the sender to cancel a stream. This instruction will distribute the owned funds in the escrow account to senders and receiver accounts.

Let's get started with creating our program! Open a terminal in your projects folder and run the following command to create a new Rust project using the library template:

```text
cargo new --lib sol-stream-program
```

We will be now able to see the `sol-stream-program` folder in our projects directory so we can open it open our code editor. I am using Visual Studio Code (commonly called VSCode) for this tutorial. You can use code editor of your choice - in the `Cargo.toml` file, add the required dependencies:

```toml
[package]
name = "sol-stream-program"
version = "0.1.0"
edition = "2018"

[dependencies]
solana-program = "=1.8.1"
borsh = "0.9.1"
thiserror = "1.0.24"

[dev-dependencies]
solana-program-test = "=1.8.1"
solana-sdk = "=1.8.1"

[lib]
crate-type = ["cdylib", "lib"]
```

Now to download all the crates, we can save the `Cargo.toml` file and in the terminal run:

```text
cargo check
```

We will create our program with the following structure:

```text
├─ src
│  ├─ lib.rs -> registering modules
│  ├─ entrypoint.rs -> entrypoint to the program
│  ├─ instruction.rs -> program API, (de)serializing instruction data
│  ├─ processor.rs -> program logic
│  ├─ state.rs -> program objects, (de)serializing state
│  ├─ error.rs -> program specific errors
├─ .gitignore
├─ Cargo.lock
├─ Cargo.toml
```

The flow of a program using this structure looks like this:

1. Someone calls the entrypoint
2. The entrypoint forwards the arguments to the processor
3. The processor asks `instruction.rs` to decode the `instruction_data` argument from the entrypoint function.
4. Using the decoded data, the processor will now decide which processing function to use to process the request.
5. The processor may use `state.rs` to encode state into or decode the state of an account which has been passed into the entrypoint.

The code in `instruction.rs` defines the API of the program.

Now let us create the `entrypoint.rs`, `instruction.rs`, `processor.rs`, `state.rs`, and `error.rs` files in the `sol-stream-program/src` directory. We can register the newly created files in `lib.rs` by updating it to include the modules:

```rust
pub mod error;
pub mod instruction;
pub mod native_mint;
pub mod processor;
pub mod state;
pub mod entrypoint;
```

Now in the `src/entrypoint.rs` file let's add the code for our entrypoint function and register it by using the `entrypoint!` macro:

```rust
//! Program entrypoint

use solana_program::{
    account_info::AccountInfo, entrypoint, entrypoint::ProgramResult,
    program_error::PrintProgramError, pubkey::Pubkey,
};

fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    Ok(())
}
entrypoint!(process_instruction);
```

We are importing the required structs, functions and macros from the `solana_program` crate. Then we create a function that will take program_id, accounts, and instruction_data as input parameters and returns a ProgramResult. We then register this function with the `entrypoint!` macro in the last line.

Now let us open `instruction.rs` and add the following code:

```rust
use borsh::BorshDeserialize;
use solana_program::program_error::ProgramError;

/// Instructions supported by the sol-streaming program.
#[repr(C)]
#[derive(Clone, Debug, PartialEq)]
pub enum StreamInstruction {
    /// Create a stream with a escrow account created and funded by sender
    /// account should have a total_lamport=admin_cut+program_rent_account+amount_to_send.
    ///
    /// Accounts expected:
    ///
    /// `[writable]` escrow account, it will hold all necessary info about the trade.
    /// `[signer]` sender account
    /// `[]` receiver account
    /// `[]` Admin account
    CreateStream,

    /// Withdraw from a stream for receiver
    ///
    /// Accounts expected:
    ///
    /// `[writable]` escrow account, it will hold all necessary info about the trade.
    /// `[signer]` receiver account
    WithdrawFromStream,

    /// Close a stream and transfer tokens between sender and receiver.
    ///
    /// Accounts expected:
    ///
    /// `[writable]` escrow account, it will hold all necessary info about the trade.
    /// `[signer]` sender account
    /// `[]` receiver account
    CloseStream,
}
```

Note that `CreateStream` and `WithdrawFromStream` would require the some input from initiator, let us create structs for them in `state.rs`. In `state.rs` add the following code:

```rust
use borsh::{BorshDeserialize, BorshSerialize};
use solana_program::{clock::UnixTimestamp, pubkey::Pubkey};

#[derive(Clone, Debug, PartialEq, BorshDeserialize, BorshSerialize)]
pub struct CreateStreamInput {
    pub start_time: UnixTimestamp,
    pub end_time: UnixTimestamp,
    pub receiver: Pubkey,
    pub lamports_withdrawn: u64,
    pub amount_second: u64,
}

#[derive(Clone, Debug, PartialEq, BorshDeserialize, BorshSerialize)]
pub struct WithdrawInput {
    pub amount: u64,
}
```

For `CreateStream` we will want `CreateStreamInput` struct:

- `start_time`: The [Unix timestamp](https://en.wikipedia.org/wiki/Unix_time) at which the stream will start.
- `end_time`: The [Unix timestamp](https://en.wikipedia.org/wiki/Unix_time) at which the stream will end.
- `receiver`: Public key of the receiver.
- `lamports_withdrawn`: We allow the receiver to withdraw Lamports when they have ownership, we would also want to keep track of the number of Lamports withdrawn for calculation purposes.
- `amount_speed`: The number of Lamports transferred to the receiver every second.

For `WithdrawFromStream` we will want the `WithdrawInput` struct:

- `amount`: The amount of Lamports the receiver wants to withdraw.

Now let us import these structs into `instruction.rs` and use them, add this line at the very top of the file:

```rust
use crate::state::{CreateStreamInput, WithdrawInput};
```

Update `CreateStream` and `WithdrawFromStream` to :

```rust
// CreateStream,
CreateStream(CreateStreamInput),
```

```rust
// WithdrawFromStream
WithdrawFromStream(WithdrawInput)
```

For reference, you can check the file [HERE](https://github.com/SushantChandla/sol-stream-program/blob/main/src/instruction.rs).

Now that our enums are complete, let us add a function to unpack the instruction given to our program. At the end of `instruction.rs`:

```rust
impl StreamInstruction {
    pub fn unpack(instruction_data: &[u8]) -> Result<Self, ProgramError> {
        let (tag, data) = instruction_data
            .split_first()
            .ok_or(ProgramError::InvalidInstructionData)?;
        match tag {
            1 => Ok(StreamInstruction::CreateStream(
                CreateStreamInput::try_from_slice(data)?,
            )),
            2 => Ok(StreamInstruction::WithdrawFromStream(
                WithdrawInput::try_from_slice(data)?,
            )),
            3 => Ok(StreamInstruction::CloseStream),
            _ => Err(ProgramError::InvalidInstructionData),
        }
    }
}
```

We have added a function to unpack data since there is only one entrypoint. We have used the first element of `instruction_data` as a tag. Then we have used the `BorshDeserialization` derivation which provides us with the `try_from_slice` function to unpack data. We are using:

- Tag 1 -> Create Stream Instruction.
- Tag 2 -> Withdraw Instruction.
- Tag 3 -> Close Stream Instruction.

Returning an error otherwise. Now we can open the `processor.rs` file and add the logic for instructions in it.

```rust
use std::str::FromStr;

use crate::{
    instruction::StreamInstruction,
    state::{CreateStreamInput, StreamData, WithdrawInput},
};
use borsh::{BorshDeserialize, BorshSerialize};
use solana_program::{
    account_info::{next_account_info, AccountInfo},
    clock::Clock,
    entrypoint::ProgramResult,
    program_error::ProgramError,
    pubkey::Pubkey,
    sysvar::{rent::Rent, Sysvar},
};
pub struct Processor;

impl Processor {
    pub fn process(
        program_id: &Pubkey,
        accounts: &[AccountInfo],
        instruction_data: &[u8],
    ) -> ProgramResult {
        let instruction = StreamInstruction::unpack(instruction_data)?;

        match instruction {
            StreamInstruction::CreateStream(data) => todo!(),
            StreamInstruction::WithdrawFromStream(data) =>todo!(),
            StreamInstruction::CloseStream => todo!(),
        }
    }
}
```

We have imported the required struct and functions from the `solana_program` crate. Then we will create a struct called `Processor` and implement it by creating a function named `process` which will take as input:

- `program_id`: Program ID of program.
- `accounts`: Accounts passed in the transaction.
- `instruction_data`: Instruction data passed for instruction.

This function will return `ProgramResult`. In the function, we have unpacked the `instruction_data` using the unpack function we created on `StreamInstruction` enum. We have used the `?` operator to unwrap valid values or return the error value, propagating them to the calling function. Then we have used a match statement on the instructions and stubbed them out by adding the `todo!()` macro to each of them.

Before we move on to defining the instructions, let us update the `entrypoint.rs` file.

```rust
use crate::processor::Processor;
fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    Processor::process(program_id, accounts, instruction_data)
}
```

We have imported the `Processor` and updated the function to call the `process` function passing the same arguments.
For reference, you can check the file [HERE](https://github.com/SushantChandla/sol-stream-program/blob/main/src/processor.rs).

Now let us go to `processor.rs` and remove the todos. For each instruction, we will create a function and write the logic in there.

Update the code in `processor.rs` to:

```rust
impl Processor {
    pub fn process(
        program_id: &Pubkey,
        accounts: &[AccountInfo],
        instruction_data: &[u8],
    ) -> ProgramResult {
        let instruction = StreamInstruction::unpack(instruction_data)?;

        match instruction {
            StreamInstruction::CreateStream(data) => {
                Self::process_create_stream(program_id, accounts, data)
            }
            StreamInstruction::WithdrawFromStream(data) => {
                Self::process_withdraw(program_id, accounts, data)
            }
            StreamInstruction::CloseStream => Self::process_close(program_id, accounts),
        }
    }

    fn process_create_stream(
        _program_id: &Pubkey,
        accounts: &[AccountInfo],
        data: CreateStreamInput,
    ) -> ProgramResult {
        Ok(())
    }

    fn process_withdraw(
        _program_id: &Pubkey,
        accounts: &[AccountInfo],
        data: WithdrawInput,
    ) -> ProgramResult {
        Ok(())
    }

    fn process_close(_program_id: &Pubkey, accounts: &[AccountInfo]) -> ProgramResult {
        Ok(())
    }
}
```

We have created empty functions `process_create_stream`, `process_withdraw`, and `process_close`. The `process_create_stream` and `process_withdraw` functions have a parameter `data` which would be the struct `CreateStreamInput` and `WithdrawInput` respectively.

Now let's open `errors.rs` and write the errors that our program might return in some cases.

```rust
use thiserror::Error;
use solana_program::{msg, program_error::ProgramError};

#[derive(Error, Debug, Copy, Clone)]
pub enum StreamError {
    #[error("Failed to parse the pubkey")]
    PubKeyParseError,
    #[error("Admin account invalid")]
    AdminAccountInvalid,
    #[error("Not enough lamports in account")]
    NotEnoughLamports,
    #[error("Start time or end time for the stream is invalid")]
    InvalidStartOrEndTime,
    #[error("Receiver does not own enough tokens")]
    WithdrawError,
}

impl From<StreamError> for ProgramError {
    fn from(e: StreamError) -> Self {
         msg!("{}", e);
        ProgramError::Custom(e as u32)
    }
}
```

For reference, you can check the file [HERE](https://github.com/SushantChandla/sol-stream-program/blob/main/src/error.rs).

## Writing Instruction logic

Now we can open the `processor.rs` file again and complete `process_create_stream`, `process_withdraw` and `process_close` function.

**process_create_stream**
We will parse the public key for admin. I have used my pubkey here as an example, but you can use your public key. We are using the `from_str` method. In case of an error, we return `PubKeyParseError` which we defined in `errors.rs`.

Then we will get all the accounts. First we create an iterator and then we can use the `next_account_info` function to get all the accounts. We will store all the accounts with a corresponding variable name.

Now we can check if the admin account provided was correct or incorrect by comparing its key with the pubkey we provided in our function. If it is incorrect, we return an error.

```rust
// Updated at top of file.
use crate::{
    error::StreamError,
    instruction::StreamInstruction,
    state::{CreateStreamInput, StreamData, WithdrawInput},
};
    ...
    fn process_create_stream(
        _program_id: &Pubkey,
        accounts: &[AccountInfo],
        data: CreateStreamInput,
    ) -> ProgramResult {
        let admin_pub_key = match Pubkey::from_str("DGqXoguiJnAy8ExJe9NuZpWrnQMCV14SdEdiMEdCfpmB") {
            Ok(key) => key,
            Err(_) => return Err(StreamError::PubKeyParseError.into()),
        };

        let account_info_iter = &mut accounts.iter();
        let escrow_account = next_account_info(account_info_iter)?;
        let sender_account = next_account_info(account_info_iter)?;
        let receiver_account = next_account_info(account_info_iter)?;
        let admin_account = next_account_info(account_info_iter)?;

        if *admin_account.key != admin_pub_key {
            return Err(StreamError::AdminAccountInvalid.into());
        }
```

Now, we can make a transaction of 0.03 lamports (an arbitrary amount) and send those to the admin account.

> Note: Transactions are atomic, if program fails the whole transaction will be reverted.

Now, we can check the given instruction data.
When we check the end time, it shouldn't be less than the start time and the start time shouldn't be less than the current time. We can get the current `unix_timestamp` by using `Clock::get()?.unix_timestamp`. We will return an `InvalidStartOrEndTime` error in case of failure.

Then we can check that the total Lamport deposited in the account should be equal to the amount we want to send + the minimum number of Rent required to create the account on the chain. We can get the minimum amount of balance required by using the `Rent::get()?.minimum_balance(len)` method. If this failed we can return the `NotEnoughLamports` error.

```rust
        // 0.03 sol token admin account fee
        // 30000000 Lamports = 0.03 sol
        **escrow_account.try_borrow_mut_lamports()? -= 30000000;
        **admin_account.try_borrow_mut_lamports()? += 30000000;

        if data.end_time <= data.start_time || data.start_time < Clock::get()?.unix_timestamp {
            return Err(StreamError::InvalidStartOrEndTime.into());
        }

        if data.amount_second * ((data.end_time - data.start_time) as u64)
            != **escrow_account.lamports.borrow()
                - Rent::get()?.minimum_balance(escrow_account.data_len())
        {
            return Err(StreamError::NotEnoughLamports.into());
        }
```

Then we will check who signed this transaction, and the public key of the receiver is equal to the account provided to us.

```rust
        if !sender_account.is_signer {
            return Err(ProgramError::MissingRequiredSignature);
        }

        if *receiver_account.key != data.receiver {
            return Err(ProgramError::InvalidAccountData);
        }
```

Now we are all set to write the stream data to our program account. We will create a `StreamData` struct and store that in our escrow account.
In `state.rs` at the end add a new struct:

```rust
#[derive(Clone, Debug, PartialEq, BorshDeserialize, BorshSerialize)]
pub struct StreamData {
    pub start_time: UnixTimestamp,
    pub end_time: UnixTimestamp,
    pub receiver: Pubkey,
    pub lamports_withdrawn: u64,
    pub amount_second: u64,
    pub sender: Pubkey,
}

impl StreamData {
    pub fn new(data: CreateStreamInput, sender: Pubkey) -> Self {
        StreamData {
            start_time: data.start_time,
            end_time: data.end_time,
            receiver: data.receiver,
            lamports_withdrawn: 0,
            amount_second: data.amount_second,
            sender,
        }
    }
}
```

For reference, you can check the file [HERE](https://github.com/SushantChandla/sol-stream-program/blob/main/src/state.rs).

We have added a new method to create an instance of the struct with the help of `CreateStreamInput` and the sender's public key.

Now let's jump back to the `processor.rs` file and complete the function:

```rust
       let escrow_data = StreamData::new(data, *sender_account.key);

        escrow_data.serialize(&mut &mut escrow_account.data.borrow_mut()[..])?;
        Ok(())
    }
```

We will first create the `escrow_data` and then with the help of the borsh `serialize` method we can write data to `escrow_account`. We complete the function by returning the result `Ok(())` at the end of the function.

**process_withdraw**

We have stored accounts into variables just like we did in the `process_create_stream` function. Then we have deserialized data in the `escrow_account` not that this data is what we saved in the `process_create_stream` function. Then we perform a check that the receiver of this account is the singer and this `escrow_account` belongs to him.

```rust
 fn process_withdraw(
        _program_id: &Pubkey,
        accounts: &[AccountInfo],
        data: WithdrawInput,
    ) -> ProgramResult {
        let account_info_iter = &mut accounts.iter();
        let escrow_account = next_account_info(account_info_iter)?;
        let receiver_account = next_account_info(account_info_iter)?;

        let mut escrow_data = StreamData::try_from_slice(&escrow_account.data.borrow())
            .expect("failed to serialize escrow data");

        if *receiver_account.key != escrow_data.receiver {
            return Err(ProgramError::IllegalOwner);
        }

        if !receiver_account.is_signer {
            return Err(ProgramError::MissingRequiredSignature);
        }
```

Then we can check if the user can withdraw the required Lamports or not. We will get the current time and calculate the total number of Lamports owned by them. By subtracting `lamports_withdrawn`, we can keep track of the Lamports that are already withdrawn by the receiver.

```rust
        let time = Clock::get()?.unix_timestamp;

        let total_token_owned = escrow_data.amount_second
            * ((std::cmp::min(time, escrow_data.end_time) - escrow_data.start_time) as u64)
            - escrow_data.lamports_withdrawn;

        if data.amount > total_token_owned {
            return Err(StreamError::WithdrawError.into());
        }
```

Now we can proceed with the transaction and send the token to `receiver_account`. We will also make an increment in `lamports_withdrawn`. We finish the function by writing the new `escrow_data` to `escrow_account` and then returning the result `Ok(())`.

```rust
        **escrow_account.try_borrow_mut_lamports()? -= data.amount;
        **receiver_account.try_borrow_mut_lamports()? += data.amount;
        escrow_data.lamports_withdrawn += data.amount;

        escrow_data.serialize(&mut &mut escrow_account.data.borrow_mut()[..])?;
        Ok(())
    }
```

**process_close**

In this function also we will get the accounts and store them in variables. Then we get the `escrow_data` just like we did in the `process_withdraw` function. We then check if the `sender_account` is the owner of `escrow_account` and if `sender` has signed the transaction or not.

```rust
fn process_close(_program_id: &Pubkey, accounts: &[AccountInfo]) -> ProgramResult {
        let account_info_iter = &mut accounts.iter();
        let escrow_account = next_account_info(account_info_iter)?;
        let sender_account = next_account_info(account_info_iter)?;
        let receiver_account = next_account_info(account_info_iter)?;

        let mut escrow_data = StreamData::try_from_slice(&escrow_account.data.borrow())
            .expect("failed to serialize escrow data");

        if escrow_data.sender != *sender_account.key {
            return Err(ProgramError::IllegalOwner);
        }
        if !sender_account.is_signer {
            return Err(ProgramError::MissingRequiredSignature);
        }
```

We are closing the escrow account, so we want to transfer the funds to the receiver and sender which they own. So we can calculate the total tokens owned by the receiver.

```rust
        let time: i64 = Clock::get()?.unix_timestamp;
        let mut lamport_streamed_to_receiver: u64 = 0;

        if time > escrow_data.start_time {
            lamport_streamed_to_receiver = escrow_data.amount_second
                * ((std::cmp::min(time, escrow_data.end_time) - escrow_data.start_time) as u64)
                - escrow_data.lamports_withdrawn;
        }
```

Now we have the total Lamports that are owned by the receiver. We can send the remaining Lamports to the sender. At last, we set the `escrow_account` balance to 0, then we can return the result `Ok(())`.

```rust
        **receiver_account.try_borrow_mut_lamports()? += lamport_streamed_to_receiver;
        escrow_data.lamports_withdrawn += lamport_streamed_to_receiver;
        **sender_account.try_borrow_mut_lamports()? += **escrow_account.lamports.borrow();

        **escrow_account.try_borrow_mut_lamports()? = 0;

        Ok(())
    }
```

For reference, you can check the file [HERE](https://github.com/SushantChandla/sol-stream-program/blob/main/src/processor.rs).

You can check the complete code of the Solana program on [GitHub](https://github.com/SushantChandla/sol-stream-program).

## Deploy the program

Now to deploy the program, we can use the following command to create a build. In this command, the manifest-path should be the path of your `Cargo.toml` file. This will output the compiled program in Shared Object format (.so) in the `dist/program` directory:

```text
cargo build-bpf --manifest-path=Cargo.toml --bpf-out-dir=dist/program
```

We will create a new Solana account to deploy the program. Run the following command:

```text
solana-keygen new -o keypair.json
```

The command will prompt you for a passphrase to secure the recovery seed phrase:

```text
Generating a new keypair

For added security, enter a BIP39 passphrase

NOTE! This passphrase improves security of the recovery seed phrase NOT the
keypair file itself, which is stored as insecure plain text

BIP39 Passphrase (empty for none):
```

You can choose a passphrase or leave it empty. Continuing will provide both the public key of the account and the seed phrase used to create the private key:

```text
Wrote new keypair to keypair.json
=====================================================================
pubkey: 7WQDnydTTtyb2DsTuuFpeu2bDxQdpZMRc4R6qja1UzP
=====================================================================
Save this seed phrase and your BIP39 passphrase to recover your new keypair:
lemon avoid all erase chair acid fire govern glue outside wheel clock
=====================================================================
```

Once complete you will have the `keypair.json` file, containing the private and public keys of a new Solana account. It is important to keep your keypair safe. Do not commit this file to a remote code repository. It is best to add this file to a `.gitignore` to prevent this from happening.

Now we are going to request an airdrop of SOL tokens on the Solana Devnet. This command will add 1 SOL token to the account:

```text
solana airdrop 1 <YourPublicKey> --url https://api.devnet.solana.com
```

Example

```text
solana airdrop 1 7WQDnydTTtyb2DsTuuFpeu2bDxQdpZMRc4R6qja1UzP --url https://api.devnet.solana.com
```

> If you are getting an error: "**RPC request error**: cluster version query failed: error sending request for url (https://api.devnet.solana.com/): error trying to connect: dns error: failed to lookup address information: nodename nor servname provided, or not known." - Please consider switching your primary DNS server over to one of Google, DNSWatch, OpenDNS, SAFEDNS, Dyn, Yandex, AdGuard, or Cloudflare.

If you get **insufficient balance** while deploying, you can re-run the command to airdrop funds on Devnet.

Now we will use the following command to deploy. Note that the path of `keypair.json` and `dist/program/program.so` might be different in your case. Please check and then run the command.

```text
solana deploy --keypair keypair.json dist/program/sol_stream_program.so --url https://api.devnet.solana.com
```

We will get the program id as output.

```text
Program Id: DcGPfiGbubEKh1EnQ86EdMvitjhrUo8fGSgvqtFG4A9t
```

We can verify the deployment by checking on the [Solana Explorer for Devnet.](https://explorer.solana.com/?cluster=devnet). We can search for our program id to see any related transactions, including the deployment.

# Backend

We will be creating our backend with help of the [rocket web framework](https://rocket.rs/), but before we start writing the code for the backend it would be a great time to understand why we need a backend.

Let's think of a scenario where we do not have a backend for our protocol. We have stored all the data in the Program driven account(PDA) so for fetching all streams we can `getProgramAccounts` function provided in the `@solana/web3.js` package and then with the help of the `borsh` package we will deserialize the byte data. Then we can check among all the PDA's data which one of those belongs to a user i.e either they are sending or receiving.

Now let us suppose we have made our app live and it has about 1000 users. If all users have created 2 streams, this means we would have 2000 program driven accounts! Fetching all the accounts just to display the 2 streams for each user would make our protocol slow and with any increase in users, it becomes unusable.

We will use our backend to index our PDA's and fix the scalability issue. Let us create a rust project again but this time with the default template i.e binary (application) template.
Open the console in your projects folder and run the following command:

```text
cargo new sol-stream-backend
```

This will generate `sol-stream-backend` directory. We can go ahead and open it on VS-Code now. For now, our project will look like:

```text
├─ src
│  ├─ main.rs -> contain
├─ .gitignore
├─ Cargo.lock
├─ Cargo.toml
```

Now open the `Cargo.toml` file and update it to add the required crates:

```toml
[package]
name = "sol-stream-backend"
version = "0.1.0"
edition = "2018"


[dependencies]
borsh = "0.9.1"
rocket = {version = "0.5.0-rc.1", features = ["tls", "json"]}
rocket_cors = {git = "https://github.com/lawliet89/rocket_cors", branch = "master"}
solana-client = "1.8.3"
solana-account-decoder = "1.8.3"
serde = "1.0.130"
solana-sdk = "1.8.3"
diesel = { version = "1.4.4", features = ["postgres"] }
dotenv = "0.15.0"
```

- borsh: To serialize and deserialize data
- rocket_cors: To enable cross-origin resource sharing
- solana-client: To fetch all program accounts on the solana blockchain
- serde: For JSON Serialization and Deserialization.
- diesel: For SQL query building
- dotenv: To manage our database_url

We are going to need all of these crates, to download them we can save the `Cargo.toml` file and in the terminal run:

```text
cargo check
```

Now in the `main.rs` let us add code to create a "Hello world!" route with rocket.rs.

```rust
use rocket::{get, routes};

#[rocket::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let cors = rocket_cors::CorsOptions::default().to_cors()?;

    rocket::build()
        .mount("/", routes![index,route_with_pubkey])
        .attach(cors)
        .launch()
        .await?;

    Ok(())
}

#[get("/")]
fn index() -> &'static str {
    "Hello, world!"
}

#[get("/<pubkey>")]
fn route_with_pubkey(pubkey: &str)-> String{
    format!("Hello {}",pubkey)
}
```

Before I explain this code run the following command in the terminal:

```text
cargo run
```

It will compile and run the program then we can open `http://127.0.0.1:8000/` on our browser to see:

![browser image](https://github.com/figment-networks/learn-tutorials/raw/master/assets/image-solana-streaming-protocol-1.png?raw=true)

![browser image](https://github.com/figment-networks/learn-tutorials/raw/master/assets/image-solana-streaming-protocol-2.png?raw=true)

Now let's see what code made this happen. In the first line, we have imported `routes` macro and `get` from the rocket. We will use `#[rocket::main]` on our function which will transform our function into a regular main function that internally initializes a Rocket-specific tokio runtime and runs the attributed async fn inside of it.


Then inside the function, we will get default cors (cors stands for Cross Object Resource Sharing) in a variable `cors`. This line of code attaches our `cors` and will start a server with a base `"/"` and it will have routes that will be passed in the `routes!` macro. As you can see we have two routes and the functions for them are called `index` and `route_with_pubkey`. I have passed them in the macro. Then we await this call and return `Ok(())` so that it compiles.

Now take a look at the `index` and `route_with_pubkey` functions. We have made use of the `get` macro here. For the `index` function, we will return a String "Hello world". In the case of `route_with_pubkey` we will get the pubkey from the URL and return "Hello <pubkey>".

> NOTE: You can stop the running server by using `cmd+c` on mac and `control+c` on windows.

Now let us install `diesel` CLI. We can do this by running:

```text
cargo install diesel_cli --feature postgres
```

> This will fail if you do not have Postgres installed.

Now in our project folder, we can run:

```text
diesel setup
```

Now run the following command to create a migration:

```text
diesel migration generate Stream
```

This will create a migrations folder with the name `<Timestamp>_Stream` with `down.sql` and `up.sql` files.

Update the `up.sql` file to:

```sql
-- Your SQL goes here

Create Table streams(
   pda_account Varchar PRIMARY KEY,
    start_time BIGINT NOT NULL,
    end_time BIGINT NOT NULL,
    receiver Varchar Not NULL,
    lamports_withdrawn BIGINT NOT NULL,
    amount_second BIGINT NOT NULL,
    sender Varchar Not NULL,
    total_amount BIGINT NOT NULL
)
```

and `down.sql` file:

```sql
-- This file should undo anything in `up.sql`
Drop Table streams
```

Now we can run the following command which would create `src/schema.rs` file which would contain schema required for `diesel`:

```text
diesel migration run
```

Now we can run the following command to create a `.env` file in our project which would contain the database URL.

```text
DATABASE_URL=postgres://username:password@localhost/sol_stream_indexer > .env
```

> Note that you have to change the `username` and `password` here.

We are going to structure our backend in the following:
![backend-files](https://github.com/figment-networks/learn-tutorials/raw/master/assets/image-solana-streaming-protocol-3.png)

In the `src` directory:

- `main.rs`: will contain the main function and we will use it to add all the modules.
- `models.rs`: will contain `Stream` and `StreamData` model
- `routes.rs`: will contain all the routes in our app.
- `schema.rs`: is generated by diesel cli.
- `solana.rs`: will contain a function to subscribe to the solana program and get all program accounts.

Create the empty files and let us write the code for our backend.

Firstly in `main.rs` at the top add:

```rust
#[macro_use]
extern crate diesel;

mod models;
mod routes;
mod schema;
mod solana;
use rocket::routes;

use diesel::prelude::*;
use dotenv::dotenv;
use std::env;

pub fn establish_connection() -> PgConnection {
    dotenv().ok();
    let database_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");
    PgConnection::establish(&database_url)
        .unwrap_or_else(|_| panic!("Error connecting to {}", database_url))
}
```

We have added `#[macro_use]` for diesel. We will create the function to get `PgConnection` with the help `database_url` we can get with the dotenv. We have not added the `main` function here yet, We will add that later.

Now we can go in `models.rs` and add:

```rust
use crate::diesel::ExpressionMethods;
use borsh::{BorshDeserialize, BorshSerialize};
use diesel::{Insertable, PgConnection, QueryDsl, Queryable, RunQueryDsl};
use serde::Serialize;
use solana_sdk::clock::UnixTimestamp;
use solana_sdk::pubkey::Pubkey;

use crate::schema::streams;

#[derive(Clone, Debug, PartialEq, BorshDeserialize, BorshSerialize)]
struct StreamData {
    pub start_time: UnixTimestamp,
    pub end_time: UnixTimestamp,
    pub receiver: Pubkey,
    pub lamports_withdrawn: u64,
    pub amount_second: u64,
    pub sender: Pubkey,
}

#[derive(Queryable, Insertable, Serialize)]
#[table_name = "streams"]
pub struct Stream {
    pub pda_account: String,
    pub start_time: i64,
    pub end_time: i64,
    pub receiver: String,
    pub lamports_withdrawn: i64,
    pub amount_second: i64,
    pub sender: String,
    pub total_amount: i64,
}
```

Here we have 2 structs one of them is the same we had in our program `StreamData` and `Stream`

- `StreamData`: Used to get the data from the PDA account.
- `Stream`: This is the struct we would like to store in our database. We will have `pda_account` PubKey as ID. We have changed Pubkey types to `String` for `sender` and `receiver` as we can use the `.tostring` function on pubkey and it would be easy to store that in our database compared to an array of i32. We have also drived `Queryable` and `Insertable` from diesel so we can generate find, insert, update, and other SQL queries with the help of diesel. We have also drived `Serialize` from the `serde` crate to convert this to JSON format.

Now we will need some more functions in on `Stream` so at the end of the file add:

```rust
impl Stream {
    pub fn new(pda_pubkey: String, pda_data: &Vec<u8>) -> Option<Self> {
        let stream_data = match StreamData::try_from_slice(pda_data) {
            Ok(a) => a,
            Err(e) => {
                println!(
                    "Failed to deserialize {} with error {:?}",
                    pda_pubkey.to_string(),
                    e
                );
                return None;
            }
        };

        Some(Stream {
            sender: stream_data.sender.to_string(),
            end_time: stream_data.end_time,
            receiver: stream_data.receiver.to_string(),
            lamports_withdrawn: stream_data.lamports_withdrawn as i64,
            start_time: stream_data.start_time,
            total_amount: (stream_data.end_time - stream_data.start_time)
                * stream_data.amount_second as i64,
            pda_account: pda_pubkey,
            amount_second: stream_data.amount_second as i64,
        })
    }

    pub fn get_all_with_sender(pubkey: &String, conn: &PgConnection) -> Vec<Stream> {
        use crate::schema::streams::dsl::*;
        streams
            .filter(sender.eq(pubkey))
            .load::<Stream>(conn)
            .unwrap()
    }
    pub fn get_all_with_receiver(pubkey: &String, conn: &PgConnection) -> Vec<Stream> {
        use crate::schema::streams::dsl::*;
        streams
            .filter(receiver.eq(pubkey))
            .load::<Stream>(conn)
            .unwrap()
    }
    fn id_is_present(id: &String, conn: &PgConnection) -> bool {
        use crate::schema::streams::dsl::*;
        match streams.find(id).first::<Stream>(conn) {
            Ok(_s) => true,
            _ => false,
        }
    }
    pub fn insert_or_update(stream: Stream, conn: &PgConnection) -> bool {
        if Stream::id_is_present(&stream.pda_account, conn) {
            diesel::insert_into(crate::schema::streams::table)
                .values(&stream)
                .execute(conn)
                .is_ok()
        } else {
            use crate::schema::streams::dsl::{
                amount_second as a_s, end_time as e_t, lamports_withdrawn as l_w,
                pda_account as p_a, receiver as r, sender as s, streams, total_amount as t_a,
            };
            diesel::update(streams.find(stream.pda_account.clone()))
                .set((
                    a_s.eq(stream.amount_second),
                    e_t.eq(stream.end_time),
                    r.eq(stream.receiver),
                    p_a.eq(stream.pda_account),
                    s.eq(stream.sender),
                    l_w.eq(stream.lamports_withdrawn),
                    t_a.eq(stream.total_amount),
                    e_t.eq(stream.end_time),
                ))
                .execute(conn)
                .is_ok()
        }
    }
}
```

We have added the following functions:

- `new`: function to create a new `Stream` with the help of `pda` pub key and `pda` data. We first create the `StreamData` with the help of the `borsh` crate and then we can create a return `Stream`.

- `get_all_with_sender`: function to get all the streams with the sender equal to the given public key.

- `get_all_with_receiver`: function to get all the streams with the receiver equal to the given public key.

- `id_is_present`: function to check if the database contains the particular id.

- `insert_or_update`: function to `insert` a new Stream if the `id` is not present which we can check with `id_is_present` function and `update` if it is present.

If you want to learn more about diesel you can read [HERE](https://diesel.rs/guides/getting-started).

Now we can move to the `src/solana.rs` file:

We have added three functions to this file.

```rust
use std::{str::FromStr, thread};

use solana_client::{pubsub_client, rpc_client::RpcClient};
use solana_sdk::{account::Account, pubkey::Pubkey};

use crate::{establish_connection, models::Stream};

pub fn get_all_program_accounts() -> Vec<(Pubkey, Account)> {
    let program_pub_key = Pubkey::from_str("DcGPfiGbubEKh1EnQ86EdMvitjhrUo8fGSgvqtFG4A9t")
        .expect("program address invalid");
    let url = "https://api.devnet.solana.com".to_string();
    let client = RpcClient::new(url);

    client
        .get_program_accounts(&program_pub_key)
        .expect("Something went wrong")
}

pub fn subscribe_to_program() {
    let url = "ws://api.devnet.solana.com".to_string();
    let program_pub_key = Pubkey::from_str("DcGPfiGbubEKh1EnQ86EdMvitjhrUo8fGSgvqtFG4A9t")
        .expect("program address invalid");

    thread::spawn(move || loop {
        let subscription =
            pubsub_client::PubsubClient::program_subscribe(&url, &program_pub_key, None)
                .expect("Something went wrong");
        let conn = establish_connection();

        loop {
            let response = subscription.1.recv();
            match response {
                Ok(response) => {
                    let pda_pubkey = response.value.pubkey;
                    let pda_account: Account = response.value.account.decode().unwrap();

                    let stream = Stream::new(pda_pubkey, &pda_account.data);
                    match stream {
                        Some(a) => Stream::insert_or_update(a, &conn),
                        _ => {
                            println!("data didn't parsed");
                            continue;
                        }
                    };
                }
                Err(_) => {
                    break;
                }
            }
        }
        get_accounts_and_update()
    });
}

pub fn get_accounts_and_update() {
    let program_accounts = get_all_program_accounts();
    let conn = establish_connection();
    for item in program_accounts.iter() {
        let stream = Stream::new(item.0.to_string(), &item.1.data);
        match stream {
            Some(a) => Stream::insert_or_update(a, &conn),
            _ => continue,
        };
    }
}
```

- `get_all_program_accounts`: function will return all the program accounts owned by a program.
- `subscribe_to_program`: function to subscribe to the program which would give us notification whenever there is an update in any program-owned account.
- `get_accounts_and_update`: function to get all the program accounts and fill insert or update them in our database.

> Note: The `pubsub_client` has an [issue](https://github.com/solana-labs/solana/issues/16102) that is still WIP due to which we are missing some notifications.

You can read about Solana RPC [HERE](https://docs.solana.com/developing/clients/jsonrpc-api).

In the `subscribe_to_program` we have spawned a thread to listen to updates we will receive and in case we receive an error we have called the function again. For each notification in the for loop, we have created the stream with the help of `RPC Notification` you can see the JSON schema for that [HERE](https://docs.solana.com/developing/clients/jsonrpc-api#notification-format-2). Then we have called the `insert_or_update` function which we created in our model.

Now let's go to `src/routes.rs`:

```rust
use std::str::FromStr;
use rocket::get;
use rocket::serde::json::serde_json::json;
use rocket::serde::json::{Json, Value};

use crate::establish_connection;
use crate::models::Stream;

#[get("/")]
pub fn index() -> &'static str {
    "Hello, world!"
}

#[get("/<pubkey>")]
pub fn get_all_stream(pubkey: &str) -> Json<Value> {
    let pubkey_string = String::from_str(pubkey).unwrap();
    let conn = establish_connection();
    Json(
        json!({"status":"success","sending":Stream::use std::str::FromStr;

use rocket::get;
use rocket::serde::json::serde_json::json;
use rocket::serde::json::{Json, Value};

use crate::establish_connection;
use crate::models::Stream;

#[get("/")]
pub fn index() -> &'static str {
    "Hello, world!"
}

#[get("/<pubkey>")]
pub fn get_all_stream(pubkey: &str) -> Json<Value> {
    let pubkey_string = String::from_str(pubkey).unwrap();
    let conn = establish_connection();
    Json(
        json!({"status":"success","sending":Stream::get_all_with_sender(&pubkey_string, &conn),"receiving":Stream::get_all_with_receiver(&pubkey_string, &conn)}),
    )
}
```

We have added create two functions the `index` function is the same as we had in our rocket example.

- `get_all_stream`: This function will return Json response we are using `get_all_with_sender` and `get_all_with_receiver` function to get them from our indexed table.

Now let us move to `src/main.rs` and add the main function. At top of the file add the following Code:

```rust
use solana::get_all_program_accounts;
use solana::subscribe_to_program;

use crate::models::Stream;
use crate::routes::get_all_stream;
use crate::routes::index;
```

Now add the `main` function:

```rust
#[rocket::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let program_accounts = get_all_program_accounts();
    let conn = establish_connection();
    for item in program_accounts.iter() {
        let stream = Stream::new(item.0.to_string(), &item.1.data);
        match stream {
            Some(a) => Stream::insert_or_update(a, &conn),
            _ => continue,
        };
    }

    subscribe_to_program();

    let cors = rocket_cors::CorsOptions::default().to_cors()?;

    rocket::build()
        .mount("/", routes![index, get_all_stream])
        .attach(cors)
        .launch()
        .await?;

    Ok(())
}
```

In the main function we are getting all the program accounts and populating our database, then we can call the `subscribe_to_program` function. After this, we are starting the rocket server.

You can see the complete Backend [HERE](https://github.com/SushantChandla/sol-stream-backend).

# Frontend

Now let's write the frontend part of the app. I have created a React app for the frontend, using Redux for state management. I have already created the UI part of it, which you can find [HERE](https://github.com/SushantChandla/sol-stream-frontend/tree/ui).

You can clone and checkout the `ui` branch and run it with the help of the following commands:

```text
yarn
yarn start
```

Once it is running, you can see the UI in the browser:

{% embed url="https://youtu.be/r9CwLIR4afY" caption="UI Sample"  %}

Now let us write the functions to connect with the sol-stream-program.

In `src/actions/index.js`:

```js
import {
  SystemProgram,
  Transaction,
  PublicKey,
  TransactionInstruction,
  Connection,
} from "@solana/web3.js";

import axios from "../config";
import { deserialize, serialize } from "borsh";

const programAccount = new PublicKey(
  "DcGPfiGbubEKh1EnQ86EdMvitjhrUo8fGSgvqtFG4A9t"
);

const adminAddress = new PublicKey(
  "DGqXoguiJnAy8ExJe9NuZpWrnQMCV14SdEdiMEdCfpmB"
);

const cluster = "https://api.devnet.solana.com";
const connection = new Connection(cluster, "confirmed");
```

We have created `programAccount` and `adminAddress` variables to store the program id and admin address for the program.
We have also created a `connection` variable with the help of the `cluster` variable which contains the link to devnet RPC.

Now let's write the `getAllStreams` function first which you can find at the very last in the `action/index.js` file. It just contains a request to our backend client it takes in the string value of `pubkey` as a parameter in it. I have used the axios package for it we can change the base URL in `src/config/index.js`:

```js
import axios from "axios";

export default axios.create({
  baseURL: "http://127.0.0.1:8000",
});
```

Now that our `baseURL` is set we can write the function in `src/actions/index.js` update the `getAllStreams` function to:

```js
export const getAllStreams = (pubkey) => {
  return async (dispatch, getState) => {
    try {
      let response = await axios.get(`/${pubkey}`);
      console.log(response);
      if (response.status !== 200) throw new Error("Something went wrong");
      dispatch({
        type: "DATA_RECEIVED",
        result: response.data,
      });
    } catch (e) {
      console.log(e);
      dispatch({
        type: "DATA_NOT_RECEIVED",
        result: { data: null },
      });
    }
  };
};
```

We are requesting the route "/<pubkey>" and then we dispatch the result. We have added the code in the try and catch block to handle network errors.

## Create Stream

Now let's write the functions that will contain the instructions to our program. We will also create a Schema for data Serialization.

In `src/actions/index.js` file:

```js
class CreateStreamInput {
  constructor(properties) {
    Object.keys(properties).forEach((key) => {
      this[key] = properties[key];
    });
  }
  static schema = new Map([
    [
      CreateStreamInput,
      {
        kind: "struct",
        fields: [
          ["start_time", "u64"],
          ["end_time", "u64"],
          ["receiver", [32]],
          ["lamports_withdrawn", "u64"],
          ["amount_second", "u64"],
        ],
      },
    ],
  ]);
}
```

For creating the stream instructions we have defined a Schema that describes the data type for variables. We will be using this in the `createStream` function:

```js
export const createStream = ({
  receiverAddress,
  startTime,
  endTime,
  amountSpeed,
  wallet
}) => {
  return async (dispatch, getState) => {
    try {
      const SEED = "abcdef" + Math.random().toString();
      let newAccount = await PublicKey.createWithSeed(
        wallet.publicKey,
        SEED,
        programAccount
      );
      let create_stream_input = new CreateStreamInput({
        start_time: startTime,
        end_time: endTime,
        receiver: new PublicKey(receiverAddress).toBuffer(),
        lamports_withdrawn: 0,
        amount_second: amountSpeed,
      });

      let data = serialize(CreateStreamInput.schema, create_stream_input);
```

In the code above we have created a pubkey which we will use to create a Program Drived account(PDA). We have used the function `serialize` from the borsh package to get the Uint8Array of the input data.

```js
      let data_to_send = new Uint8Array([1, ...data]);
      let rent = await connection.getMinimumBalanceForRentExemption(96);
```

We have to append `1` in our Uint8Array as we are using the first character as the tag. Please refer to the unpack function in `instructions.rs`.

We can then get the minimum lamports required to make the account by using the `getMinimumBalanceForRentExemption` function on the connection variable.

```js
      const createProgramAccount = SystemProgram.createAccountWithSeed({
        fromPubkey: wallet.publicKey,
        basePubkey: wallet.publicKey,
        seed: SEED,
        newAccountPubkey: newAccount,
        lamports: ((endTime - startTime) * amountSpeed) + 30000000 + rent,
        space: 96,
        programId: programAccount,
      });

      const instructionTOOurProgram = new TransactionInstruction({
        keys: [
          { pubkey: newAccount, isSigner: false, isWritable: true },
          { pubkey: wallet.publicKey, isSigner: true, },
          { pubkey: receiverAddress, isSigner: false, },
          { pubkey: adminAddress, isSigner: false, isWritable: true }
        ],
        programId: programAccount,
        data: data_to_send
      });
```

In the above code, We have created 2 instruction:

1. To the system program to create a Program drived account with the space of `96` and lamports which are equal to `0.03(admin cut)+solana rent+total amount user want to send`. We are passing these arguments in the `SystemProgram.createAccountWithSeed` function.

2. This instruction contains an array of public keys associated with the transaction, specifically the `newAccount` (the PDA being created), the signer's public key coming from their connected wallet, and also the receiver and admin addresses. The `programId` is the deployed program address and `data` is the instruction data which includes the tag to specify which instruction to execute (CreateStream, WithdrawFromStream or CloseStream).

Then we create a transaction object we have used the `setPayerAndBlockhashTransaction` function which we will add later.
Once we have the transaction object we have to send the transaction with the help of the wallet. We have then dispatched the result.
In case of an error, we can just alert the user.

```js
      const trans = await setPayerAndBlockhashTransaction(
        [createProgramAccount, instructionTOOurProgram], wallet
      );

      let signature = await wallet.sendTransaction(trans, connection);
      const result = await connection.confirmTransaction(signature);
      console.log("end sendMessage", result);
      dispatch(getAllStreams(wallet.publicKey.toString()));
      dispatch({
        type: "CREATE_RESPONSE",
        result: true,
        id: newAccount.toString(),
      });
    } catch (e) {
      alert(e);
      dispatch({ type: "CREATE_FAILED", result: false });
    }
  };
};
```

`setPayerAndBlockhashTransaction`: This function takes in the instructions array and then returns a `Transaction` object.

```js
async function setPayerAndBlockhashTransaction(instructions, wallet) {
  const transaction = new Transaction();
  instructions.forEach(element => {
    transaction.add(element);
  });
  transaction.feePayer = wallet.publicKey;
  let hash = await connection.getRecentBlockhash();
  transaction.recentBlockhash = hash.blockhash;
  return transaction;
}
```

You can check out the `CreateStream` function [HERE](https://github.com/SushantChandla/sol-stream-frontend/blob/418bcdcd434d6b7e1dc3cd9e9d323066c54a6f0b/src/actions/index.js#L102), for reference.

> Note: I checked the size of the data (the value 96) by a test [HERE](https://github.com/SushantChandla/sol-stream-program/blob/d0d537889b155d16fa3c0879e61e4f68c3d47f07/src/state.rs#L42).

## Withdraw

Now let's write the `withdraw` function for this one also we have created the `WithdrawInput` class with schema:

We only have the `amount` variable in the struct for withdrawal input.

```js
class WithdrawInput {
  constructor(properties) {
    Object.keys(properties).forEach((key) => {
      this[key] = properties[key];
    });
  }
  static schema = new Map([[WithdrawInput,
    {
      kind: 'struct',
      fields: [
        ['amount', 'u64'],
      ]
    }]]);
}
```

This time we aren't creating a PDA so we only have one instruction in this transaction which is to our program.
Note that the `streamId` is the address to the `PDA` account.

Here in `data_to_send`, we have to specify 2 at the beginning of the array. As you recall, in our program's `instruction.rs` file we have set the tag 2 for the WithdrawFromStream instruction.

```js
export const withdraw = (streamId, amountToWithdraw, wallet) => {
  return async (dispatch, getState) => {
    try {
      let create_stream_input = new WithdrawInput({
        amount: amountToWithdraw
      });
      let data = serialize(WithdrawInput.schema, create_stream_input);
      let data_to_send = new Uint8Array([2, ...data]);

      const instructionTOOurProgram = new TransactionInstruction({
        keys: [
          { pubkey: streamId, isSigner: false, isWritable: true },
          { pubkey: wallet.publicKey, isSigner: true, },
        ],
        programId: programAccount,
        data: data_to_send
      });

      const trans = await setPayerAndBlockhashTransaction(
        [instructionTOOurProgram], wallet
      );

      let signature = await wallet.sendTransaction(trans, connection);
      const result = await connection.confirmTransaction(signature);
      console.log("end sendMessage", result);
      dispatch({ type: "WITHDRAW_SUCCESS" });
    } catch (e) {
      alert(e);
      dispatch({ type: "WITHDRAW_FAILED" });
    }
  };
}
```

We have dispatched the withdraw result in the end. You can check out the `Withdraw` function [HERE](https://github.com/SushantChandla/sol-stream-frontend/blob/418bcdcd434d6b7e1dc3cd9e9d323066c54a6f0b/src/actions/index.js#L23), for reference.

## Close Stream

Now let's write the last function in the file to close a stream.

For this function, we do not have any instruction data we are only passing the tag. Then we have packed the instruction by calling `setPayerAndBlockhashTransaction` and then dispatched the result.

```js
export const closeStream = (streamId, receiverAddress, wallet) => {
  return async (dispatch, getState) => {
    try {
      const instructionTOOurProgram = new TransactionInstruction({
        keys: [
          { pubkey: streamId, isSigner: false, isWritable: true },
          { pubkey: wallet.publicKey, isSigner: true, isWritable: true },
          { pubkey: receiverAddress, isSigner: false, isWritable: true }
        ],
        programId: programAccount,
        data: new Uint8Array([3])
      });

      const trans = await setPayerAndBlockhashTransaction(
        [instructionTOOurProgram], wallet
      );

      let signature = await wallet.sendTransaction(trans, connection);
      const result = await connection.confirmTransaction(signature);
      console.log("end sendMessage", result);
      dispatch({ type: "CANCEL_SUCCESS" });
    } catch (e) {
      alert(e);
      dispatch({ type: "CANCEL_FAILED" });
    }
  };
};
```

You can check the frontend code [HERE](https://github.com/SushantChandla/sol-stream-frontend). You can check out the `closeStream` function [HERE](https://github.com/SushantChandla/sol-stream-frontend/blob/418bcdcd434d6b7e1dc3cd9e9d323066c54a6f0b/src/actions/index.js#L72), for reference.

Now for testing it we will need the backend server running. Here is a video walkthrough.

{% embed url="https://youtu.be/MUyt5Rz9T64" caption="Sol Streaming Protocol"  %}

# Conclusion

In this tutorial, you learned how to build a Sol token streaming protocol on Solana. We covered the on-chain program's code using the Rust programming language. We built the User Interface with React. And we have created a backend with Rocket framework to index PDA's data.

# About the Author

This tutorial was created by [Sushant Chandla](https://github.com/SushantChandla).
