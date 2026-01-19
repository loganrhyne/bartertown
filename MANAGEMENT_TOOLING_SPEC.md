# Management and Observability Tooling Specification

**Version:** 1.0
**Date:** 2026-01-19
**Status:** Planning Specification
**System:** Congressional Trading Follower - Management Layer
**Related:** [RISK_FRAMEWORK_SPEC.md](RISK_FRAMEWORK_SPEC.md)

## Executive Summary

This document specifies management and observability tooling for the congressional trading follower system. The tooling provides operators with comprehensive control over strategy execution, risk management, performance monitoring, learning progress tracking, A/B test management, and emergency interventions.

The design follows Gas Town CLI patterns (Cobra-based, grouped commands) and web dashboard architecture (htmx-based real-time updates). All tools integrate with the Risk Management Framework defined in RISK_FRAMEWORK_SPEC.md.

## 1. Design Principles

### 1.1 Core Principles

1. **CLI-First Design**: All operations accessible via `gt` CLI commands for scriptability and automation
2. **Web Dashboard Secondary**: Rich visual interfaces for monitoring and analysis, but never required
3. **Real-Time Updates**: Live data streams for critical metrics (positions, P&L, risk metrics)
4. **Safety by Default**: Destructive operations require confirmation, emergency controls are always available
5. **Audit Trail**: All configuration changes and manual interventions are logged
6. **Progressive Disclosure**: Simple commands for common tasks, advanced flags for power users

### 1.2 Integration Points

```
┌─────────────────────────────────────────────────────────────┐
│                    Management Layer                          │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   CLI Tools  │  │  Web Dashboard│ │  Alert System│     │
│  │   (gt risk)  │  │   (port 8080) │ │  (Slack/Mail)│     │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │
│         │                  │                  │              │
├─────────┼──────────────────┼──────────────────┼─────────────┤
│         │                  │                  │              │
│  ┌──────▼──────────────────▼──────────────────▼───────┐    │
│  │         Risk Management Framework                   │    │
│  │  - Limit Enforcement  - Metrics Calculation         │    │
│  │  - Configuration      - Alert Generation            │    │
│  └──────┬──────────────────────────────────────────────┘    │
│         │                                                    │
├─────────┼────────────────────────────────────────────────────┤
│         │                                                    │
│  ┌──────▼──────────────────────────────────────────────┐   │
│  │           Strategy Execution Layer                   │   │
│  │  - Strategy Instances  - Order Generation            │   │
│  │  - Signal Processing   - Position Management         │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## 2. CLI Command Structure

All management commands are organized under the `gt risk` namespace to avoid conflicts with existing Gas Town commands.

### 2.1 Command Hierarchy

```
gt risk                          # Risk management commands
├── status                       # Show current risk status
├── limits                       # Manage risk limits
│   ├── list                    # List all limits
│   ├── get <limit-name>        # Get specific limit value
│   ├── set <limit> <value>     # Update limit (requires confirmation)
│   └── reset                   # Reset to defaults
├── config                       # Configuration management
│   ├── show                    # Show current config
│   ├── edit                    # Open config in $EDITOR
│   ├── validate                # Validate config file
│   ├── reload                  # Hot-reload config
│   └── history                 # Show config change history
├── strategy                     # Strategy control
│   ├── list                    # List all strategies
│   ├── show <id>               # Show strategy details
│   ├── pause <id>              # Pause strategy
│   ├── resume <id>             # Resume strategy
│   ├── halt <id>               # Emergency halt strategy
│   └── metrics <id>            # Show strategy metrics
├── portfolio                    # Portfolio operations
│   ├── status                  # Portfolio summary
│   ├── positions               # List all positions
│   ├── exposure                # Show exposure breakdown
│   ├── risk                    # Risk metrics (VaR, volatility, etc.)
│   └── liquidate               # Emergency liquidation (with confirmation)
├── learning                     # Learning progress tracking
│   ├── status                  # Current phase and progress
│   ├── phase                   # Phase management
│   │   ├── show               # Show current phase
│   │   ├── advance            # Manually advance phase
│   │   └── history            # Phase transition history
│   ├── metrics                 # Learning metrics
│   └── experiments             # List active experiments
├── ab-test                      # A/B test management
│   ├── list                    # List all tests
│   ├── create                  # Create new test
│   ├── status <id>             # Test status and results
│   ├── stop <id>               # Stop test
│   └── analyze <id>            # Statistical analysis
├── emergency                    # Emergency controls
│   ├── halt-all                # Halt all strategies
│   ├── halt-trading            # Stop all trading (hold positions)
│   ├── liquidate-all           # Emergency liquidation
│   ├── pause-new-orders        # Pause new orders only
│   └── resume                  # Resume normal operations
├── alerts                       # Alert management
│   ├── list                    # List recent alerts
│   ├── show <id>               # Show alert details
│   ├── ack <id>                # Acknowledge alert
│   └── config                  # Alert configuration
├── audit                        # Audit trail
│   ├── log                     # Show audit log
│   ├── interventions           # Manual interventions
│   └── config-changes          # Configuration changes
└── dashboard                    # Dashboard management
    ├── start                   # Start web dashboard
    └── url                     # Get dashboard URL
```

### 2.2 Command Details

#### 2.2.1 `gt risk status`

**Purpose:** Quick overview of current risk state

**Output:**
```
🎯 Risk Status: greenplace (paper trading)

Portfolio Overview:
  Equity:         $98,450.23  (Peak: $102,340.56, -3.8% from peak)
  Cash:           $12,340.00
  Buying Power:   $87,660.00
  Positions:      12 / 20 max
  Strategies:     3 active, 0 paused, 1 halted

Risk Metrics:
  Portfolio Vol:  18.2% (target: 20.0%)  ✓
  Max Position:   $14,230 (NVDA, 14.5% of portfolio)  ✓
  Sector Conc:    Tech 42%, Finance 28%, Healthcare 18%  ⚠️

Active Alerts:
  ⚠️  WARN - Tech sector concentration at 42% (warning threshold 40%)
  ℹ️  INFO - Strategy 'ml-momentum-v2' reduced position sizing (volatility)

Phase:  Phase 1 (Exploration) - Day 18/30
Status: 🟢 Normal Operations
```

**Flags:**
- `--json`: Output as JSON for scripting
- `--watch, -w`: Watch mode (refresh every 2s)
- `--verbose, -v`: Show detailed breakdown

#### 2.2.2 `gt risk limits`

**Purpose:** View and modify risk limits

**Subcommands:**

**`gt risk limits list`**
```
Hard Limits:
  account.initial_capital              $100,000
  account.margin_multiplier            1.0
  position.max_position_pct            15%
  position.max_position_dollars        $25,000
  position.min_position_dollars        $500
  strategy.max_strategy_allocation_pct 40%
  strategy.max_strategies_active       3

