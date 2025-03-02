# Create and Interact With a Local R-Squared Blockchain

> Get grasp of the R-Squared blockchain and it's CLI to interact with it. This is a great resource for anyone looking to get started on building, testing or a general overview of the chain's functionality.

### What you will need:
- Linux Or WSL environment with bash
- Git
- Docker
---

## Lets Get Started:

**1. Clone the R-Squared Core [repository](https://github.com/R-Squared-Project/R-Squared-core)**

```bash
mkdir r2

cd r2

git clone git@github.com:R-Squared-Project/R-Squared-core.git

cd r-squared-core
```

**2. Create a new config file for your local blockchain instance (copy this verbatim):**

```bash
mkdir -p witness_node_data_dir

cat > witness_node_data_dir/config.ini << EOL
# Endpoint for P2P node to listen on
p2p-endpoint = 0.0.0.0:2001

# Endpoint for websocket RPC to listen on
rpc-endpoint = 0.0.0.0:8090

# Enable block production
enable-stale-production = true

# ID of witness controlled by this node
witness-id = "1.6.1"

# Tuple of [PublicKey, WIF private key] This allows the witness node to sign blocks
private-key = ["RQRX6JaiMEZZ57Q75Xh3kVbJ4owX13p7f1kkV76B3xLNFuWHVbRSyZ","5KXbCDyCPL3eGX6xX5uJHVwoAYheF7L5fKf67oQocgJA8kNvVHF"]

# Empty seed nodes to prevent connecting to mainnet
seed-nodes = []
p2p-seed-nodes = []
EOL
```
> Keep the WIF private key below handy; we will use this multiple times later on.
>
> "5KXbCDyCPL3eGX6xX5uJHVwoAYheF7L5fKf67oQocgJA8kNvVHF"
>

**3. Copy genesis-dev.json to our witness_node_data_dir folder:**

```bash
sudo docker run --rm \
  -v $(pwd)/witness_node_data_dir:/data \
  -v $(pwd)/libraries/egenesis:/egenesis \
  rsquared-node \
  cp /egenesis/genesis-dev.json /data/
```

**Start the witness node:**

```bash
sudo docker run -d \
  --name rsquared-witness \
  -p 8090:8090 \
  -p 2001:2001 \
  -v $(pwd)/witness_node_data_dir:/var/lib/rsquared \
  rsquared-node witness_node \
  --data-dir=/var/lib/rsquared \
  --genesis-json=/var/lib/rsquared/genesis-dev.json
```

**Check the logs to verify:**

```bash
sudo docker logs -f rsquared-witness
```

We should see an output with something like this:

```
...
...
...

*******************************
*                             *
*  ------- NEW CHAIN -------  *
*  - Welcome to R-Squared! -  *
*  -------------------------  *
*                             *
*******************************

...
...
...
3120378ms th_a       main.cpp:235                  main                 ] Chain ID is a1543995e78a217ea109101d439b7f9d64da1135900b057ec5a12cf09a361f5c
3170003ms th_a       db_maint.cpp:292              update_active_witnes ] No top witnesses with reveals found. No-reveal penalties are not applicable.
3170003ms th_a       witness.cpp:591               block_production_loo ] Generated block #1 with 0 transaction(s) and timestamp 2025-02-16T00:52:50 at time 2025-02-16T00:52:50
```

This tells us we just successfully initialized a brand new chain—we also see the Chain Id which is what we will need for the next step:

```
Chain ID is a1543995e78a217ea109101d439b7f9d64da1135900b057ec5a12cf09a361f5c
```

You should start seeing logs declaring new block production from our new local RQRX blockchain instance.

We can now exit the dockers logs with CTRL+C in our terminal, or just create new terminal.

**Connect to node and import a wallet on our local chain:**

```bash
sudo docker run -it --rm \
  --name rsquared-wallet \
  --network container:rsquared-witness \
  rsquared-node cli_wallet \
  -s "ws://127.0.0.1:8090" \
  --chain-id=a1543995e78a217ea109101d439b7f9d64da1135900b057ec5a12cf09a361f5c
```

**We should then have access to the Wallet CLI with a prompt to set a password for a new wallet:**

```bash
Please use the "set_password" method to initialize a new wallet before continuing
new >>> 
```

Okay, lets set our password as “dev”, just for the tutorial:

```bash
Please use the "set_password" method to initialize a new wallet before continuing
new >>> set_password dev
```

This will log “null” to the console, this is normal—let’s continue: Now we need to unlock our wallet with our new password:

```bash
 locked >>> unlock dev
```

Now that it’s unlocked, we will check the blockchain info—just to make sure we are on our expected local Chain Id:

```bash
unlocked >>> info
{
  "head_block_num": 5,
  "head_block_id": "00000005ca20351f1310be69e9c729ef052bdfd2",
  "head_block_age": "42 seconds old",
  "next_maintenance_time": "3 minutes in the future",
  "chain_id": "a1543995e78a217ea109101d439b7f9d64da1135900b057ec5a12cf09a361f5c",
  # above is the same Chain Id we used earlier, looks like we are following these steps correctly.*
  ...
}
```

We can now use the list_my_accounts method to see what accounts we currently have ownership of:

```bash
unlocked >>> list_my_accounts
[] # empty array is returned
```

Looks like we need to create or add an account, lets import the “init0” account defined in the genesis-dev.json file:

```bash
# "5KXbCDyCPL3eGX6xX5uJHVwoAYheF7L5fKf67oQocgJA8kNvVHF" 
# ^ this is the private key from our inital config.ini from the first step

unlocked >>> import_key init0 5KXbCDyCPL3eGX6xX5uJHVwoAYheF7L5fKf67oQocgJA8kNvVHF
2327282ms th_a       wallet_api_impl.cpp:437       save_wallet_file     ] saving wallet to file wallet.json
2327283ms th_a       wallet_api_impl.cpp:456       save_wallet_file     ] saved successfully wallet to tmp file wallet.json.tmp
2327283ms th_a       wallet_api_impl.cpp:462       save_wallet_file     ] validated successfully tmp wallet file wallet.json.tmp
2327283ms th_a       wallet_api_impl.cpp:466       save_wallet_file     ] renamed successfully tmp wallet file wallet.json.tmp
2327283ms th_a       wallet_api_impl.cpp:473       save_wallet_file     ] successfully saved wallet to file wallet.json
2327283ms th_a       wallet_api_impl.cpp:277       copy_wallet_file     ] backing up wallet wallet.json to after-import-key-397f0e77.wallet
true 

# true was returned, that means the account was successfully imported
```

**Great, we can now use this account within the blockchain. Using the list_account_balances method we can display our accounts holdings, currently it holds nothing—let’s change that.**

Using the import_balance method we can import the initial balance of the genesis-dev.json file referenced below:

```bash
{
...
"initial_balances": [
    {
      "owner": "RQRXBrWn6V3fn1KdCZRo4DVmrMq7MpTtic5CW",
      "asset_symbol": "RQRX",
      "amount": "100000000000000"
    }
  ],
 ...
 }
```

Import balance into “init0” account, with the private key inside a vector:

```bash
unlocked >>> import_balance init0 ["5KXbCDyCPL3eGX6xX5uJHVwoAYheF7L5fKf67oQocgJA8kNvVHF"] true
[{
    "ref_block_num": 5,
    "ref_block_prefix": 523575498,
    "expiration": "2025-02-16T00:57:15",
    "operations": [[
        32,{
          "fee": {
            "amount": 0,
            "asset_id": "1.3.0"
          },
          "deposit_to_account": "1.2.6",
          "balance_to_claim": "1.15.0",
          "balance_owner_key": "RQRX6JaiMEZZ57Q75Xh3kVbJ4owX13p7f1kkV76B3xLNFuWHVbRSyZ",
          "total_claimed": {
            "amount": "100000000000000",
            "asset_id": "1.3.0"
          }
        }
      ]
    ],
    "extensions": [],
    "signatures": [
      "20744d7394913cddc07459b2e2724166f6cb5bf44ee79a12dcd53229118a2d1fe701ac4c6e6a096c591872a76d588964bdcfa903ad118e7bb8db8e36f22ba84096"
    ]
  }
]
```

Looks like it worked, we just imported the entire balance of the RQRX assets from the genesis block. Time to verify if it’s in our “init0” account:

```bash
unlocked >>> list_account_balances init0
1000000000 RQRX
```

Yup, we have 1 billion RQRX in our account on our local blockchain (no you cannot buy lambos with this).

As I stated; the purpose of this is to get a basic start on developing and experimenting on top of the R-Squared blockchain. To list all methods in the Wallet CLI, simply type “help”—like so:

```bash
unlocked >>> help											
```

This will reveal all of the methods from the 'wallet_cli' program to play with. Happy exploring, builders.
