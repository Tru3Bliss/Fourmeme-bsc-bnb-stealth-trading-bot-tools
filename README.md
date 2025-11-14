# Four.meme Trading Bot

A TypeScript trading bot for four.meme tokens with automatic migration detection, dual exchange support, and volume trading capabilities.

## Features

- **Migration Detection**: Automatically detects if token has migrated to PancakeSwap
- **Four.meme Trading**: Buy/sell tokens before migration (using Four.meme contracts)
- **PancakeSwap Trading**: Buy/sell tokens after migration (using PancakeRouter)
- **Smart Routing**: Automatically chooses the correct exchange based on migration status
- **Volume Bot**: Automated multi-wallet volume trading with stealth funding
- **Two-Phase Workflow**: Separate BNB distribution from trading (create wallets first, then trade)
- **Pause/Resume**: Save and resume trading cycles from where you left off
- **Wallet Reuse**: Continue using existing wallets after cycles complete
- **State Persistence**: Bot state saved automatically for seamless resumption
- **Stealth Mode**: Use StealthFund contract for anonymous wallet funding
- **Web UI**: User-friendly interface for bot control and monitoring
- **Simple Setup**: Just install and run with environment variables

## Setup

1. **Install Dependencies**
   ```bash
   npm install
   ```

2. **Configure Environment**
   Create a `.env` file with your configuration (see `.env.example`):
   ```bash
   # Required
   PRIVATE_KEY=0x...
   RPC_URL=https://bsc-dataseed.binance.org
   TOKEN_ADDRESS=0x... # Token to trade (optional for distribution phase)

   # Web UI
   UI_PORT=3000

   # Contract Addresses (defaults provided in code)
   PANCAKE_ROUTER_ADDRESS=0x10ED43C718714eb63d5aA57B78B54704E256024E
   WBNB_ADDRESS=0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c
   STEALTH_FUND_ADDRESS=0x3774b227aee720423a62710c7Ce2D70EA16eE0D0

   # Volume Bot Budget / Wallets
   TOTAL_BNB=18
   TOTAL_NUM_WALLETS=50
   MIN_DEPOSIT_BNB=0.01
   MAX_DEPOSIT_BNB=0.02

   # Per-cycle Buy Count
   MIN_BUY_NUM_PER_CYCLE=3
   MAX_BUY_NUM_PER_CYCLE=5

   # Buy amount as percent of (BNB - GAS_BUFFER_BNB)
   MIN_BUY_PERCENT_BNB=100
   MAX_BUY_PERCENT_BNB=100

   # Per-cycle Sell: percent of bought wallets to sell
   MIN_PERCENT_SELL=0
   MAX_PERCENT_SELL=0

   # Per-wallet Sell: percent of token balance to sell
   MIN_PERCENT_SELL_AMOUNT_AFTER_BUY=50
   MAX_PERCENT_SELL_AMOUNT_AFTER_BUY=80

   # Cycle control
   CYCLE_LIMIT=3
   MIN_PER_CYCLE_TIME=5000
   MAX_PER_CYCLE_TIME=10000

   # Timing
   MIN_DELAY_BUY=2000
   MAX_DELAY_BUY=8000
   MIN_DELAY_SELL=4000
   MAX_DELAY_SELL=12000

   # Gas buffer (BNB left for gas)
   GAS_BUFFER_BNB=0.001

   # Features
   STEALTH_MODE=false
   USE_PANCAKE_AFTER_MIGRATION=true
   
   # Distribution Mode (CLI only - UI uses buttons)
   DISTRIBUTE_ONLY=false  # Set to true for distribution-only mode via CLI
   ```

3. **Run the Bot**

   **Option 1: Web UI (Recommended)**
   ```bash
   npm run ui
   ```
   Then open http://localhost:3000 in your browser
   
   The UI provides:
   - **💰 Distribute BNB**: Create wallets and fund them (no token address needed)
   - **▶ Start Trading**: Begin trading cycles (requires token address)
   - **⏸ Pause Bot**: Gracefully pause and save state for resumption
   - **📥 Gather**: Collect funds from wallets back to main wallet
   - **Real-time Logs**: View bot activity and gather process logs
   - **State Display**: See saved state for resuming cycles

   **Option 2: Command Line**
   ```bash
   # Distribution only (Phase 1)
   DISTRIBUTE_ONLY=true npm run start
   
   # Trading mode (Phase 2 - requires TOKEN_ADDRESS)
   DISTRIBUTE_ONLY=false npm run start
   
   # Gather funds from wallets
   npm run gather
   ```