Soft Limits:
  concentration.sector_warning_pct     40%
  concentration.sector_reduction_pct   50%
  volatility.position_warning          40%
  volatility.position_reduction        60%
  drawdown.strategy_halt_pct           20%
  drawdown.account_emergency_pct       25%
```

**`gt risk limits set <limit> <value>`**
```bash
$ gt risk limits set position.max_position_pct 20

⚠️  WARNING: This will change a hard limit
Current:  15%
New:      20%

This change will:
  - Allow larger individual positions
  - Increase concentration risk
  - Take effect immediately

Continue? [y/N]: y

✓ Limit updated: position.max_position_pct = 20%
✓ Config written to: risk_config.yaml
✓ Change logged to audit trail
```

**Flags:**
- `--force, -f`: Skip confirmation (dangerous, use in scripts only)
- `--dry-run`: Show what would change without applying

#### 2.2.3 `gt risk strategy`

**Purpose:** Control individual strategies

**`gt risk strategy list`**
```
ID                   Name                Status    P&L       DD    Positions  Last Trade
ml-momentum-v2       ML Momentum v2      ACTIVE    +$2,340   -5%   4          2m ago
heuristic-value-1    Value Following     ACTIVE    +$1,120   -3%   5          15m ago
ml-sentiment-exp     Sentiment (Exp)     HALTED    -$3,450   -21%  0          2d ago
contrarian-swing     Contrarian Swing    PAUSED    +$890     -2%   3          1h ago
```

**`gt risk strategy show ml-momentum-v2`**
```
Strategy: ml-momentum-v2 (ML Momentum v2)
Status:   ACTIVE
Owner:    system
Phase:    Phase 1 (Exploration)

Performance:
  Total P&L:        +$2,340.56 (+5.85%)
  Peak Equity:      $42,450.00
  Current DD:       -4.8%
  Sharpe Ratio:     1.23
  Win Rate:         58%
  Avg Win:          +2.3%
  Avg Loss:         -1.8%

Positions (4):
  NVDA  +100 shares  +$1,234  (+8.7%)
  TSLA  +50 shares   +$456    (+4.3%)
  AMD   +150 shares  -$123    (-2.1%)
  MSFT  +75 shares   +$789    (+5.6%)

Capital Allocation:
  Allocated:        $40,000 (40% of account)
  Deployed:         $38,560 (96.4% utilization)
  Available:        $1,440

Recent Orders (last 10):
  2026-01-19 09:45  BUY  NVDA  +20  $520.30  FILLED
  2026-01-19 09:12  SELL AAPL  -30  $185.20  FILLED
  ...
```

**`gt risk strategy pause ml-momentum-v2`**
```
Pausing strategy: ml-momentum-v2

This will:
  ✓ Stop generating new signals
  ✓ Reject any pending orders
  ✓ Keep existing positions open
  ✓ Allow manual resume later

Continue? [Y/n]: y

✓ Strategy paused: ml-momentum-v2
✓ 2 pending orders cancelled
✓ 4 positions remain open
```

**`gt risk strategy halt ml-momentum-v2`**
```
⚠️  EMERGENCY HALT: ml-momentum-v2

This will:
  ⚠️  Immediately halt all new orders
  ⚠️  Cancel pending orders
  ⚠️  Mark strategy for review
  ℹ️  Existing positions remain open

Reason (optional): Detected anomalous behavior in signal generation

✓ Strategy halted: ml-momentum-v2
✓ Halt reason logged
✓ Alert sent to monitoring channels
```

#### 2.2.4 `gt risk portfolio`

**Purpose:** Portfolio-level operations and metrics

**`gt risk portfolio status`**
```
Portfolio Status: greenplace

Capital:
  Initial Capital:    $100,000.00
  Current Equity:     $98,450.23
  Total P&L:          -$1,549.77 (-1.55%)
  Peak Equity:        $102,340.56
  Drawdown:           -3.80%

Positions (12):
  Longs:              12 positions, $86,110 gross
  Shorts:             0 positions, $0 gross
  Gross Exposure:     $86,110 (87.5% of equity)
  Net Exposure:       $86,110 (87.5% of equity)

Cash:
  Cash Balance:       $12,340.00
  Buying Power:       $87,660.00
  Reserve (5%):       $4,922.51
  Available:          $82,737.49

Risk Metrics:
  Portfolio Volatility:  18.2% annualized
  Value at Risk (95%):   $3,240 daily
  Beta:                  0.95 (vs SPY)
  Correlation (max):     0.72 (NVDA-AMD)
```

**`gt risk portfolio positions`**
```
Symbol  Qty    Entry      Current    P&L       P&L%    Days  Strategy           Sector
NVDA    +100   $512.30    $524.60    +$1,230   +2.4%   5     ml-momentum-v2     Tech
TSLA    +50    $245.60    $253.20    +$380     +3.1%   3     ml-momentum-v2     Auto
JPM     +80    $178.50    $182.30    +$304     +2.1%   8     heuristic-value-1  Finance
AMD     +150   $142.80    $141.95    -$127     -0.6%   5     ml-momentum-v2     Tech
...

Total:  12 positions, $86,110 deployed, +$2,340 unrealized (+2.7%)
```

**`gt risk portfolio exposure`**
```
Exposure Analysis:

By Sector:
  Technology      42.3%  ($36,400)  ⚠️  Warning: 40% threshold
  Financials      27.8%  ($23,920)  ✓
  Healthcare      18.5%  ($15,930)  ✓
  Consumer        8.2%   ($7,060)   ✓
  Energy          3.2%   ($2,800)   ✓

By Strategy:
  ml-momentum-v2      $38,560  (96.4% utilization)
  heuristic-value-1   $34,220  (85.6% utilization)
  contrarian-swing    $13,330  (33.3% utilization)

By Market Cap:
  Large Cap (>$200B)   68%
  Mid Cap ($10-200B)   24%
  Small Cap (<$10B)    8%

Concentration:
  Largest Position:    NVDA $14,230 (14.5%)  ✓
  Top 3 Positions:     $38,450 (39.1%)
  Top 5 Positions:     $56,780 (57.7%)
```

**`gt risk portfolio liquidate`**
```
⛔ EMERGENCY LIQUIDATION ⛔

This will liquidate ALL positions immediately.
This is an EMERGENCY measure with significant consequences:

Current Positions: 12
Total Value:       $86,110
Estimated Impact:  -0.5% to -2.0% (slippage & fees)

You must type "LIQUIDATE ALL" to confirm: LIQUIDATE ALL

