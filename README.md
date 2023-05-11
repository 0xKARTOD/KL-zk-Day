# How to make your first analysis for free?


### TL;DR

- This topic will try to tell you how to get data from the blockchain and do it for free
- Among the programming languages we will use Python (my version is 3.10) and SQL
- Some useful docs:
  1. https://dune.com/docs/
  2. https://dash.plotly.com/
  3. https://plotly.com/python/
  4. https://pandas.pydata.org/docs/
  5. https://requests.readthedocs.io/en/latest/
  6. https://numpy.org/doc/
  7. https://docs.etherscan.io/
  8. https://www.quicknode.com/docs/ethereum


### Dune & Flipside

### Dune:

```basg
SELECT
    *
FROM erc20_arbitrum.evt_Transfer
WHERE ("to" = 0x671E028C95A09A2CeCA188e619E4cE76E58d1d03
OR "from" = 0x671E028C95A09A2CeCA188e619E4cE76E58d1d03)
AND contract_address = 0xAf5db6E1CC585ca312E8c8F7c499033590cf5C98
AND evt_block_number >= 78700000
```

Link to query: https://dune.com/queries/2459532

### Flipside:

```basg
SELECT
    *
FROM arbitrum.core.fact_token_transfers
WHERE ("TO_ADDRESS" = lower('0x671E028C95A09A2CeCA188e619E4cE76E58d1d03') OR 
"FROM_ADDRESS" = lower('0x671E028C95A09A2CeCA188e619E4cE76E58d1d03'))
AND CONTRACT_ADDRESS = lower('0xAf5db6E1CC585ca312E8c8F7c499033590cf5C98')
AND BLOCK_NUMBER >= 78700000
```



### Dune API:

Import libs:

```basg
import os
import json
import requests
import pandas as pd

from dune_client.types import QueryParameter
from dune_client.client import DuneClient
from dune_client.query import Query
```

API key:

```basg
API_KEY_DUNE = 'PASTE_YOUR_API_KEY'
```

Grab data:

```basg
def convert_Dune_to_DF(results):
        str_results = str(results).replace('ResultsResponse(', '')
        str_results = str_results[str_results.find('result=ExecutionResult(rows='):].replace('result=ExecutionResult(rows=[', '')
        str_remove_results = str_results[str_results.find('], metadata=ResultMetadata(column_names='):]
        str_results = str_results.replace(str_remove_results, '')
        
        string_value = "[" + str_results + "]"
        
        string_value = string_value.replace("'", "\"")
        
        json_data = json.loads(string_value)
        
        df = pd.DataFrame(json_data)

        return df

data_params = []
internal_url = 2459532

query = Query(
    name = "Zk day Query",
    query_id = int(internal_url),
    params = data_params
)

print("Results available at", query.url())

dune = DuneClient(API_KEY_DUNE)
results = dune.refresh(query)


df = convert_Dune_to_DF(results)

df
```


### ETH/BSC/ARBscan API:

Import all:

```basg
from requests import get
import pandas as pd
import json
import time

API_KEY = 'PASTE_ARB_API_KEY'
```


Get transactions functions:

```basg
def make_api_url(token_contract, address, **kwargs):
    BASE_URL = "https://api.arbiscan.io/api"
    url = BASE_URL + f"?module=account&action=tokentx&contractaddress={token_contract}&address={address}&apikey={API_KEY}"

    for key, value in kwargs.items():
        url += f"&{key}={value}"
    return url

def get_transactions(token_contract, address):
    transactions_url = make_api_url(
        token_contract,
        address, 
        startblock = 78700000, 
        endblock = 99999999, 
        page = 1,
        sort = "asc"
    )
    response = get(transactions_url)
    data = response.json()["result"]

    return data
```

Request:

```basg
token_contract = '0xAf5db6E1CC585ca312E8c8F7c499033590cf5C98'
address = '0x671E028C95A09A2CeCA188e619E4cE76E58d1d03'


df = get_transactions(token_contract, address)

df = pd.DataFrame(df)

df
```


### Web3 Python:


Connect to web3:

```basg
# import the following dependencies
import json
from web3 import Web3

# add your blockchain connection information
infura_url = 'PASTE_ARBITRUM_RPC_URL'
web3 = Web3(Web3.HTTPProvider(infura_url))

contract_address = '0xAf5db6E1CC585ca312E8c8F7c499033590cf5C98'

#wallet = '0x7e797199fe79e82b9E77193F36F8bF89f0D4756e'
wallet = '0x671E028C95A09A2CeCA188e619E4cE76E58d1d03'
wallet_sign = '0x000000000000000000000000' + wallet[2:]

latest_block = web3.eth.block_number#83156100
first_block = 78700000


transfer_signature_hash = web3.keccak(text = "Transfer(address,address,uint256)").hex()
```

Event filters:

```basg
### to address
event_filter_to = web3.eth.filter({
    "address": contract_address,
    "fromBlock" : first_block,
    "toBlock" : latest_block,
    "topics": [transfer_signature_hash, wallet_sign]
})

events_to = event_filter_to.get_all_entries()

### from address

event_filter_from = web3.eth.filter({
    "address": contract_address,
    "fromBlock" : first_block,
    "toBlock" : latest_block,
    "topics": [transfer_signature_hash, None, wallet_sign]
})

events_from = event_filter_from.get_all_entries()

events = []

for event in events_from:
    events.append(event)

for event in events_to:
    events.append(event)

    
events
```


Print transaction hashes:

```basg
txs = []

for event in events:
    txs.append(event['transactionHash'].hex())
    
txs
```
Get data from transaction hash:

```basg
result = web3.eth.get_transaction_receipt('0x65c31f92e6d8dd5a0bbc49edfd9448ec495cb0e6b47db8e3763b49a8744591d6')
    
result
```

Decoding data:

```basg
for event in result['logs']:
    if event['address'].lower() == contract_address.lower() and (event['topics'][0].hex()).lower() == transfer_signature_hash.lower():
        s = event
        
value = int(s['data'].hex(), 0)

if (s['topics'][2].hex()).lower() == (wallet_sign).lower():
    print(result['transactionHash'].hex(), ' -- Transfer', value, 'to address')
elif (s['topics'][1].hex()).lower() == (wallet_sign).lower():
    print(result['transactionHash'].hex(), ' -- Transfer', value, 'from address')
```
