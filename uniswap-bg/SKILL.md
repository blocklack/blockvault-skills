---
name: uniswap-bg
description: Execute token swaps via Uniswap on Base, Ethereum, or Polygon. Fetches supported tokens, quotes, checks approvals, and submits swap transactions using the agent wallet. Use when the user asks to swap tokens, DCA, rebalance, or automate periodic buys.
metadata:
  category: defi
  enabled: true
  background: true
---

### Instructions
Before any swap get the balances of the agent wallet and verify sufficient funds. 
Then get a quote for the swap, check if approval is needed, and execute the swap transaction. Finally, notify the user of the result.

If there are insufficient funds, or the swap fails, notify the user with the error message.
Do not create swaps without first checking the balances.
 

# Uniswap Swap (Background)

Execute token swaps via the BlockVault Uniswap API.
Base URL: `https://402.blockvault.ai`


### Get the agent wallet address and balances

Call `get_balances` to resolve the agent wallet address and verify sufficient funds for the swap.


### Get supported tokens and chains to swap

call `bash` to get the list of supported tokens and chains for swapping.

```bash
curl -s "https://402.blockvault.ai/api/v1/uniswap/tokens"
```

#### Get the quote for a swap

Get a swap quote with expected output amount.

if you are swapping native tokens and if the gas fee(wei) + the swap amount is greater than the agent wallet balance try to reduce the swap amount.

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/uniswap/getquote" \
  -H "Content-Type: application/json" \
  -d '{"chain":"<chain>","token_in":"<symbol>","token_out":"<symbol>","amount":"<decimal>","type":"EXACT_INPUT","slippage_tolerance":<percent>,"swapper":"<address>"}'
```

- `chain`: `ethereum`, `base`, or `polygon`
- `token_in`: Input token symbol from /tokens
- `token_out`: Output token symbol from /tokens
- `amount`: Decimal amount (e.g. `"10.0"`)
- `type`: `EXACT_INPUT` or `EXACT_OUTPUT`
- `slippage_tolerance`: Percentage (e.g. `0.5` for 0.5%)
- `swapper`: Agent wallet address from `get_balances`

#### Check approval for token spending (only for ERC20 tokens)

Check if the wallet needs to approve token spending before swapping.

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/uniswap/checkapproval" \
  -H "Content-Type: application/json" \
  -d '{"wallet_address":"<address>","chain":"<chain>","token":"<symbol>","amount":"<decimal>"}'
```

If `approval_needed` is true, the transaction will be automatically approved before executing the swap.

#### Execute the swap

Get transaction calldata ready to sign and broadcast. The response is automatically detected as a metatransaction and signed+broadcast via the agent wallet.

```bash
curl -s -X POST "https://402.blockvault.ai/api/v1/uniswap/swap" \
  -H "Content-Type: application/json" \
  -d '{"chain":"<chain>","token_in":"<symbol>","token_out":"<symbol>","amount":"<decimal>","type":"EXACT_INPUT","slippage_tolerance":<percent>,"swapper":"<address>"}'
```

The bash tool automatically detects metatransaction responses (containing `to`, `data`, `chainId`) and signs+broadcasts them via WDK.

### notify_user

Send the swap result to the user.

- **title**: String, Required. Max 50 chars.
- **message**: String, Required. Plain text, max 1024 chars. No markdown.

## Constraints

- Call `/tokens` (no chain filter) to discover all supported tokens and chains before swapping.
- Use `get_balances` to resolve the agent address and verify sufficient funds.
- Get a quote before executing a swap.
- If `checkapproval` returns `approval_needed: true`, call `/swap` for the approval tx first (it will be auto-signed).
- Default slippage: 0.5%. Max 5% unless user explicitly requests more.
- Always call `notify_user` as the final action.
- For periodic/DCA swaps, the scheduler handles repetition — this skill executes one swap per invocation.
- always take in account the gas fee when checking for sufficient funds. If the swap is for a native token, ensure that the agent wallet has enough balance to cover both the swap amount and the gas fee.