⚠️  Generating liquidation orders...
⚠️  12 market sell orders created
⚠️  Submitting orders...

Results:
  ✓ NVDA  -100 shares  $524.58  FILLED
  ✓ TSLA  -50 shares   $253.15  FILLED
  ✓ JPM   -80 shares   $182.28  FILLED
  ...

✓ All positions liquidated
✓ Cash balance: $98,234.56
✓ Emergency liquidation logged
✓ Alert sent to monitoring channels
```

#### 2.2.5 `gt risk learning`

**Purpose:** Track learning progress and phase transitions

**`gt risk learning status`**
```
Learning Progress:

Current Phase: Phase 1 (Exploration)
  Started:     2026-01-01 (18 days ago)
  Duration:    30 days target
  Progress:    ████████████░░░░░░░░ 60%
  Next Phase:  2026-01-31 (estimated)

Phase 1 Goals:
  ✓ Complete 100+ trades (142 completed)
  ✓ Test 3+ strategies (4 deployed)
  ⏳ Achieve Sharpe > 0.5 (current: 0.42)
  ⏳ Max drawdown < 15% (current: 12.3%)

Strategy Graduation Status:
  ml-momentum-v2:       ⏳ 142 trades, Sharpe 1.23  (on track)
  heuristic-value-1:    ⏳ 89 trades, Sharpe 0.67   (on track)
  ml-sentiment-exp:     ❌ Halted (21% DD)
  contrarian-swing:     ⏳ 34 trades, Sharpe 0.31   (needs data)

Phase Transition Criteria:
  ✓ Minimum trade count (100)
  ✓ Minimum strategies (3)
  ⏳ Strategy maturity (1/4 ready to graduate)
  ⏳ Risk-adjusted returns (Sharpe 0.42 vs 0.5 target)

Risk Parameters (Phase 1):
  Position Size Mult:   0.5x (reduced)
  Exploration Bonus:    +50% (enabled)
  Soft Limit Tolerance: High
```

**`gt risk learning phase advance`**
```
⚠️  Advance to Phase 2 (Refinement)?

Current: Phase 1 (Exploration) - Day 18/30
Next:    Phase 2 (Refinement) - 60 days

Phase 2 Changes:
  Position Size:        0.5x → 0.75x  (50% increase)
  Soft Limits:          Relaxed → Standard
  New Strategy Limit:   Unlimited → 1 per month
  Exploration Mode:     Enabled → Disabled

Graduation Status:
  ⚠️  Only 0/4 strategies meet graduation criteria (Sharpe > 0.5, 50+ trades)
  ⚠️  Portfolio Sharpe: 0.42 (target: 0.5)
  ⚠️  12 days remaining in Phase 1

Recommendation: Wait for strategies to mature

Force advance? [y/N]: n

✗ Phase advance cancelled
ℹ️  Use --force flag to advance despite warnings
```

**`gt risk learning metrics`**
```
Learning Metrics:

Exploration Effectiveness:
  Strategies Tested:        4
  Strategies Graduated:     0
  Strategies Halted:        1 (25% failure rate)
  New Strategies (30d):     2

Strategy Performance Distribution:
  Top Performer:      ml-momentum-v2 (Sharpe 1.23)
  Median:             0.49
  Bottom Performer:   ml-sentiment-exp (Sharpe -0.34, halted)

Parameter Learning:
  Risk Config Changes:      7 in last 30 days
  Limit Violations:         23 (0.16 per trade)
  Soft Limit Warnings:      142 (3.2% of trades)
  Configuration Stability:  Moderate (2-3 changes/week)

Discovery Metrics:
  Profitable Discoveries:   8 new entry patterns identified
  Failed Experiments:       3 strategies rejected
  Cost of Learning:         -$3,450 (ml-sentiment-exp loss)
  Knowledge Gained:         12 validated hypotheses
```

**`gt risk learning experiments`**
```
Active Learning Experiments:

Experiment: exp-202601-momentum-vol-adjust
  Type:        Parameter tuning (volatility adjustment)
  Strategy:    ml-momentum-v2
  Started:     2026-01-12 (7 days ago)
  Status:      Active
  Hypothesis:  Dynamic position sizing by volatility improves Sharpe
  Trades:      34 (A: 18, B: 16)
  Results:     A: Sharpe 1.31, B: Sharpe 1.15  (A winning, p=0.08)

Experiment: exp-202601-sector-rotation
  Type:        Strategy variant test
  Strategy:    New (sandbox)
  Started:     2026-01-15 (4 days ago)
  Status:      Active
  Hypothesis:  Sector rotation timing improves returns
  Trades:      8 (insufficient data)
  Results:     Sharpe 0.45 (early)
```

#### 2.2.6 `gt risk ab-test`

**Purpose:** Manage A/B tests for strategy improvements

**`gt risk ab-test list`**
```
Active A/B Tests:

ID             Name                          Strategy        Status    Duration  Winner
test-20240119  Vol Adjustment Multiplier     ml-momentum-v2  ACTIVE    7d        A (p=0.08)
test-20240115  Entry Timing Delay            heuristic-v1    ACTIVE    4d        TBD
test-20240110  Stop Loss Tightness           contrarian      COMPLETE  14d       B (p=0.03)*

Completed Tests (last 30 days): 3
  Winner A: 1
  Winner B: 1
  No Difference: 1
```

**`gt risk ab-test create`**
```bash
$ gt risk ab-test create \
    --strategy ml-momentum-v2 \
    --name "Exit timing optimization" \
    --variant-a "Current (5% profit target)" \
    --variant-b "New (Dynamic based on volatility)" \
    --duration 14d \
    --traffic-split 50/50

Creating A/B Test:

Name:           Exit timing optimization
Strategy:       ml-momentum-v2
Variants:       A (Current), B (New)
Traffic Split:  50% A, 50% B
Duration:       14 days (until 2026-02-02)
Success Metric: Sharpe Ratio

Variant A (Control):
  Exit Target:  5% profit target (fixed)
  Stop Loss:    3% stop loss (fixed)

Variant B (Treatment):
  Exit Target:  Dynamic (2x realized volatility)
  Stop Loss:    Dynamic (1x realized volatility)

✓ Test created: test-20260119-exit-timing
✓ Traffic routing configured
✓ Metrics collection enabled
✓ Test will run until 2026-02-02
```

**`gt risk ab-test status test-20240119`**
```
A/B Test: test-20240119 (Vol Adjustment Multiplier)

Configuration:
  Strategy:       ml-momentum-v2
  Started:        2026-01-12
  Duration:       14 days (7d remaining)
  Traffic:        50% A / 50% B
  Success Metric: Sharpe Ratio

Variant A (Control): Standard position sizing
Variant B (Treatment): Volatility-adjusted sizing

