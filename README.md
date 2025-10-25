# Four.meme Trading Bot

A TypeScript trading bot for four.meme tokens with automatic migration detection, dual exchange support, and volume trading capabilities.

## Features

- **Migration Detection**: Automatically detects if token has migrated to PancakeSwap
- **Four.meme Trading**: Buy/sell tokens before migration (using Four.meme contracts)
- **PancakeSwap Trading**: Buy/sell tokens after migration (using PancakeRouter)
- **Smart Routing**: Automatically chooses the correct exchange based on migration status
- **Volume Bot**: Automated multi-wallet volume trading with stealth funding
- **Stealth Mode**: Use StealthFund contract for anonymous wallet funding
- **Simple Setup**: Just install and run with environment variables

## 📞 Get in Touch

**Telegram**: [@0xMuseNine](https://t.me/xMuseNine)
**Discord**: [@0xMuseNine](https://discord.com/users/1274339638668038187)

💼 **Open to freelance projects, full-time positions, and consulting opportunities**

## Setup

1. **Install Dependencies**

   ```bash
   npm install
   ```

2. **Configure Environment**
   Create a `.env` file with your configuration:

   ```bash
   # Required
   PRIVATE_KEY=your_private_key_here
   RPC_URL=https://bsc-dataseed.binance.org
   TOKEN_ADDRESS=0x... # Token to trade

   # Contract Addresses (defaults provided)
   TOKEN_MANAGER2=0x5c952063c7fc8610FFDB798152D69F0B9550762b
   HELPER3_ADDRESS=0xF251F83e40a78868FcfA3FA4599Dad6494E46034
   PANCAKE_ROUTER_ADDRESS=0x10ED43C718714eb63d5aA57B78B54704E256024E
   WBNB_ADDRESS=0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c
   STEALTH_FUND_ADDRESS=your_stealth_address

   # Volume Bot Configuration
   TOTAL_BNB=18                    # Total BNB to use for volume trading
   MIN_DEPOSIT_BNB=0.01           # Minimum BNB per wallet
   MAX_DEPOSIT_BNB=0.01           # Maximum BNB per wallet
   MIN_BUY_WALLETS_NUM=3          # Minimum wallets per cycle
   MAX_BUY_WALLETS_NUM=5          # Maximum wallets per cycle
   MIN_SELL_WALLETS_NUM=0         # Minimum wallets to sell
   MAX_SELL_WALLETS_NUM=0         # Maximum wallets to sell
   MIN_PERCENT_SELL_AMOUNT_AFTER_BUY=50  # Min % of tokens to sell
   MAX_PERCENT_SELL_AMOUNT_AFTER_BUY=80  # Max % of tokens to sell
   CYCLE_LIMIT=3                  # Number of trading cycles
   MIN_PER_CYCLE_TIME=5           # Min seconds between cycles
   MAX_PER_CYCLE_TIME=15          # Max seconds between cycles
   MIN_DELAY_BUY=2000             # Min delay before buy (ms)
   MAX_DELAY_BUY=8000             # Max delay before buy (ms)
   MIN_DELAY_SELL=4000            # Min delay before sell (ms)
   MAX_DELAY_SELL=12000           # Max delay before sell (ms)
   GAS_BUFFER_BNB=0.001           # BNB buffer for gas fees

   # Optional Features
   STEALTH_MODE=true              # Use StealthFund for wallet funding
   USE_PANCAKE_AFTER_MIGRATION=true  # Auto-switch to PancakeSwap after migration
   ```

3. **Run the Bot**
   ```bash
   npm run dev
   ```

## How It Works

### Volume Bot Mode

The VolumeBot creates multiple random wallets and executes coordinated trading:

1. **Wallet Creation**: Generates random wallets for each trading cycle
2. **Funding**: Uses either direct transfers or StealthFund contract (stealth mode)
3. **Sequential Funding**: Funds wallets one by one to avoid nonce conflicts
4. **Parallel Trading**: Executes buy orders in parallel after funding
5. **Sell Strategy**: Randomly selects wallets to sell based on configuration
6. **Cycle Management**: Repeats process for specified number of cycles

### Migration Detection

The bot automatically detects the migration status of any four.meme token:

- **Before Migration**: Uses Four.meme contracts for trading
- **After Migration**: Uses PancakeSwap Router for trading

The bot checks the `liquidityAdded` status from the Helper3 contract:

- `false` = Token still on Four.meme (use Four.meme contracts)
- `true` = Token migrated to PancakeSwap (use PancakeRouter)

### Trading Flow

1. **Check Migration Status**: Determines if token has migrated
2. **Route to Correct Exchange**:
   - Four.meme → Uses `buyTokenAMAP` and `sellToken`
   - PancakeSwap → Uses `swapExactETHForTokens` and `swapExactTokensForETH`
3. **Execute Trade**: Handles approvals, timing, and transaction execution

### Stealth Mode

When `STEALTH_MODE=true`, the bot uses the StealthFund contract to fund wallets:

- Creates anonymous funding transactions
- Hides the connection between main wallet and trading wallets
- Uses `stealthMultipleFund` function for efficient batch funding

## Code Structure

### Main Components

- **`FourMemeTrader`**: Main trading class with migration detection
- **`VolumeBot`**: Automated multi-wallet volume trading bot
- **`getMigrationStatus()`**: Checks if token has migrated
- **`buyToken()`**: Four.meme buy (before migration)
- **`sellAmount()`**: Four.meme sell (before migration)
- **`buyPancakeToken()`**: PancakeSwap buy (after migration)
- **`sellPancakeToken()`**: PancakeSwap sell (after migration)
- **`fundStealthChild()`**: Stealth funding via StealthFund contract
- **`fundChild()`**: Direct wallet funding

### Example Usage

#### Basic Trading

```typescript
const trader = new FourMemeTrader();
const tokenAddress = "0x...";

// Check migration status
const migrated = await trader.getMigrationStatus(tokenAddress);

if (migrated) {
  // Token migrated to PancakeSwap
  await trader.buyPancakeToken(tokenAddress, 0.01); // Buy with 0.01 BNB
  await trader.sellPancakeToken(tokenAddress, 1000); // Sell 1000 tokens
} else {
  // Token still on Four.meme
  const result = await trader.buyToken(tokenAddress, 0.01);
  console.log(`Estimated tokens: ${result.estimatedTokens}`);
  await trader.sellAmount(tokenAddress, parseFloat(result.estimatedTokens));
}
```

#### Volume Bot Usage

```typescript
import VolumeBot from "./src/volume-bot";

const params = {
  totalBNB: 1.0, // Total BNB to use
  minDepositBNB: 0.01, // Min BNB per wallet
  maxDepositBNB: 0.05, // Max BNB per wallet
  minBuyWalletsNum: 3, // Min wallets per cycle
  maxBuyWalletsNum: 5, // Max wallets per cycle
  minSellWalletsNum: 1, // Min wallets to sell
  maxSellWalletsNum: 3, // Max wallets to sell
  minPercentSellAmountAfterBuy: 50, // Min % to sell
  maxPercentSellAmountAfterBuy: 80, // Max % to sell
  cyclesLimit: 3, // Number of cycles
  tokenAddress: "0x...", // Token to trade
  gasBufferBNB: 0.001, // Gas buffer
  // ... other parameters
};

const bot = new VolumeBot(params);
await bot.run();
```

## Environment Variables

### Required Variables

| Variable        | Description                     | Default  |
| --------------- | ------------------------------- | -------- |
| `PRIVATE_KEY`   | Your wallet private key         | Required |
| `RPC_URL`       | BSC RPC endpoint                | Required |
| `TOKEN_ADDRESS` | Token contract address to trade | Required |

### Contract Addresses

| Variable                 | Description                                | Default                                      |
| ------------------------ | ------------------------------------------ | -------------------------------------------- |
| `TOKEN_MANAGER2`         | Four.meme TokenManager contract            | `0x5c952063c7fc8610FFDB798152D69F0B9550762b` |
| `HELPER3_ADDRESS`        | Four.meme Helper3 contract                 | `0xF251F83e40a78868FcfA3FA4599Dad6494E46034` |
| `PANCAKE_ROUTER_ADDRESS` | PancakeSwap Router contract                | `0x10ED43C718714eb63d5aA57B78B54704E256024E` |
| `WBNB_ADDRESS`           | Wrapped BNB contract                       | `0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c` |
| `STEALTH_FUND_ADDRESS`   | StealthFund contract for anonymous funding | `your_stealth_address`                       |

### Volume Bot Configuration

| Variable                            | Description                         | Default |
| ----------------------------------- | ----------------------------------- | ------- |
| `TOTAL_BNB`                         | Total BNB to use for volume trading | `18`    |
| `MIN_DEPOSIT_BNB`                   | Minimum BNB per wallet              | `0.01`  |
| `MAX_DEPOSIT_BNB`                   | Maximum BNB per wallet              | `0.01`  |
| `MIN_BUY_WALLETS_NUM`               | Minimum wallets per cycle           | `3`     |
| `MAX_BUY_WALLETS_NUM`               | Maximum wallets per cycle           | `5`     |
| `MIN_SELL_WALLETS_NUM`              | Minimum wallets to sell             | `0`     |
| `MAX_SELL_WALLETS_NUM`              | Maximum wallets to sell             | `0`     |
| `MIN_PERCENT_SELL_AMOUNT_AFTER_BUY` | Min % of tokens to sell             | `50`    |
| `MAX_PERCENT_SELL_AMOUNT_AFTER_BUY` | Max % of tokens to sell             | `80`    |
| `CYCLE_LIMIT`                       | Number of trading cycles            | `3`     |
| `MIN_PER_CYCLE_TIME`                | Min seconds between cycles          | `5`     |
| `MAX_PER_CYCLE_TIME`                | Max seconds between cycles          | `15`    |
| `MIN_DELAY_BUY`                     | Min delay before buy (ms)           | `2000`  |
| `MAX_DELAY_BUY`                     | Max delay before buy (ms)           | `8000`  |
| `MIN_DELAY_SELL`                    | Min delay before sell (ms)          | `4000`  |
| `MAX_DELAY_SELL`                    | Max delay before sell (ms)          | `12000` |
| `GAS_BUFFER_BNB`                    | BNB buffer for gas fees             | `0.001` |

### Optional Features

| Variable                      | Description                                | Default |
| ----------------------------- | ------------------------------------------ | ------- |
| `STEALTH_MODE`                | Use StealthFund for wallet funding         | `true`  |
| `USE_PANCAKE_AFTER_MIGRATION` | Auto-switch to PancakeSwap after migration | `true`  |

## Security Notes

- Keep your private keys secure
- Use environment variables for sensitive data
- Test with small amounts first
- Consider using hardware wallets for large amounts

## Error Handling

Common errors and solutions:

- **"Disabled"**: Token has migrated, use PancakeSwap methods
- **"Insufficient BNB balance"**: Add more BNB to your wallet
- **"Insufficient token balance"**: You don't have enough tokens to sell
- **"Missing environment variable"**: Check your `.env` file
- **"Selling amount is 0 or greater than estimated tokens"**: Adjust sell percentage or check token balance
- **"Funding failed"**: Check StealthFund contract or increase gas price
- **"SKIPPED_LOW_DEPOSIT"**: Increase deposit amount or reduce gas buffer

## Wallet Logs

The VolumeBot creates a `wallets.json` file to track all trading activity:

```json
[
  {
    "index": 0,
    "address": "0x...",
    "privateKey": "0x...",
    "depositBNB": 0.01,
    "buyTxHash": "0x...",
    "sellTxHash": "0x...",
    "boughtVia": "fourmeme",
    "soldVia": "pancake",
    "estimatedTokens": "1000000",
    "timestamp": "2024-01-01T00:00:00.000Z"
  }
]
```

## Development

```bash
# Install dependencies
npm install

# Run the volume bot
npm run start

# Run gather from wallet.json
npm run gather

# Type checking
npx tsc --noEmit
```

## License

MIT
