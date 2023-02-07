# near mainnet onboarding corrections

This README clarifies some of the confusion at [NEAR mainnet onboarding](https://github.com/Open-Shard-Alliance/MainnetOnboarding/blob/main/MainnetGO.md).
Steps 1-3 at that link are okay. But Step 4 is incorrect and very confusing.

For the instructions below, our POOL_ID will be `foo`, our NEAR wallet will be `foo.near`, and our validator account will be `foo.poolv1.near`.

## Check that Step 3 worked

If you did step 3 correctly, the following command will display your credentials for your NEAR wallet:
```bash
cat ~/.near-credentials/mainnet/foo.near.json
```

It should look something like
```bash
{"account_id":"foo.near","public_key":"ed25519:xxxxxx.....","private_key":"ed25519:xxxx....."}
```

## Step 4 revised

### install aws cli
```bash
sudo apt-get install awscli -y
```

### create ~/.near and download snapshot

If you have not already created ~/.near, you can do it now
```bash
mkdir ~/.near
cd ~/.near
```

Now download the snapshot, it is about 400GB+, and will take some time to download:
```bash
aws s3 --no-sign-request cp s3://near-protocol-public/backups/mainnet/rpc/latest .
LATEST=$(cat latest)
aws s3 --no-sign-request cp --no-sign-request --recursive s3://near-protocol-public/backups/mainnet/rpc/$LATEST ~/.near/data
```
Note: You can browse this bucket by logging into AWS and visiting:
https://s3.console.aws.amazon.com/s3/buckets/near-protocol-public

###  Initialize node

Please do **NOT** do the `wget` steps to download the config.json and genesis.json. In fact, clear out any existing \*.json files: 

```bash
cd ~/.near
/bin/rm -f *.json
```

The guide is wrong, if you `wget` the file suggested, they are out-of-date, and will cause the `init` step to fail. Let's simply do the init step:
```bash
cd ~/nearcore
./target/release/neard init --chain-id="mainnet" --account-id=foo.poolv1.near
```
This command will create all the necessary \*.json files **including** `validator_key.json`.

If all is good, the result of
```bash
ls ~/.near
```
should show
```bash
config.json  data  genesis.json  node_key.json  validator_key.json
```
That's it. No key files need to be moved around or copied. At this point, you can jump to the section in the guide for the creation of the systemd file `/etc/systemd/system/neard.service`. Once that file is created, enable and start the service:
```bash
sudo systemctl enable neard
sudo systemctl start neard
```

Then use the `journalctl` commands in the guide to view the logs to make sure everything looks good:
```bash
journalctl -n 100 -f -u neard | ccze -A
```

## Deploy staking pool contract

The command should look like this (for a 5% fee):
```bash
near call poolv1.near create_staking_pool '{"staking_pool_id": "foo", "owner_id": "foo.near", "stake_public_key": "UPDATE", "reward_fee_fraction": {"numerator": 5, "denominator": 100}}' --accountId="foo.near" --amount=30 --gas=300000000000000
```

The `stake_public_key` is the public key listed in `~/.near/validator_key.json`

If everything is setup properly, the above command should return something that looks like:
```bash
Receipt: ...
        Failure [poolv1.near]: Error: {"index":0,"account_id":"foo.poolv1.near","minimum_stake":"1908390594064193520557898801","stake":"29999999999999000000000000","kind":{"account_id":"foo.poolv1.near","minimum_stake":"1908390594064193520557898801","stake":"29999999999999000000000000"}}
Receipts: ..., ...
        Log [poolv1.near]: The staking pool @foo.poolv1.near was successfully created. Whitelisting...
Transaction Id ...
To see the transaction in the transaction explorer, please open this url in your browser
https://explorer.mainnet.near.org/transactions/...
true
```

Don't worry about the `Failure [poolv1.near]: Error: ` (this is because the above command has only staked 30 NEAR, which does not meet the minimum staking requirement... this can be fixed later when the NEAR delegation is received. The important thing is that the staking pool gets created and the last line of the command shows `true`.