Results (7 days, 34 trades):

Metric               Variant A    Variant B    Difference   P-Value
Sharpe Ratio         1.15         1.31         +0.16        0.082
Total Return         +4.2%        +5.8%        +1.6pp       0.091
Win Rate             54%          61%          +7pp         0.156
Avg Win              +2.1%        +2.4%        +0.3pp       0.412
Avg Loss             -1.8%        -1.6%        +0.2pp       0.223
Max Drawdown         -6.2%        -4.8%        +1.4pp       0.134
Volatility           12.3%        11.1%        -1.2pp       0.078

Statistical Analysis:
  Sample Size:    34 trades (18 A, 16 B)
  Power:          56% (target: 80%, need ~60 trades)
  Early Signal:   ✓ Variant B trending better
  Significance:   ⏳ Not yet significant (p=0.08 > 0.05)
  Recommendation: Continue test (7 days remaining)

If B wins (projected):
  Expected Improvement: +14% Sharpe
  Rollout Risk:         Low (stable improvement across metrics)
  Deployment:           Recommend gradual rollout (10%→50%→100%)
```

**`gt risk ab-test analyze test-20240110`**
```
A/B Test Analysis: test-20240110 (Stop Loss Tightness)

Status:          COMPLETED
Winner:          Variant B (p=0.03) *SIGNIFICANT*
Duration:        14 days
Trades:          67 total (34 A, 33 B)

Final Results:

Metric               Variant A    Variant B    Difference   P-Value
Sharpe Ratio         0.82         1.14         +0.32        0.028  *
Total Return         +3.1%        +4.8%        +1.7pp       0.042  *
Win Rate             48%          58%          +10pp        0.089
Avg Win              +3.2%        +2.8%        -0.4pp       0.287
Avg Loss             -2.4%        -1.9%        +0.5pp       0.034  *
Max Drawdown         -8.9%        -6.2%        +2.7pp       0.019  *

Analysis:
  ✓ Statistically significant improvement in Sharpe (p=0.028)
  ✓ Lower losses with tighter stops
  ✓ Higher win rate compensates for smaller wins
  ✓ Significantly reduced max drawdown

Recommendation: DEPLOY VARIANT B

Deployment Plan:
  1. Roll out to 25% of traffic (2 days)
  2. Monitor for regressions
  3. Roll out to 100% if stable
  4. Update strategy baseline

✓ Test archived
✓ Results logged
✓ Variant B configuration saved to: configs/contrarian-tight-stops.yaml
```

#### 2.2.7 `gt risk emergency`

**Purpose:** Emergency controls for critical situations

**`gt risk emergency halt-all`**
```
⛔ EMERGENCY HALT ALL STRATEGIES ⛔

This will IMMEDIATELY halt all strategies.

Current State:
  Active Strategies:     3
  Pending Orders:        7
  Open Positions:        12
  Portfolio Value:       $86,110

Actions:
  ⛔ Halt all strategies
  ⛔ Cancel all pending orders
  ℹ️  Existing positions remain open
  ℹ️  Manual resume required

You must type "HALT ALL" to confirm: HALT ALL

⚠️  Halting all strategies...

✓ ml-momentum-v2: HALTED (2 orders cancelled)
✓ heuristic-value-1: HALTED (3 orders cancelled)
✓ contrarian-swing: HALTED (2 orders cancelled)

✓ All strategies halted
✓ 7 orders cancelled
✓ 12 positions remain open
✓ Emergency halt logged
✓ Alert sent to all channels

To resume: gt risk emergency resume
```

**`gt risk emergency halt-trading`**
```
⛔ EMERGENCY TRADING HALT ⛔

This will stop ALL trading activity.

Actions:
  ⛔ Stop all new orders
  ⛔ Cancel all pending orders
  ⛔ Prevent position changes
  ℹ️  Strategies continue running (signals only)
  ℹ️  Existing positions held

This is a "freeze" - useful for:
  - Market circuit breakers
  - Exchange outages
  - Data feed issues
  - Pending investigation

Reason (optional): Market volatility - S&P down 5% in 30 minutes

✓ Trading halted
✓ All pending orders cancelled
✓ Position changes blocked
✓ Emergency logged with reason
```

**`gt risk emergency liquidate-all`**
```
⛔⛔⛔ FULL EMERGENCY LIQUIDATION ⛔⛔⛔

This will:
  ⛔ Halt all strategies
  ⛔ Cancel all orders
  ⛔ LIQUIDATE ALL POSITIONS
  ⛔ Return to 100% cash

Current Portfolio:
  Positions:      12
  Value:          $86,110
  Unrealized P&L: +$2,340

⚠️  This is the most drastic emergency measure
⚠️  Only use in catastrophic scenarios:
    - System malfunction
    - Runaway algorithm
    - External security breach
    - Regulatory requirement

You must type "LIQUIDATE EVERYTHING" to confirm: LIQUIDATE EVERYTHING

⚠️  Halting all strategies...
⚠️  Cancelling all pending orders...
⚠️  Generating liquidation orders...
⚠️  Submitting market orders...

Liquidation Results:
  ✓ NVDA  -100 shares  $524.58  FILLED
  ✓ TSLA  -50 shares   $253.15  FILLED
  ...

✓ All positions liquidated
✓ Cash balance: $98,234.56
✓ All strategies halted
✓ Emergency liquidation logged
✓ CRITICAL alert sent to all channels

System Status: EMERGENCY MODE
To resume: gt risk emergency resume
```

**`gt risk emergency resume`**
```
Resume from Emergency Mode?

Current State:
  System Mode:        EMERGENCY
  Strategies Halted:  3
  Trading:            SUSPENDED
  Positions:          12 open
  Last Emergency:     2026-01-19 10:23:45 (5m ago)
  Reason:             Manual halt (market volatility check)

Resume Actions:
  ✓ Resume normal operations
  ✓ Enable trading
  ✓ Strategies remain PAUSED (manual resume each)

This will NOT automatically resume strategies.
You must manually resume each strategy after verification.

Continue? [Y/n]: y

✓ Emergency mode lifted
✓ Trading enabled
✓ Normal operations resumed
ℹ️  Strategies remain paused - resume individually with:
    gt risk strategy resume <strategy-id>
```

#### 2.2.8 `gt risk alerts`

**Purpose:** Manage alerts and notifications

**`gt risk alerts list`**
```
Recent Alerts (last 24h):

Time      Level     Source              Message
10:23:45  CRITICAL  emergency          Full emergency halt triggered
10:15:32  ERROR     strategy           Strategy ml-sentiment-exp halted (20% DD)
09:45:12  WARN      concentration      Tech sector at 42% (threshold: 40%)
09:30:45  WARN      volatility         Portfolio vol at 23.4% (target: 20%)
08:15:23  INFO      soft-limit         Position size reduced (NVDA, high vol)
07:45:10  INFO      phase              Phase 1 day 18/30 checkpoint

