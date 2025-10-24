# 🦄 Uniswap Arbitrage Analysis

## 0. Foreword

Uniswap is the most popular decentralized exchange (DEX) with more than **12,000 trading pairs** and over **$2 billion in liquidity**.
Wherever there is a market, there are **arbitrage opportunities**.

On Uniswap, we can trade token **A** for token **D** through a sequence of swaps, e.g., `A → B → C → D`.
Interestingly, **A and D can be the same token**, meaning we can trade `A → B → C → A`.
If executed correctly, it’s possible to end up with **more A than we started with** — that’s arbitrage.

This documentation explores:

1. How to find such profitable circular paths
2. How to determine the **optimal input amount** for maximum profit

---

## 1. Arbitrage Problem Analysis

To perform an arbitrage, we must solve two sub-problems:

1. **Path Finding** — how to find the best circular trading route
   (e.g., `A → ? → ? → … → A`)
2. **Optimal Input Amount** — how much of token A should be traded

---

### 1.1 Path Finding

Each token is treated as a **vertex** in a graph, and each trading pair as an **edge**.
Thus, finding arbitrage paths becomes a **cycle detection problem** in graph theory.

A **Depth-First Search (DFS)** approach efficiently explores all possible circular routes.
We can also limit the maximum path length (`maxHops`) to control gas usage.

#### Example: DFS-based Path Finder

```python
def findArb(pairs, tokenIn, tokenOut, maxHops, currentPairs, path, circles):
    for i in range(len(pairs)):
        newPath = path.copy()
        pair = pairs[i]

        # Continue only if tokenIn is in the current pair
        if not pair['token0']['address'] == tokenIn['address'] and not pair['token1']['address'] == tokenIn['address']:
            continue

        # Skip low-liquidity pairs
        if pair['reserve0']/pow(10, pair['token0']['decimal']) < 1 or pair['reserve1']/pow(10, pair['token1']['decimal']) < 1:
            continue

        # Determine next token in path
        if tokenIn['address'] == pair['token0']['address']:
            tempOut = pair['token1']
        else:
            tempOut = pair['token0']

        newPath.append(tempOut)

        # Found a cycle
        if tempOut['address'] == tokenOut['address'] and len(path) > 2:
            c = {'route': currentPairs, 'path': newPath}
            circles.append(c)

        # Continue searching deeper
        elif maxHops > 1 and len(pairs) > 1:
            pairsExcludingThisPair = pairs[:i] + pairs[i+1:]
            circles = findArb(
                pairsExcludingThisPair,
                tempOut,
                tokenOut,
                maxHops - 1,
                currentPairs + [pair],
                newPath,
                circles
            )

    return circles
```

---

### 1.2 Optimal Input Amount

Uniswap follows a **Constant Function Market Maker (CFMM)** model:

[
(R_0 + r \Delta a)(R_1 - \Delta b) = R_0 R_1
]

Where:

* ( R_0, R_1 ): reserves of tokens A and B
* ( \Delta a, \Delta b ): input and output amounts
* ( r = 1 - \text{fee} ): remaining ratio after swap fees

The reserves product remains constant during swaps — hence “constant function.”

---

#### Multi-Hop Optimization

For a circular path like `A → B → C → A`, we aim to maximize profit:

`[
\max (\Delta a' - \Delta a)
]`

subject to:
[
(R_0 + r\Delta a)(R_1 - \Delta b) = R_0R_1
]
[
(R_1' + r\Delta b)(R_2 - \Delta c) = R_1'R_2
]
[
(R_2' + r\Delta c)(R_3 - \Delta a') = R_2'R_3
]

For longer paths (`A → B → C → D → A`), we define **virtual pools** with parameters ( E_0, E_1 ), derived recursively from real pools.

#### Virtual Pool Formulas

[
E_0 = \frac{R_0 R_1'}{R_1' + R_1 r}
]
[
E_1 = \frac{r R_1 R_2}{R_1' + R_1 r}
]

When a circular path is collapsed to a virtual pool `A → A`,
if ( E_a < E_b ), an arbitrage opportunity exists.

The profit is:

[
f = \Delta a' - \Delta a
]
and the **optimal input** is obtained from:

[
\Delta a = \frac{E_a E_b r - E_a r}{E_b - E_a r}
]

---

### Code Example – Path Finding + Optimal Input Calculation

```python
def findArb(pairs, tokenIn, tokenOut, maxHops, currentPairs, path, bestTrades, count=5):
    for i in range(len(pairs)):
        newPath = path.copy()
        pair = pairs[i]

        # Check token inclusion
        if not pair['token0']['address'] == tokenIn['address'] and not pair['token1']['address'] == tokenIn['address']:
            continue

        # Ignore low-liquidity pairs
        if pair['reserve0']/pow(10, pair['token0']['decimal']) < 1 or pair['reserve1']/pow(10, pair['token1']['decimal']) < 1:
            continue

        # Determine output token
        if tokenIn['address'] == pair['token0']['address']:
            tempOut = pair['token1']
        else:
            tempOut = pair['token0']

        newPath.append(tempOut)

        # Cycle found
        if tempOut['address'] == tokenOut['address'] and len(path) > 2:
            Ea, Eb = getEaEb(tokenOut, currentPairs + [pair])
            newTrade = {'route': currentPairs + [pair], 'path': newPath, 'Ea': Ea, 'Eb': Eb}

            if Ea and Eb and Ea < Eb:
                newTrade['optimalAmount'] = getOptimalAmount(Ea, Eb)

                if newTrade['optimalAmount'] > 0:
                    newTrade['outputAmount'] = getAmountOut(newTrade['optimalAmount'], Ea, Eb)
                    newTrade['profit'] = newTrade['outputAmount'] - newTrade['optimalAmount']
                    newTrade['p'] = int(newTrade['profit']) / pow(10, tokenOut['decimal'])
                else:
                    continue

                bestTrades = sortTrades(bestTrades, newTrade)
                bestTrades.reverse()
                bestTrades = bestTrades[:count]

        # Continue deeper
        elif maxHops > 1 and len(pairs) > 1:
            pairsExcludingThisPair = pairs[:i] + pairs[i+1:]
            bestTrades = findArb(
                pairsExcludingThisPair,
                tempOut,
                tokenOut,
                maxHops - 1,
                currentPairs + [pair],
                newPath,
                bestTrades,
                count
            )
    return bestTrades
```

---

## 2. Implementation Notes

Each Ethereum block is produced approximately every **15 seconds**.
Within that interval, an arbitrage bot must:

1. **Update pair reserves**

   * For small datasets: batch request all pair states
   * For large datasets: parse new blocks for `Swap`, `Sync`, `Mint`, or `Burn` events

2. **Find the best path and optimal input**

   * Optimize the DFS or explore algorithms like **Bellman-Ford** or **SPFA**

3. **Send the transaction**

   * Always verify profitability via `UniswapV2Router02.getAmountsOut()` before execution

---

## 3. Conclusion

Uniswap arbitrage is a **highly competitive** field.
While profits are possible, **gas costs and latency** often erode opportunities.
However, decentralized finance (DeFi) remains a fertile ground for arbitrage,
especially across multiple DEXes like **Curve**, **Balancer**, or even cross-chain setups.

With **flash loans**, capital requirements can be eliminated —
but success depends on precision, speed, and smart optimization.


