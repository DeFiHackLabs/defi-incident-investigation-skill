# On-Chain Verification Reference

Multi-chain RPC patterns, Etherscan V2 API, and account type detection for Phase 4 verification.

## Etherscan V2 API

V1 is deprecated. Use V2 with `chainid` parameter:
```
https://api.etherscan.io/v2/api?chainid=1&module=...&apikey=KEY
```

Key extraction:
```bash
ETHERSCAN_KEY=$(grep ETHERSCAN_API_KEY ~/.zshenv | sed 's/.*=["'\'']\([^"'\'']*\)["'\''].*/\1/' | head -1)
```

### Common Queries

| Action | URL |
|--------|-----|
| Account txs | `module=account&action=txlist&address=0x...&startblock=X&endblock=Y&sort=asc` |
| Internal txs | `module=account&action=txlistinternal&address=0x...` |
| Token transfers | `module=account&action=tokentx&address=0x...` |
| Contract creation | `module=contract&action=getcontractcreation&contractaddresses=0x...` |
| Tx by hash | `module=proxy&action=eth_getTransactionByHash&txhash=0x...` |
| Tx receipt | `module=proxy&action=eth_getTransactionReceipt&txhash=0x...` |
| Block by number | `module=proxy&action=eth_getBlockByNumber&tag=0x...&boolean=true` |
| Code | `module=proxy&action=eth_getCode&address=0x...&tag=latest` |

**BSC:** V2 with `chainid=56` requires paid plan. Use public BSC RPC instead.

### Block by Timestamp
```
module=block&action=getblocknobytime&timestamp=1725926400&closest=before
```

## Public RPC Endpoints

### Ethereum
| Endpoint | Notes |
|----------|-------|
| `https://ethereum-rpc.publicnode.com` | Best for historical `eth_getCode` + `trace_transaction` |
| `https://eth.drpc.org` | Reliable for standard reads (403 on receipts) |
| `https://1rpc.io/eth` | Fast, standard JSON-RPC |
| `https://rpc.ankr.com/eth` | Good availability |

### BSC
| Endpoint | Notes |
|----------|-------|
| `https://bsc-dataseed.binance.org/` | Primary |
| `https://bsc-dataseed1.binance.org/` | Mirror |
| `https://bsc.publicnode.com` | Fallback |
| `https://rpc.ankr.com/bsc` | Additional |

### Arbitrum
| Endpoint | Notes |
|----------|-------|
| `https://arb1.arbitrum.io/rpc` | Official, reliable for reads |
| `https://arbitrum.drpc.org` | Secondary |
| Blockscout API | `https://arbitrum.blockscout.com/api/v2/addresses/{addr}/transactions` — no key needed |

### Other Chains
For Solana, TRON, Sui, etc. — To be added. Will be added in future self-improvement loop.

## JSON-RPC Verification Pattern

```python
import json, urllib.request, ssl, time

SSL_CTX = ssl.create_default_context()
SSL_CTX.check_hostname = False
SSL_CTX.verify_mode = ssl.CERT_NONE

def rpc_call(endpoint, method, params, retries=3):
    for attempt in range(retries):
        try:
            payload = json.dumps({"jsonrpc":"2.0","method":method,"params":params,"id":1}).encode()
            req = urllib.request.Request(endpoint, data=payload,
                headers={'Content-Type':'application/json','User-Agent':'Mozilla/5.0'})
            with urllib.request.urlopen(req, timeout=20, context=SSL_CTX) as r:
                return json.loads(r.read()).get("result")
        except Exception as e:
            if attempt < retries - 1:
                time.sleep(2 ** attempt)  # exponential backoff
            else:
                raise

# IMPORTANT: Add time.sleep(1) between sequential RPC calls to avoid HTTP 429.
# Public endpoints (especially 1rpc.io/eth) rate-limit after ~8-10 rapid calls.
# If 429 occurs, rotate to fallback: 1rpc.io/eth → rpc.ankr.com/eth → eth.drpc.org
```

## Verification Steps (per tx hash)

1. **`eth_getTransactionByHash`** — verify `from`, `to`, `blockNumber`, `input` selector, `value`
2. **`eth_getBlockByNumber`** — get `block.timestamp` (convert: `datetime.utcfromtimestamp(int(ts, 16))`)
3. **`eth_getTransactionReceipt`** — verify `status` (0x1=SUCCESS, 0x0=FAIL) and `logs`
4. **`eth_getCode`** for every address — distinguish EOA vs Contract (see table below)

## EOA vs Contract Detection

| `eth_getCode` result | Classification |
|----------------------|----------------|
| `"0x"` (len=2) | EOA |
| len > 4 | Contract |
| len ~48, starts with `0xef0100` | EIP-7702 delegated account (Contract) |
| len 344 | Gnosis Safe v1.3.0 proxy |
| len 4302 | OpenZeppelin ProxyAdmin |