Total: 47 alerts (1 critical, 2 error, 12 warn, 32 info)
Acknowledged: 45
Pending: 2
```

**`gt risk alerts config`**
```
Alert Configuration:

Channels:
  Dashboard:  ✓ Enabled (always on)
  Logs:       ✓ Enabled (JSON format)
  Email:      ✓ Enabled (operator@example.com)
  Slack:      ✗ Disabled

Alert Levels by Channel:
  Dashboard:  All levels (INFO, WARN, ERROR, CRITICAL)
  Logs:       All levels
  Email:      ERROR and CRITICAL only
  Slack:      N/A

Rate Limiting:
  Max per hour:     100 (any level)
  Max CRITICAL/h:   10
  Cooldown:         5m between duplicate alerts

Quiet Hours:
  Enabled:    ✓ Yes
  Window:     22:00 - 07:00 PST
  Behavior:   Buffer non-critical alerts, send in morning digest
  Override:   CRITICAL alerts always sent immediately
```

#### 2.2.9 `gt risk audit`

**Purpose:** Audit trail and compliance

**`gt risk audit log`**
```
Audit Log (last 50 events):

Time                Event Type         User      Details
2026-01-19 10:23:45 emergency.halt_all system    Reason: Manual intervention
2026-01-19 10:15:32 strategy.halt      system    Strategy: ml-sentiment-exp, Reason: 20% DD threshold
2026-01-19 09:45:12 config.change      operator  Limit: position.max_position_pct, 15% → 20%
2026-01-19 08:30:00 phase.checkpoint   system    Phase 1 day 18/30
2026-01-18 16:45:23 strategy.pause     operator  Strategy: contrarian-swing, Reason: Review signals
2026-01-18 14:20:10 ab_test.create     operator  Test: test-20260118-exit-timing
...

Filter by:
  Type:  [all|emergency|config|strategy|position]
  User:  [all|system|operator|<username>]
  Level: [all|info|warn|error|critical]
```

**`gt risk audit interventions`**
```
Manual Interventions (last 30 days):

Date       User      Action                  Details                           Outcome
2026-01-19 operator  emergency.halt_all      Market volatility check           Resumed after 10m
2026-01-18 operator  strategy.pause          contrarian-swing review           Still paused
2026-01-17 operator  config.change           Increased position limit          Applied successfully
2026-01-15 operator  strategy.halt           ml-sentiment-exp poor performance Emergency halt
2026-01-12 operator  limits.reset            Reset to defaults after testing   Applied successfully

Total Interventions: 12
  Emergency Halts:   2
  Strategy Controls: 6
  Config Changes:    4

Average Response Time: 8 minutes (time from alert to intervention)
```

#### 2.2.10 `gt risk dashboard`

**Purpose:** Start web dashboard

**`gt risk dashboard start`**
```
🎯 Risk Management Dashboard starting...

Server:   http://localhost:8080
API:      http://localhost:8080/api
Updates:  WebSocket on /ws (real-time)

Dashboard Features:
  - Portfolio overview and P&L chart
  - Real-time position monitoring
  - Risk metrics and gauges
  - Strategy performance comparison
  - Alert feed
  - Learning progress tracker

✓ Dashboard running at http://localhost:8080
  Press Ctrl+C to stop
```

**Flags:**
- `--port <n>`: Use custom port (default: 8080)
- `--open`: Automatically open browser
- `--read-only`: Disable control actions (monitoring only)

## 3. Web Dashboard

### 3.1 Dashboard Architecture

**Technology Stack:**
- **Backend**: Go HTTP server (similar to convoy dashboard)
- **Frontend**: HTML + htmx for real-time updates
- **Styling**: Tailwind CSS for responsive design
- **Charts**: Chart.js for visualizations
- **Updates**: Server-Sent Events (SSE) for real-time data push

**Design Principles:**
- Mobile-responsive
- Works without JavaScript (graceful degradation)
- Real-time updates via htmx
- No complex SPA framework (keep it simple)

### 3.2 Dashboard Views

#### 3.2.1 Overview Dashboard (`/`)

**Layout:**
```
┌─────────────────────────────────────────────────────────────┐
│  Risk Management Dashboard                    🟢 NORMAL     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Portfolio Summary                  Risk Metrics             │
│  ┌──────────────────────┐          ┌──────────────────────┐ │
│  │ Equity: $98,450      │          │ Vol:  18.2%  ✓       │ │
│  │ P&L:    -$1,550      │          │ VaR:  $3,240  ✓      │ │
│  │ DD:     -3.8%        │          │ Beta: 0.95    ✓      │ │
│  │ Cash:   $12,340      │          │ Conc: 42%     ⚠️      │ │
│  └──────────────────────┘          └──────────────────────┘ │
│                                                              │
│  P&L Chart (30 days)                                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         📈 [Interactive P&L line chart]              │  │
│  │                                                       │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  Active Strategies                                          │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ ml-momentum-v2     🟢  +$2,340  Sharpe: 1.23        │  │
│  │ heuristic-value-1  🟢  +$1,120  Sharpe: 0.67        │  │
│  │ contrarian-swing   ⏸️   +$890   Sharpe: 0.31        │  │
│  │ ml-sentiment-exp   🔴  -$3,450  Sharpe: -0.34       │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  Recent Alerts                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ ⚠️  10:15  Tech sector concentration at 42%          │  │
│  │ ℹ️  09:45  Position size reduced (volatility)        │  │
│  │ ℹ️  08:30  Phase 1 checkpoint (day 18/30)            │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Features:**
- Real-time updates (every 5s via htmx)
- Color-coded status indicators
- Interactive charts with zoom/pan
- Click-through to detailed views

#### 3.2.2 Positions View (`/positions`)

**Features:**
- Real-time position updates
- P&L sparklines for each position
- Sortable columns (P&L, size, days held)
- Filter by strategy, sector, status
- Quick actions: close position, adjust stop-loss

#### 3.2.3 Risk Metrics View (`/risk`)

**Features:**
- Risk metric gauges (volatility, VaR, etc.)
- Limit utilization bars
- Concentration heatmaps (sector, strategy)
- Correlation matrix
- Historical risk metric charts

#### 3.2.4 Strategy Performance View (`/strategies`)

**Features:**
- Strategy comparison table
- Performance metrics (Sharpe, win rate, etc.)
- Equity curves for each strategy
- Trade history
- Strategy controls (pause/resume/halt)