## How It Works

### Volume Bot Mode

The VolumeBot operates in two phases for optimal workflow:

#### Phase 1: Distribution (No Token Address Required)
1. **Wallet Creation**: Generates random wallets up to `TOTAL_NUM_WALLETS`
2. **BNB Distribution**: Funds wallets with random amounts between `MIN_DEPOSIT_BNB` and `MAX_DEPOSIT_BNB`
3. **Funding Method**: Uses either direct transfers or StealthFund contract (stealth mode)
4. **Sequential Funding**: Funds wallets one by one to avoid nonce conflicts
5. **State Check**: Verifies actual blockchain balances (not just stored values)

#### Phase 2: Trading (Requires Token Address)
1. **Wallet Reuse**: Uses existing funded wallets (or funds new ones if needed)
2. **Token Approval**: Approves token for trading on all wallets
3. **Cycle Execution**: Runs trading cycles with buy/sell operations
4. **State Persistence**: Saves progress after each cycle for pause/resume
5. **Sell Strategy**: Randomly selects wallets to sell based on configuration
6. **Cycle Management**: Repeats process for specified number of cycles

#### Pause/Resume Feature
- Bot state is automatically saved to `bot-state.json` after each cycle
- When paused, state includes: current cycle, total cycles, token address, timestamps
- Resume by clicking "START TRADING" - bot continues from saved cycle
- State is cleared automatically when all cycles complete

#### Wallet Reuse
- After gather or cycle completion, wallets are preserved in `wallets.json`
- Bot checks actual blockchain balances (not just stored `depositBNB` field)
- Can change settings and continue with same wallets
- No need to delete `wallets.json` to run again

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
const tokenAddress = '0x...';

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
import VolumeBot from './src/volume-bot';

const params = {
  totalBNB: 1.0,                    // Total BNB to use
  minDepositBNB: 0.01,             // Min BNB per wallet
  maxDepositBNB: 0.05,             // Max BNB per wallet
  minBuyWalletsNum: 3,             // Min wallets per cycle
  maxBuyWalletsNum: 5,             // Max wallets per cycle
  minSellWalletsNum: 1,            // Min wallets to sell
  maxSellWalletsNum: 3,            // Max wallets to sell
  minPercentSellAmountAfterBuy: 50, // Min % to sell
  maxPercentSellAmountAfterBuy: 80, // Max % to sell
  cyclesLimit: 3,                  // Number of cycles
  tokenAddress: '0x...',           // Token to trade
  gasBufferBNB: 0.001,            // Gas buffer
  // ... other parameters
};