**EIP-7702 detection:** `eth_getCode(eoa, "latest")` returns `"0x"` after attack window. Must snapshot at attack block height: `eth_getCode(eoa, hex(blockNumber))`. If result starts with `0xef0100`, the next 40 hex chars are the delegated contract address.

**EIP-7702 timing pitfall:** An attacker may be a plain EOA at the attack block but upgrade to EIP-7702 *after* the attack. Always check `eth_getCode` at BOTH `latest` AND `hex(attackBlock)`. The `accountType` in the report should reflect the state at the attack block. If they differ, note the post-attack upgrade in the description. See Pitfall #17 in `references/pitfalls/general.md`.

## Input Data Decoding

| Selector | Function | Decoding |
|----------|----------|----------|
| `0xa9059cbb` | `transfer(address,uint256)` | recipient at `input[34:74]`, amount at `input[74:138]` |
| `0x40c10f19` | `mint(address,uint256)` | same layout as transfer |
| `0x095ea7b3` | `approve(address,uint256)` | ERC-20 approval — detect in lure tx to identify open-allowance attacks |
| `0x42842e0e` | `safeTransferFrom(address,address,uint256)` | ERC-721 / ERC-1155 |
| `0x2e1a7d4d` | `withdraw(uint256)` | WETH withdrawal |
| `0x6a761202` | Gnosis Safe `execTransaction` | offline-signed Safe tx |

## ERC20 Identity Verification

```python
def verify_token(addr, endpoint):
    # symbol() = 0x95d89b41, name() = 0x06fdde03
    sym = rpc_call(endpoint, 'eth_call', [{'to': addr, 'data': '0x95d89b41'}, 'latest'])
    name = rpc_call(endpoint, 'eth_call', [{'to': addr, 'data': '0x06fdde03'}, 'latest'])
    # Decode ABI-encoded strings: skip 0x + 64 hex (offset) + 64 hex (length) + read data
```

## Block Range Scanning

Don't rely on report's listed tx hashes. Scan every block in the attack window to find the complete sequence:

```python
for block_num in range(start_block, end_block + 1):
    block = rpc_call(endpoint, 'eth_getBlockByNumber', [hex(block_num), True])
    txs = [tx for tx in block['transactions'] if tx['from'].lower() == attacker.lower()]
```

## Trace Transaction (Ethereum)

`trace_transaction` returns every internal call, delegatecall, staticcall. Only available on public RPCs with `trace_` namespace (e.g., https://eth.drpc.org).

```python
traces = rpc_call('https://eth.drpc.org', 'trace_transaction', ['0x<TX_HASH>'])
# Each trace has: action.from, action.to, action.value, action.callType, traceAddress
```

## TRON On-Chain Verification

TRON has no Etherscan-style API. Use `apilist.tronscanapi.com` (no auth):

| Endpoint | Purpose |
|----------|---------|
| `/api/account?address=T...` | Balance, tx count, token holdings |
| `/api/token_trc20/transfers?relatedAddress=T...&start=0&limit=50&direction=2` | TRC20 transfers (paginated 50/page) |
| `/api/transaction?sort=-timestamp&count=true&limit=50&start=0&address=T...` | Normal TRX txs |
| `/api/transaction-info?hash=<64-hex-no-0x>` | Tx detail with `contract_map` (identifies all named contracts in call stack) |

**Pagination trap:** `limit` is silently capped at 50. Always paginate by `start += 50`.

**Address format:** base58check, starts with `T`, 34 chars. No `0x` prefix for tx hashes.

**Account type:** Check `account_map` in transaction-info response. Addresses with no code entry = EOA. Named protocol contracts = Contract.

## Cross-Source Decision Hierarchy

1. **Direct state reads** (`eth_getCode`, `eth_call`, `trace_transaction`) — highest authority
2. **Transaction receipts and blocks** — second highest
3. **Etherscan labels** — convenient but sometimes wrong
4. **Blog posts / news articles** — lowest authority

## Known Router Addresses

### Ethereum
| Service | Address Prefix |
|---------|---------------|
| 1inch v4 | `0x1111111254fb6c44bac0bed2854e76f90643097d` |
| Uniswap v3 | `0xE592427A0AEce92De3Edee1F18E0157C05861564` |
| Uniswap v2 | `0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D` |
| dYdX SoloMargin | `0x1E0447b19BB6EcFdAe1e4AE1694b0C3659614e4e` |

### BSC
| Service | Address |
|---------|---------|
| 1inch v3 | `0x11111112542D85B3EF69AE05771c2dCCff4fAa26` |
| PancakeSwap v2 | `0x10ED43C718714eb63d5aA57B78B54704E256024E` |
| PancakeSwap v3 | `0x13f4EA83D0bd40E75C8222255bc855a974568Dd4` |