#### 3.2.5 Learning Progress View (`/learning`)

**Features:**
- Phase progress bar
- Strategy graduation status
- Learning metrics charts
- Experiment list
- Phase history timeline

#### 3.2.6 A/B Test Dashboard (`/ab-tests`)

**Features:**
- Active test list with status
- Test result visualizations
- Statistical significance indicators
- Metric comparison tables
- Test history

#### 3.2.7 Alerts View (`/alerts`)

**Features:**
- Alert feed (real-time)
- Filter by level, source
- Alert details modal
- Acknowledge alerts
- Alert statistics

### 3.3 Real-Time Updates

**Server-Sent Events (SSE) Endpoint:** `/api/stream`

**Event Types:**
```json
{
  "type": "portfolio_update",
  "timestamp": "2026-01-19T10:15:32Z",
  "data": {
    "equity": 98450.23,
    "pnl": -1549.77,
    "drawdown": -0.038
  }
}

{
  "type": "alert",
  "timestamp": "2026-01-19T10:15:32Z",
  "data": {
    "level": "WARN",
    "source": "concentration",
    "message": "Tech sector at 42%"
  }
}

{
  "type": "position_update",
  "timestamp": "2026-01-19T10:15:32Z",
  "data": {
    "symbol": "NVDA",
    "quantity": 100,
    "pnl": 1230.45
  }
}
```

### 3.4 Emergency Controls in Dashboard

**Emergency Control Panel** (top-right corner):
```
┌──────────────────────────────┐
│ Emergency Controls            │
├──────────────────────────────┤
│ [Halt All Strategies]         │
│ [Halt Trading]                │
│ [Liquidate All] ⛔           │
│ [Resume Operations]           │
└──────────────────────────────┘
```

**Safety Features:**
- Require confirmation modal
- Type-to-confirm for destructive actions
- Disabled in read-only mode
- All actions logged to audit trail

## 4. Alert System

### 4.1 Alert Levels

| Level    | Severity | Channel           | Response Required |
|----------|----------|-------------------|-------------------|
| INFO     | Low      | Dashboard, Logs   | No                |
| WARN     | Medium   | Dashboard, Logs, Email (digest) | Review within 24h |
| ERROR    | High     | Dashboard, Logs, Email (immediate) | Review within 1h |
| CRITICAL | Urgent   | All channels      | Immediate action  |

### 4.2 Alert Configuration

**Config File:** `risk_alerts.yaml`

```yaml
alerts:
  channels:
    dashboard:
      enabled: true
      levels: [INFO, WARN, ERROR, CRITICAL]

    logs:
      enabled: true
      levels: [INFO, WARN, ERROR, CRITICAL]
      format: json
      path: /var/log/risk/alerts.log
      rotation: daily

    email:
      enabled: true
      levels: [ERROR, CRITICAL]
      to: operator@example.com
      from: risk-system@example.com
      smtp_server: smtp.example.com

    slack:
      enabled: false
      levels: [ERROR, CRITICAL]
      webhook_url: https://hooks.slack.com/...

  rate_limiting:
    max_per_hour: 100
    max_critical_per_hour: 10
    duplicate_cooldown_minutes: 5

  quiet_hours:
    enabled: true
    start_time: "22:00"
    end_time: "07:00"
    timezone: "America/Los_Angeles"
    buffer_non_critical: true
    always_send_critical: true
```

### 4.3 Alert Rules

**Hard Limit Violations:**
```yaml
alert:
  trigger: hard_limit_violation
  level: ERROR
  message: "Order rejected: {limit_name} exceeded"
  channels: [dashboard, logs, email]
```

**Soft Limit Warnings:**
```yaml
alert:
  trigger: soft_limit_warning
  level: WARN
  message: "{limit_name} warning threshold reached"
  channels: [dashboard, logs]
```

**Strategy Halt:**
```yaml
alert:
  trigger: strategy_halt
  level: ERROR
  message: "Strategy {strategy_id} halted: {reason}"
  channels: [dashboard, logs, email]
```

**Emergency Halt:**
```yaml
alert:
  trigger: emergency_halt
  level: CRITICAL
  message: "Emergency halt triggered: {reason}"
  channels: [dashboard, logs, email, slack]
```

## 5. Configuration Management

### 5.1 Configuration Files

**Directory Structure:**
```
config/
├── risk_config.yaml          # Main risk configuration
├── risk_alerts.yaml          # Alert configuration
├── strategies/               # Strategy configurations
│   ├── ml-momentum-v2.yaml
│   ├── heuristic-value-1.yaml
│   └── ...
├── ab_tests/                 # A/B test configurations
│   ├── test-20260119.yaml
│   └── ...
└── history/                  # Configuration history (version control)
    ├── risk_config_20260119_102345.yaml
    └── ...
```

### 5.2 Hot Reload

**Implementation:**
- File watcher monitors config files
- Changes trigger validation
- Valid changes applied atomically
- Invalid changes rejected with error message
- Reload logged to audit trail

**Safety:**
- Config validation before reload
- Rollback on validation failure
- Alert on config reload
- Gradual rollout option for sensitive changes

### 5.3 Configuration Versioning

**Git Integration:**
- All config files tracked in git
- Automatic commits on change (with audit info)
- Config history queryable via `gt risk audit config-changes`
- Easy rollback to previous versions

**Example Commit:**
```
risk: increase position limit to 20%

Changed by: operator
Reason: Strategy maturity reached
Previous: 15%
New: 20%

Audit-Id: aud-20260119-102345
```

## 6. Data Storage and APIs

### 6.1 Data Requirements

**Time-Series Data:**
- Portfolio equity (real-time, 1s granularity)
- Position values (real-time, 5s granularity)
- Risk metrics (calculated, 1m granularity)
- Market data (real-time, tick-by-tick)

**Relational Data:**
- Positions (current and historical)
- Orders (all orders with status)
- Strategies (config and state)
- A/B tests (config and results)
- Alerts (all alerts with ack status)
- Audit log (all events)

**Storage Solutions:**
- **TimescaleDB**: Time-series data (equity, prices, metrics)
- **PostgreSQL**: Relational data (positions, orders, strategies)
- **Redis**: Real-time data (current positions, live metrics)
- **JSON Files**: Configuration (git-backed)

### 6.2 API Endpoints

**REST API** (read operations):
```
GET  /api/portfolio/status         # Portfolio summary
GET  /api/portfolio/positions      # All positions
GET  /api/portfolio/exposure       # Exposure analysis
GET  /api/portfolio/risk           # Risk metrics
GET  /api/strategies                # List strategies
GET  /api/strategies/{id}          # Strategy details
GET  /api/strategies/{id}/metrics  # Strategy metrics
GET  /api/learning/status          # Learning progress
GET  /api/learning/experiments     # Active experiments
GET  /api/ab-tests                 # List A/B tests
GET  /api/ab-tests/{id}            # Test details
GET  /api/alerts                   # Recent alerts
GET  /api/audit/log                # Audit log
GET  /api/config                   # Current config
```

