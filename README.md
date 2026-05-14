# Building a Polygon MEV Arbitrage Bot in Python

> A practical, production-ready guide to detecting and executing DEX arbitrage on Polygon.
> Includes real working code, RPC setup, and gas optimization strategies.
>
> **Tips / hire:** `0xa66c92bcb095533ed878fc30a4cbd24dc8edde93` (EVM)
> **TRC20:** `TEwbbfoUtQTTfQFFD6fbLcnSD7tdrdpRx6`
> **Solana:** `BvXqSW5Fwc6LMTyJopbRkQPLYDQFV9hEfR5sMthq73m8`

---

## Table of Contents

1. [How DEX Arbitrage Works](#1-how-dex-arbitrage-works)
2. [Architecture Overview](#2-architecture-overview)
3. [Prerequisites and Setup](#3-prerequisites-and-setup)
4. [RPC Connection Layer](#4-rpc-connection-layer)
5. [Pair Discovery via Factory Contracts](#5-pair-discovery-via-factory-contracts)
6. [Price and Liquidity Scanning](#6-price-and-liquidity-scanning)
7. [Spread Detection](#7-spread-detection)
8. [Profit Calculation](#8-profit-calculation)
9. [Execution and Slippage](#9-execution-and-slippage)
10. [Monitoring Mode](#10-monitoring-mode)
11. [Full Working Code](#11-full-working-code)
12. [Running in Production](#12-running-in-production)

---

## 1. How DEX Arbitrage Works

DEX arbitrage exploits **price differences for the same token pair across different exchanges**. Example:

- MATIC/USDT on QuickSwap: 1 MATIC = 0.0959 USDT
- MATIC/USDT on SushiSwap: 1 MATIC = 0.0957 USDT

**Spread = 0.20%** — buy on SushiSwap, sell on QuickSwap, profit the difference minus gas.

For a trade to be profitable, the spread must exceed:
- Gas cost (Polygon: ~0.01-0.05 MATIC per tx)
- DEX fees (QuickSwap: 0.30%, SushiSwap: 0.25%)
- Slippage (price impact from your trade)

### MEV vs. Manual Arbitrage

| Approach | Capital Needed | Speed | Complexity |
|----------|---------------|-------|------------|
| Manual arbitrage | Yes (for swaps) | Seconds | Low |
| Flash loan arbitrage | No (borrowed) | Same block | High |
| Sandwich attack | No (front-run) | Same block | High |
| Back-running | No | Same block | Medium |

This tutorial focuses on **manual arbitrage** (the simplest, most accessible approach) and lays the groundwork for flash loan strategies.

---

## 2. Architecture Overview

```
┌──────────────────────────────────────────────┐
│                  Scanner                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ QuickSwap│  │ SushiSwap│  │  ...DEX  │   │
│  │ Factory  │  │ Factory  │  │  Factory │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
│       │              │              │         │
│  ┌────▼──────────────▼──────────────▼─────┐   │
│  │          Pair Scanner                  │   │
│  │  (getReserves, prices, liquidity)      │   │
│  └────────────────┬───────────────────────┘   │
│                   │                            │
│  ┌────────────────▼───────────────────────┐   │
│  │        Spread Analyzer                 │   │
│  │  (compare prices, calc profit,         │   │
│  │   check thresholds)                    │   │
│  └────────────────┬───────────────────────┘   │
│                   │                            │
│  ┌────────────────▼───────────────────────┐   │
│  │          Execution Engine              │   │
│  │  (build tx, estimate gas, submit)      │   │
│  └────────────────────────────────────────┘   │
└──────────────────────────────────────────────┘
```

---

## 3. Prerequisites and Setup

### Requirements

```bash
# Python 3.10+
python -m pip install requests web3

# Configure RPC for Polygon
# Recommended: https://polygon-bor.publicnode.com (free, no API key)
```

### Polygon DEX Addresses (as of 2026)

```python
DEXES = {
    "QuickSwap": {
        "router": "0xa5E0829CaCEd8fFDD4De3c43696c57F7D7A678ff",
        "factory": "0x5757371414417b8C6CAad45bAeF941aBc7d3Ab32",
        "fee": 0.0030,  # 0.30%
    },
    "SushiSwap": {
        "router": "0x1b02dA8Cb0d097eB8D57A175b88c7D8b47997506",
        "factory": "0xc35DADB65012eC5796536bD9864eD8773aBc74C4",
        "fee": 0.0025,  # 0.25%
    },
}
```

### Key Token Addresses on Polygon

```python
TOKENS = {
    "WMATIC": "0x0d500B1d8E8eF31E21C99d1Db9A6444d3ADf1270",
    "USDC":   "0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359",
    "USDT":   "0xc2132D05D31c914a87C6611C10748AEb04B58e8F",
    "WETH":   "0x7ceB23fD6bC0adD59E62ac25578270cFf1b9f619",
    "DAI":    "0x8f3Cf7ad23Cd3CaDbD9735AFf958023239c6A063",
    "WBTC":   "0x1bfd67037b42cf73acF2047067bd4F2C47D9BfD6",
}
```

---

## 4. RPC Connection Layer

Polygon RPCs need special middleware for the `extraData` field:

```python
from web3 import Web3
from web3.middleware import extra_to_flat_middleware, geth_poa_middleware

RPC_URL = "https://polygon-bor.publicnode.com"

def create_web3():
    """Create a Web3 instance with Polygon POA middleware."""
    w3 = Web3(Web3.HTTPProvider(RPC_URL, request_kwargs={"timeout": 30}))
    w3.middleware_onion.inject(geth_poa_middleware, layer=0)
    # Flatten extraData to handle Polygon's 67-byte blocks
    w3.middleware_onion.inject(extra_to_flat_middleware, layer=0)
    return w3
```

### Health Check

Always add a health check with fallback:

```python
def check_rpc_health(w3) -> bool:
    try:
        block = w3.eth.block_number
        gas = w3.eth.gas_price
        logger.info(f"RPC OK — block {block}, gas {w3.from_wei(gas, 'gwei'):.1f} gwei")
        return True
    except Exception as e:
        logger.error(f"RPC failed: {e}")
        return False
```

---

## 5. Pair Discovery via Factory Contracts

Uniswap V2 factories map (tokenA, tokenB) → pair contract via `getPair`:

```python
def get_pair_address(factory_address, token_a, token_b):
    """Get pair contract address from factory using CREATE2."""
    factory = w3.eth.contract(
        address=factory_address,
        abi=[{
            "constant": True,
            "inputs": [
                {"name": "tokenA", "type": "address"},
                {"name": "tokenB", "type": "address"}
            ],
            "name": "getPair",
            "outputs": [{"name": "", "type": "address"}],
            "type": "function"
        }]
    )
    return factory.functions.getPair(token_a, token_b).call()
```

### Pair ABI for Reading Reserves

```python
PAIR_ABI = [
    {
        "constant": True,
        "inputs": [],
        "name": "getReserves",
        "outputs": [
            {"name": "_reserve0", "type": "uint112"},
            {"name": "_reserve1", "type": "uint112"},
            {"name": "_blockTimestampLast", "type": "uint32"}
        ],
        "type": "function"
    },
    {
        "constant": True,
        "inputs": [],
        "name": "token0",
        "outputs": [{"name": "", "type": "address"}],
        "type": "function"
    },
    {
        "constant": True,
        "inputs": [],
        "name": "token1",
        "outputs": [{"name": "", "type": "address"}],
        "type": "function"
    }
]
```

---

## 6. Price and Liquidity Scanning

```python
def scan_pair(w3, pair_address, token0_symbol, token1_symbol, dex_name, base_token):
    """Scan a single pair and return price data."""
    try:
        pair = w3.eth.contract(address=pair_address, abi=PAIR_ABI)
        reserves = pair.functions.getReserves().call()
        token0 = pair.functions.token0().call()

        reserve0, reserve1 = reserves[0], reserves[1]
        if reserve0 == 0 or reserve1 == 0:
            return None

        # Calculate price based on token order
        if token0.lower() == base_token.lower():
            price = reserve1 / reserve0
            liquidity = (reserve0 * 2) * price  # Total liquidity in USD
        else:
            price = reserve0 / reserve1
            liquidity = (reserve1 * 2) * price

        return {
            "dex": dex_name,
            "pair": f"{token0_symbol}/{token1_symbol}",
            "price": price,
            "liquidity": liquidity,
            "reserve0": reserve0,
            "reserve1": reserve1,
        }
    except Exception as e:
        logger.error(f"Error scanning {dex_name} {token0_symbol}/{token1_symbol}: {e}")
        return None
```

---

## 7. Spread Detection

The core logic: find which DEX has the lowest and highest price for each pair.

```python
def find_arbitrage_opportunities(prices):
    """Find arbitrage opportunities from scanned prices."""
    opportunities = []

    # Group prices by pair
    pairs = {}
    for p in prices:
        if p is None:
            continue
        key = p["pair"]
        if key not in pairs:
            pairs[key] = []
        pairs[key].append(p)

    for pair, dex_prices in pairs.items():
        if len(dex_prices) < 2:
            continue

        # Find min and max
        min_price = min(dex_prices, key=lambda x: x["price"])
        max_price = max(dex_prices, key=lambda x: x["price"])

        # Calculate spread
        spread = (max_price["price"] - min_price["price"]) / min_price["price"] * 100

        # Net spread after fees
        total_fees = sum(DEXES[d["dex"]]["fee"] for d in dex_prices)
        net_spread = spread - total_fees * 100

        opportunities.append({
            "pair": pair,
            "spread": round(spread, 3),
            "net_spread": round(net_spread, 3),
            "buy_dex": min_price["dex"],
            "sell_dex": max_price["dex"],
            "buy_price": min_price["price"],
            "sell_price": max_price["price"],
            "min_liquidity": min(d["liquidity"] for d in dex_prices),
        })

    opportunities.sort(key=lambda x: x["net_spread"], reverse=True)
    return opportunities
```

---

## 8. Profit Calculation

Realistic profit calculation must account for slippage:

```python
def calculate_profit(opportunity, trade_amount_eth=1.0):
    """
    Calculate expected profit after fees and slippage.

    Args:
        opportunity: from find_arbitrage_opportunities()
        trade_amount_eth: amount to trade (in ETH equivalent)
    """
    # Uniswap V2 constant product: k = x * y
    # Price impact = (amount_in * fee) / (reserve_in + amount_in * fee)

    buy_price = opportunity["buy_price"]
    sell_price = opportunity["sell_price"]

    # Simplified: assume 0.5% slippage per leg for reasonable trade sizes
    slippage_per_leg = 0.005
    total_slippage = slippage_per_leg * 2
    total_fees = 0.003 + 0.0025  # QuickSwap + SushiSwap

    cost = total_slippage + total_fees
    net_spread = (sell_price - buy_price) / buy_price - cost

    profit_pct = net_spread * 100
    profit_eth = net_spread * trade_amount_eth

    return {
        "profit_pct": round(profit_pct, 4),
        "profit_eth": round(profit_eth, 6),
        "gas_cost_eth": 0.0005,  # Estimated
        "net_profit": round(profit_eth - 0.0005, 6),
    }
```

---

## 9. Execution and Slippage

When a profitable opportunity is found, the swap is executed via the DEX router:

```python
ROUTER_ABI = [
    {
        "inputs": [
            {"name": "amountIn", "type": "uint256"},
            {"name": "amountOutMin", "type": "uint256"},
            {"name": "path", "type": "address[]"},
            {"name": "to", "type": "address"},
            {"name": "deadline", "type": "uint256"}
        ],
        "name": "swapExactTokensForTokens",
        "outputs": [{"name": "amounts", "type": "uint256[]"}],
        "type": "function"
    },
    {
        "inputs": [],
        "name": "WETH",
        "outputs": [{"name": "", "type": "address"}],
        "type": "function"
    }
]


def build_swap_transaction(w3, router_address, token_in, token_out,
                           amount_in, min_amount_out, account):
    """Build a swap transaction with proper gas estimation."""
    router = w3.eth.contract(address=router_address, abi=ROUTER_ABI)

    # Set deadline 5 minutes from now
    deadline = w3.eth.get_block('latest')['timestamp'] + 300

    tx = router.functions.swapExactTokensForTokens(
        amount_in,
        min_amount_out,
        [token_in, token_out],
        account,
        deadline
    ).build_transaction({
        'from': account,
        'gas': 300000,
        'gasPrice': w3.eth.gas_price,
        'nonce': w3.eth.get_transaction_count(account),
    })

    return tx
```

---

## 10. Monitoring Mode

For continuous scanning:

```python
async def monitor_loop(interval=12):
    """Continuously scan for opportunities."""
    w3 = create_web3()

    while True:
        try:
            if not check_rpc_health(w3):
                logger.warning("RPC down, retrying in 30s...")
                await asyncio.sleep(30)
                continue

            prices = scan_all_pairs(w3)
            opportunities = find_arbitrage_opportunities(prices)

            if opportunities and opportunities[0]["net_spread"] > 0.3:
                opp = opportunities[0]
                profit = calculate_profit(opp)
                logger.info(
                    f"🔥 PROFITABLE: {opp['pair']} | "
                    f"Spread: {opp['spread']}% | "
                    f"Net: {profit['profit_pct']}% | "
                    f"Profit: {profit['net_profit']} ETH"
                )
            else:
                best = opportunities[0] if opportunities else None
                if best:
                    logger.debug(f"Best: {best['pair']} @ {best['net_spread']}%")

        except Exception as e:
            logger.error(f"Monitor error: {e}")

        await asyncio.sleep(interval)
```

---

## 11. Full Working Code

A complete working implementation is available here:
[github.com/jiezishu000/polygon-dex-arb-bot](https://github.com/jiezishu000/polygon-dex-arb-bot)

The bot supports:
- **scan**: One-shot scan of all configured pairs
- **monitor**: Continuous scanning every 12 seconds
- **balances**: Check wallet balances
- **status**: Show connection status
- **trade**: Execute a swap (requires private key)

### Quick Start

```bash
# 1. Install
pip install web3 requests

# 2. Configure
# Create .env:
#   WALLET_KEY=your_private_key (leave empty for read-only)
#   WALLET_ADDRESS=0x...
#   RPC_URL=https://polygon-bor.publicnode.com

# 3. Run
python polygon_swapper.py scan
python polygon_swapper.py monitor
```

---

## 12. Running in Production

### Capital Requirements

| Strategy | Min Capital | Expected Return/Month |
|----------|-------------|----------------------|
| Manual arb (MATIC pairs) | 100 MATIC | 2-5% |
| Manual arb (ETH pairs) | 0.5 ETH | 1-3% |
| Flash loan arb | 0 ETH (gas only) | Variable |

### Gas Optimization

```python
# Polygon gas pricing (as of May 2026)
GAS_STRATEGIES = {
    "slow":     {"multiplier": 1.0, "wait": 30},   # ~30 gwei
    "standard": {"multiplier": 1.2, "wait": 15},   # ~35 gwei
    "fast":     {"multiplier": 1.5, "wait": 5},    # ~42 gwei
    "flash":    {"multiplier": 2.0, "wait": 1},    # ~56 gwei
}
```

### Risk Management

1. **Always simulate before executing**: Use `eth_call` to verify the swap succeeds
2. **Set slippage tolerance**: Start with 0.5%, tighten to 0.3% as you gain confidence
3. **Don't trade thin pools**: Skip pairs with less than $10K liquidity
4. **Monitor gas prices**: If gas > 200 gwei on Polygon, wait
5. **Keep a reserve**: Never use more than 50% of your wallet for a single trade

### Common Pitfalls

| Issue | Symptom | Fix |
|-------|---------|-----|
| Stale RPC | Wrong prices | Use multiple RPCs |
| Low liquidity | High slippage | Skip pairs < $50K |
| Gas spike | Tx pending forever | Use fast gas or cancel |
| Nonce mismatch | Tx rejected | Track nonce locally |
| Token approval | Swap failed | Approve before trade |

---

## Conclusion

This is the foundation of a production MEV arbitrage bot. Key takeaways:

1. **Price scanning is the easy part** — the real challenge is execution speed and gas optimization
2. **Spread thresholds matter** — on Polygon, you need >0.3% net spread to be profitable after gas
3. **Start with manual mode** — monitor for weeks before automating execution
4. **The best arb opportunities last seconds** — speed comes from RPC choice and transaction construction

---

*If this tutorial helped you, consider supporting further development:*

**EVM**: `0xa66c92bcb095533ed878fc30a4cbd24dc8edde93`
**TRC20**: `TEwbbfoUtQTTfQFFD6fbLcnSD7tdrdpRx6`
**Solana**: `BvXqSW5Fwc6LMTyJopbRkQPLYDQFV9hEfR5sMthq73m8`

*Need a custom bot or help with your strategy? Same addresses for 1 USDT consulting.*