const bot = new VolumeBot(params);
await bot.run();
```

## Environment Variables

### Required Variables
| Variable | Description | Default |
|----------|-------------|---------|
| `PRIVATE_KEY` | Your wallet private key | Required |
| `RPC_URL` | BSC RPC endpoint | Required |
| `TOKEN_ADDRESS` | Token contract address to trade | Required |

### Contract Addresses
| Variable | Description | Default |
|----------|-------------|---------|
| `PANCAKE_ROUTER_ADDRESS` | PancakeSwap Router contract | `0x10ED43C718714eb63d5aA57B78B54704E256024E` |
| `WBNB_ADDRESS` | Wrapped BNB contract | `0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c` |
| `STEALTH_FUND_ADDRESS` | StealthFund contract for anonymous funding | `0x3774b227aee720423a62710c7Ce2D70EA16eE0D0` |

### Volume Bot Configuration
| Variable | Description | Default |
|----------|-------------|---------|
| `TOTAL_BNB` | Total BNB to use for volume trading | `18` |
| `TOTAL_NUM_WALLETS` | Total wallets to create and deposit into | `50` |
| `MIN_DEPOSIT_BNB` | Minimum BNB per wallet | `0.01` |
| `MAX_DEPOSIT_BNB` | Maximum BNB per wallet | `0.02` |
| `MIN_BUY_NUM_PER_CYCLE` | Min wallets to buy per cycle | `3` |
| `MAX_BUY_NUM_PER_CYCLE` | Max wallets to buy per cycle | `5` |
| `MIN_BUY_PERCENT_BNB` | Min % of available BNB to spend | `100` |
| `MAX_BUY_PERCENT_BNB` | Max % of available BNB to spend | `100` |
| `MIN_PERCENT_SELL` | Min % of bought wallets to sell per cycle | `0` |
| `MAX_PERCENT_SELL` | Max % of bought wallets to sell per cycle | `0` |
| `MIN_PERCENT_SELL_AMOUNT_AFTER_BUY` | Min % of tokens to sell | `50` |
| `MAX_PERCENT_SELL_AMOUNT_AFTER_BUY` | Max % of tokens to sell | `80` |
| `CYCLE_LIMIT` | Number of trading cycles | `3` |
| `MIN_PER_CYCLE_TIME` | Min ms between cycles | `5000` |
| `MAX_PER_CYCLE_TIME` | Max ms between cycles | `10000` |
| `MIN_DELAY_BUY` | Min delay before buy (ms) | `2000` |
| `MAX_DELAY_BUY` | Max delay before buy (ms) | `8000` |
| `MIN_DELAY_SELL` | Min delay before sell (ms) | `4000` |
| `MAX_DELAY_SELL` | Max delay before sell (ms) | `12000` |
| `GAS_BUFFER_BNB` | BNB buffer for gas fees | `0.001` |

### Optional Features
| Variable | Description | Default |
|----------|-------------|---------|
| `STEALTH_MODE` | Use StealthFund for wallet funding | `true` |
| `USE_PANCAKE_AFTER_MIGRATION` | Auto-switch to PancakeSwap after migration | `true` |
| `DISTRIBUTE_ONLY` | Distribution-only mode (CLI only, UI uses buttons) | `false` |

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
- **"Wallets are already funded"**: Bot detected wallets have sufficient balance (check actual balances, not just depositBNB field)
- **"Token address is required for trading"**: Set `TOKEN_ADDRESS` in `.env` or use UI to enter it before starting trading

## Workflow Examples

### Example 1: Two-Phase Workflow (Recommended)

1. **Phase 1 - Distribution**:
   ```bash
   # Via UI: Click "💰 Distribute BNB"
   # Or via CLI:
   DISTRIBUTE_ONLY=true npm run start
   ```
   - Creates wallets and distributes BNB
   - No token address needed
   - Wait for distribution to complete

2. **Phase 2 - Trading**:
   ```bash
   # Via UI: Enter token address, then click "▶ Start Trading"
   # Or via CLI:
   # Set TOKEN_ADDRESS in .env first
   npm run start
   ```
   - Uses existing funded wallets
   - Starts trading cycles immediately

### Example 2: Pause and Resume

1. Start trading via UI or CLI
2. Click "⏸ Pause Bot" (or send SIGTERM signal)
3. Bot saves state to `bot-state.json` after current cycle
4. Later, click "▶ Start Trading" again
5. Bot resumes from saved cycle automatically

### Example 3: Reuse Wallets After Cycles Complete

1. Run bot with cycles (e.g., 3 cycles)
2. All cycles complete, state is cleared
3. Change settings (e.g., different token, more cycles)
4. Click "▶ Start Trading" again
5. Bot reuses existing wallets from `wallets.json`
6. No need to delete `wallets.json` or redistribute BNB

## State Files

### wallets.json

The VolumeBot creates a `wallets.json` file to track all wallet activity:

```json
[
  {
    "index": 0,
    "address": "0x...",
    "privateKey": "0x...",
    "depositBNB": 0.01,
    "status": "DEPOSITED",
    "lastBNB": 0.01,
    "buyTxHash": "0x...",
    "sellTxHash": "0x...",
    "boughtVia": "fourmeme",
    "soldVia": "pancake",
    "estimatedTokens": "1000000",
    "timestamp": "2024-01-01T00:00:00.000Z"
  }
]
```

**Note**: The `depositBNB` field persists even after gather. The bot checks actual blockchain balances when determining if wallets need funding.

### bot-state.json

The bot automatically saves state to `bot-state.json` for pause/resume functionality:

```json
{
  "currentCycle": 2,
  "totalCycles": 3,
  "startedAt": "2024-01-01T00:00:00.000Z",
  "lastUpdated": "2024-01-01T01:00:00.000Z",
  "tokenAddress": "0x..."
}
```

- **currentCycle**: Next cycle to run (resume point)
- **totalCycles**: Total cycles configured
- **startedAt**: When trading session started
- **lastUpdated**: Last state save timestamp
- **tokenAddress**: Token being traded

State is automatically cleared when all cycles complete.

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
