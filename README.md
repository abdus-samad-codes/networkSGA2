# Banking System using Python, gRPC and SQLite

A command-line banking application demonstrating **remote procedure calls (RPCs)** with Python and gRPC. The project uses Protocol Buffers to define the communication between a client and server, and SQLite to store account balances and transfer records.

> **Note:** This is a local educational demo, not a production banking application. The server listens only on `127.0.0.1:50051` and uses an insecure gRPC channel (no authentication or TLS).

## Features

- **Check account balance** by account ID.
- **Update account balance**, with validation against negative balances.
- **Transfer money** between two different accounts, with insufficient-funds checks.
- **View transaction history** for transfers sent or received by an account.
- **Persist data** using an automatically initialized SQLite database.
- **Handle errors** such as missing accounts, invalid amounts, unavailable servers and RPC timeouts.

All monetary values are stored as **integer paise** to avoid floating-point rounding problems. The client accepts and displays amounts in rupees.

## Tech Stack

- Python 3
- gRPC (`grpcio`)
- Protocol Buffers (`protobuf`, `grpcio-tools`)
- SQLite (Python's built-in `sqlite3` module)

## Project Structure

```text
.
├── banking.proto           # Protocol Buffer messages and gRPC service definitions
├── banking_server.py       # gRPC server, banking operations and SQLite storage
├── banking_client.py       # Interactive command-line client
├── commands.txt            # Installation and code-generation commands
├── README.md               # Project documentation
├── banking_pb2.py          # Generated from banking.proto
├── banking_pb2_grpc.py     # Generated gRPC service/stub code
└── banking.db              # Created automatically when the server first starts
```

The last three files are generated at setup/runtime; they do not need to exist in a fresh checkout. The generated Python files must be present before running the client or server.

## Setup and Installation

### 1. Get the code

Clone your GitHub repository and move into its directory, or download the project files into a folder.

```bash
git clone <your-repository-url>
cd <your-repository-folder>
```

### 2. Create a virtual environment (recommended)

```bash
python -m venv env
```

Activate it:

**Windows PowerShell**

```powershell
.\env\Scripts\Activate.ps1
```

**Linux / macOS**

```bash
source env/bin/activate
```

### 3. Install dependencies

```bash
python -m pip install grpcio grpcio-tools protobuf
```

To verify installation:

```bash
python -m pip show grpcio grpcio-tools protobuf
```

### 4. Generate Python code from the `.proto` file

Run this command **from the folder containing `banking.proto`**:

```bash
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. banking.proto
```

This generates:

- `banking_pb2.py` — message classes.
- `banking_pb2_grpc.py` — gRPC client stubs and server service classes.

## Running the Application

### 1. Start the server

In the first terminal:

```bash
python banking_server.py
```

Expected startup output (the database path will depend on your machine):

```text
Banking gRPC Server started successfully!
SQLite database: <path-to-project>/banking.db
Listening on 127.0.0.1:50051
Press Ctrl+C to stop the server.
```

The server creates `banking.db` and its tables if necessary, and inserts three demo accounts if they do not already exist.

### 2. Start the client

Leave the server running. Open a **second terminal**, activate the same virtual environment, go to the project folder, and run:

```bash
python banking_client.py
```

The interactive menu appears:

```text
========== BANKING SYSTEM ==========
1. Check Account Balance
2. Update Account Balance
3. Transfer Money
4. View Transaction History
5. Exit

Enter your choice (1-5):
```

Choose an option and follow the prompts. Select **5** to exit the client. Press **Ctrl+C** in the server terminal to stop the server.

## Demo Accounts

The following accounts are added on initial database setup:

| Account ID | Account holder (demo label) | Initial balance |
| --- | --- | ---: |
| `101` | Alice | Rs. 10,000.00 |
| `102` | Bob | Rs. 5,000.00 |
| `103` | Charlie | Rs. 8,000.00 |

The labels are shown in the server's seed-data comments; the database itself stores account IDs and balances, not account-holder names.

**Note:** Existing balances are preserved when the server restarts. The initial amounts above apply to a newly created database.

## Example Usage

### Check balance

```text
Enter your choice (1-5): 1
Enter account ID: 101

Account ID: 101
Current Balance: Rs. 10,000.00
```

### Transfer money

```text
Enter your choice (1-5): 3
Enter sender account ID: 101
Enter receiver account ID: 102
Enter transfer amount (Rs.): 500

Money transferred successfully.
Transaction ID: <generated-uuid>
```

After this transfer, account `101` would contain **Rs. 9,500.00** and account `102` would contain **Rs. 5,500.00**, assuming their initial balances have not otherwise changed. Transaction history records the sender, receiver, amount, a generated transaction ID and a UTC timestamp.

## gRPC Services

The service definitions are in [`banking.proto`](banking.proto).

| Service | RPC method | Purpose |
| --- | --- | --- |
| `AccountManagementService` | `getBalance` | Retrieve an account balance |
| `AccountManagementService` | `updateBalance` | Set a new account balance |
| `TransactionService` | `initiateTransfer` | Transfer funds between accounts |
| `TransactionService` | `getTransactionHistory` | Retrieve sent and received transfers |

### How the communication works

1. The client reads the user's choice and inputs.
2. It builds a Protocol Buffer request and calls the corresponding gRPC method through a generated stub.
3. The server validates the request and accesses the SQLite database.
4. The server returns a Protocol Buffer response, which the client displays.

Transfers use a single SQLite transaction to update both account balances and insert the transaction record together. The client sets a **5-second RPC timeout** for each request.

## Validation and Error Handling

- Empty account IDs are rejected by the client.
- Amounts are parsed using Python's `Decimal`, then converted to integer paise.
- Negative balance updates and non-positive transfer amounts are rejected.
- Transfers to the same account are disallowed.
- Transfers fail when the sender lacks sufficient funds.
- Unknown accounts return a gRPC `NOT_FOUND` error.
- The client reports RPC errors and can continue displaying its menu.

## Resetting Demo Data

Stop the server, then delete `banking.db` from the project folder. Restarting the server creates a fresh database with the three initial accounts. **This permanently deletes existing local balances and transaction history.**

## Suggested `.gitignore`

To avoid committing local environment files, generated data and Python caches:

```gitignore
env/
.venv/
__pycache__/
*.pyc
banking.db
banking.db-journal
banking.db-wal
banking.db-shm
```

The generated `banking_pb2.py` and `banking_pb2_grpc.py` files may be committed or regenerated during setup; the instructions above assume they can be regenerated.

## Limitations

This application is intended for learning gRPC and client-server architecture. It has no user authentication, authorization, TLS, or production-grade auditing. The `updateBalance` RPC directly sets balances and is designed as a demo operation. It should not be exposed as a real banking service.