**Control API** (write operations, require auth):
```
POST /api/strategies/{id}/pause    # Pause strategy
POST /api/strategies/{id}/resume   # Resume strategy
POST /api/strategies/{id}/halt     # Halt strategy
POST /api/emergency/halt-all       # Emergency halt
POST /api/emergency/halt-trading   # Halt trading
POST /api/emergency/liquidate      # Emergency liquidation
POST /api/emergency/resume         # Resume operations
POST /api/config/update            # Update config
POST /api/config/reload            # Hot reload config
POST /api/alerts/{id}/ack          # Acknowledge alert
```

**WebSocket API** (real-time updates):
```
WS   /api/stream                   # SSE stream for real-time updates
```

### 6.3 API Authentication

**Methods:**
- **CLI**: Automatic (local socket)
- **Dashboard**: Session-based (HTTP-only cookies)
- **External**: API key (for integrations)

**Permissions:**
- **Read-Only**: View all data, no control actions
- **Operator**: Full control (pause/resume/halt)
- **Emergency**: Emergency controls only (halt/liquidate)

## 7. Performance Requirements

### 7.1 Latency Targets

| Operation | P50 | P99 | Notes |
|-----------|-----|-----|-------|
| CLI commands (read) | <100ms | <500ms | Simple status queries |
| CLI commands (write) | <200ms | <1s | Config updates |
| Dashboard page load | <500ms | <2s | Initial load |
| Dashboard updates | <100ms | <500ms | Real-time data |
| API endpoints (read) | <50ms | <200ms | Simple queries |
| API endpoints (write) | <100ms | <500ms | Control actions |
| Emergency halt | <50ms | <100ms | Critical path |

### 7.2 Update Frequencies

| Data Type | Update Frequency | Latency Tolerance |
|-----------|------------------|-------------------|
| Portfolio equity | 1s | 5s max |
| Position values | 5s | 30s max |
| Risk metrics | 1m | 5m max |
| Dashboard UI | 5s | 10s max |
| Alerts | Real-time | 1s max |

### 7.3 Scalability

**Current Scale (Phase 1):**
- 3-5 strategies
- 20 positions max
- 100 orders/day
- 1 operator

**Future Scale (Phase 3):**
- 10+ strategies
- 50+ positions
- 1000+ orders/day
- Multiple operators

**Design for Scale:**
- Stateless API servers (horizontal scaling)
- Time-series database partitioning
- Redis caching for hot data
- Async processing for heavy computations

## 8. Security Considerations

### 8.1 Access Control

**Local CLI:**
- No authentication required (runs as local user)
- File system permissions protect config files
- Audit log tracks all operations

**Web Dashboard:**
- Session-based authentication
- HTTPS required (self-signed cert OK for local)
- CSRF protection
- Rate limiting on control endpoints

**External API:**
- API key authentication
- IP whitelist (optional)
- TLS required
- Rate limiting

### 8.2 Emergency Controls Security

**Destructive Operations:**
- Require type-to-confirm
- Audit log with reason
- Alert to all channels
- Cannot be scripted (--force flag disabled)

**Emergency Halt:**
- Can be triggered by anyone (safety over security)
- Requires reason (logged)
- Broadcast alert

**Liquidation:**
- Most restricted operation
- Requires special permission level
- Type-to-confirm with specific phrase
- Multiple confirmations for production

### 8.3 Configuration Security

**File Permissions:**
- Config files: 0600 (owner read/write only)
- Audit log: 0600 (owner read/write only)
- API keys: 0400 (owner read only)

**Git Security:**
- Config repo: private
- Commit signing: required
- Branch protection: enabled on main

## 9. Testing Strategy

### 9.1 Unit Tests

**Coverage:**
- CLI command parsing and validation
- Limit calculations
- Risk metric computations
- Alert rule evaluation
- Configuration validation
- API endpoint logic

**Tools:**
- Go standard testing
- Testify for assertions
- Table-driven tests

### 9.2 Integration Tests

**Coverage:**
- CLI → API → Database flow
- Dashboard → API → Control actions
- Alert triggering → Notification delivery
- Config update → Hot reload
- Emergency halt → System state

**Tools:**
- Docker Compose for dependencies
- Test database seeding
- HTTP client tests

### 9.3 UI Tests

**Coverage:**
- Dashboard rendering
- Real-time updates
- User interactions
- Mobile responsiveness

**Tools:**
- Playwright for E2E tests
- Visual regression testing

### 9.4 Load Tests

**Scenarios:**
- 100+ concurrent dashboard users
- 1000 orders/minute processing
- Real-time update broadcast to 50 clients
- Database query performance under load

**Tools:**
- k6 for load testing
- PostgreSQL EXPLAIN ANALYZE

### 9.5 Disaster Recovery Tests

**Scenarios:**
- Database failure during emergency halt
- Network partition during liquidation
- Config corruption during hot reload
- Dashboard crash during alert storm

**Validation:**
- System fails safe (halts on error)
- Audit log remains intact
- Recovery procedures work
- No data loss

## 10. Deployment and Operations

### 10.1 Deployment Architecture

**Components:**
```
┌─────────────────────────────────────────────────────────┐
│  Operator Machine (Local)                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │   CLI (gt)   │  │   Browser    │  │ Config Files │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘ │
│         │                  │                  │          │
├─────────┼──────────────────┼──────────────────┼──────────┤
│         │                  │                  │          │
│  ┌──────▼──────────────────▼──────────────────▼───────┐ │
│  │          Risk Management Service                    │ │
│  │  - HTTP API - WebSocket - Config Manager           │ │
│  └──────┬──────────────────────────────────────────────┘ │
│         │                                                 │
│  ┌──────▼──────────────────────────────────────────┐    │
│  │  Data Layer                                      │    │
│  │  - PostgreSQL - TimescaleDB - Redis             │    │
│  └──────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

**Deployment Options:**
1. **Single Machine** (Phase 1): All components on operator's machine
2. **Separate Database** (Phase 2): Database on separate server
3. **Distributed** (Phase 3): Multiple API servers, load balancer

### 10.2 Installation

**Prerequisites:**
- Go 1.21+
- PostgreSQL 14+
- TimescaleDB extension
- Redis 6+

**Installation:**
```bash
# Install gt binary
gt install

# Initialize risk management
gt risk init

# Start services
gt risk service start

# Verify installation
gt risk doctor
```

### 10.3 Monitoring

**Health Checks:**
```bash
# System health
gt risk health

# Service status
gt risk service status

# Database connectivity
gt risk db ping

# API health
curl http://localhost:8080/health
```

**Metrics to Monitor:**
- API response times
- Database query times
- Dashboard active users
- Alert delivery latency
- WebSocket connection count
- Config reload frequency

**Alerting:**
- Service down
- Database connection loss
- Disk space low
- High API error rate
- Alert delivery failures

### 10.4 Backup and Recovery

**Backup Strategy:**
- **Database**: Daily full backup, continuous WAL archiving
- **Config**: Git-backed (automatic versioning)
- **Audit Log**: Append-only, backed up daily
- **System State**: Snapshot before major changes

**Recovery Procedures:**
```bash
# Restore database from backup
gt risk db restore --date 2026-01-18

# Rollback config to previous version
gt risk config rollback --commit abc123

# Verify system state
gt risk doctor --verify-data
```

## 11. Documentation and Training

### 11.1 Documentation Structure

**User Documentation:**
- Getting Started Guide
- CLI Reference (man pages)
- Dashboard User Guide
- Emergency Procedures Playbook
- Troubleshooting Guide

**Developer Documentation:**
- Architecture Overview
- API Reference (OpenAPI spec)
- Database Schema
- Configuration Reference
- Extension Guide (plugins)

**Operational Documentation:**
- Installation Guide
- Deployment Guide
- Monitoring and Alerting Setup
- Backup and Recovery Procedures
- Incident Response Playbook

### 11.2 Training Materials

**Operator Training:**
1. System Overview (30 minutes)
2. Daily Operations (1 hour)
3. Emergency Response (1 hour)
4. Hands-on Practice (2 hours)

**Emergency Drills:**
- Quarterly emergency response drill
- Practice emergency halt
- Practice full liquidation (test env)
- Config rollback scenarios

## 12. Future Enhancements

### 12.1 Phase 2 (Q2 2026)

1. **Advanced Analytics:**
   - Scenario analysis (what-if simulations)
   - Attribution analysis (factor decomposition)
   - Backtest integration (test config changes)

2. **Machine Learning:**
   - Anomaly detection (unusual trading patterns)
   - Predictive alerts (risk forecast)
   - Auto-tuning (optimal limit discovery)

3. **Collaboration:**
   - Multi-user support
   - Role-based access control
   - Shared annotations and notes

### 12.2 Phase 3 (Q3-Q4 2026)

1. **Real Money Integration:**
   - Broker API integration
   - Real-time order routing
   - Regulatory compliance checks

2. **Advanced Risk:**
   - Options Greeks
   - Portfolio hedging recommendations
   - Correlation stress testing

3. **Mobile:**
   - Mobile dashboard (iOS/Android)
   - Push notifications
   - Emergency controls on mobile

## 13. Success Metrics

### 13.1 Tooling Effectiveness

**Primary Metrics:**
1. **Operator Efficiency**: Time to complete common tasks
   - View portfolio status: <10s
   - Pause strategy: <30s
   - Emergency halt: <60s

2. **System Reliability**: Uptime and error rates
   - Dashboard uptime: >99.9%
   - API error rate: <0.1%
   - Data freshness: <5s lag

3. **Audit Compliance**: Tracking and logging
   - 100% of control actions logged
   - Config changes tracked in git
   - Alert delivery: >99.5% success

**Secondary Metrics:**
4. **User Satisfaction**: Operator feedback
   - Dashboard usability: >4/5
   - CLI intuitiveness: >4/5
   - Documentation quality: >4/5

5. **Operational Efficiency**: Time saved vs manual
   - 80% reduction in manual monitoring time
   - 90% reduction in data gathering time

### 13.2 Risk Management Impact

**Metrics from RISK_FRAMEWORK_SPEC.md:**
1. Hard Limit Violation Rate: <0.01%
2. Soft Limit Warning Rate: 2-5%
3. Maximum Drawdown: <25%
4. Strategy Halt Rate: <10%
5. Risk-Adjusted Returns: Sharpe improvement

**Tooling Contribution:**
- Faster incident response
- More accurate risk measurement
- Better strategy management
- Improved learning velocity

## 14. Appendices

### Appendix A: CLI Command Quick Reference

```bash
# Status and monitoring
gt risk status                     # Quick status overview
gt risk status --watch             # Watch mode
gt risk portfolio status           # Detailed portfolio
gt risk portfolio positions        # All positions

# Strategy control
gt risk strategy list              # List strategies
gt risk strategy pause <id>        # Pause strategy
gt risk strategy resume <id>       # Resume strategy
gt risk strategy halt <id>         # Emergency halt

# Configuration
gt risk config show                # Show config
gt risk config edit                # Edit config
gt risk limits list                # List all limits
gt risk limits set <limit> <value> # Update limit

# Emergency
gt risk emergency halt-all         # Halt everything
gt risk emergency liquidate-all    # Full liquidation
gt risk emergency resume           # Resume operations

# Learning and testing
gt risk learning status            # Learning progress
gt risk ab-test list               # List A/B tests
gt risk ab-test status <id>        # Test results

# Monitoring
gt risk alerts list                # Recent alerts
gt risk audit log                  # Audit trail
gt risk dashboard start            # Start dashboard
```

### Appendix B: Dashboard URL Structure

```
/                          # Overview dashboard
/portfolio                 # Portfolio details
/portfolio/positions       # Position list
/portfolio/exposure        # Exposure analysis
/risk                      # Risk metrics
/strategies                # Strategy list
/strategies/:id            # Strategy details
/learning                  # Learning progress
/learning/experiments      # Experiments
/ab-tests                  # A/B test list
/ab-tests/:id              # Test details
/alerts                    # Alert feed
/audit                     # Audit log
/config                    # Configuration
/emergency                 # Emergency controls
```

### Appendix C: Configuration File Templates

See `/templates` directory for:
- `risk_config.template.yaml`
- `risk_alerts.template.yaml`
- `strategy.template.yaml`
- `ab_test.template.yaml`

### Appendix D: API OpenAPI Specification

Full OpenAPI 3.0 specification available at:
- File: `docs/api/openapi.yaml`
- Endpoint: `http://localhost:8080/api/docs`

### Appendix E: Database Schema

Full database schema documentation available at:
- File: `docs/database/schema.md`
- ERD: `docs/database/erd.png`

---

**End of Specification**

**Document Metadata:**
- **Author**: Polecat Agent (bartertown/polecats/dag)
- **Review Status**: Planning Draft
- **Next Steps**: Review and approval, then implementation planning
- **Related Documents**: RISK_FRAMEWORK_SPEC.md, AGENTS.md, README.md
