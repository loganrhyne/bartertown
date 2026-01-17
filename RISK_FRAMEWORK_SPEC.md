# Risk Management Framework Specification

**Version:** 1.0
**Date:** 2026-01-17
**Status:** Draft Planning Specification
**System:** Congressional Trading Follower (Paper Trading)

## Executive Summary

This document specifies a comprehensive risk management framework for a multi-strategy congressional trading follower system operating in paper trading mode. The framework balances learning opportunities with capital protection through a layered approach of hard limits, soft limits, and dynamic adjustments.

## 1. System Context

### 1.1 Trading Environment
- **Mode:** Paper trading (simulated capital)
- **Strategies:** Multi-strategy system supporting ML-based and heuristics-based approaches
- **Data Source:** Congressional trading disclosures (public SEC filings)
- **Primary Goal:** Learning and strategy validation while maintaining realistic risk constraints

### 1.2 Risk Management Goals
1. **Capital Preservation:** Prevent catastrophic losses in paper trading scenarios
2. **Learning Enablement:** Allow strategies to explore and learn from diverse market conditions
3. **Realistic Simulation:** Enforce constraints that mirror real-world trading limitations
4. **Multi-Strategy Coordination:** Aggregate and manage risk across multiple concurrent strategies
5. **Dynamic Adaptation:** Adjust risk parameters based on strategy performance and market conditions

## 2. Hard Limits System

Hard limits are **strictly enforced** boundaries that cannot be exceeded. Violations result in order rejection.

### 2.1 Account-Level Hard Limits

#### 2.1.1 Buying Power Limit
```
Maximum Position Value = Initial Capital × Margin Multiplier
Default: $100,000 initial capital × 1.0 (no margin) = $100,000
```

**Enforcement:**
- Check before every order execution
- Total portfolio value (positions + pending orders) must not exceed limit
- Cash balance must cover order value + transaction costs

**Configuration Parameters:**
- `initial_capital`: Starting capital (default: $100,000)
- `margin_multiplier`: Leverage allowed (default: 1.0 for paper trading)
- `reserve_ratio`: Minimum cash reserve (default: 0.05 = 5%)

#### 2.1.2 Position Size Limits

**Single Position Maximum:**
```
Max Position Size = min(
    Account Value × max_position_pct,
    max_position_dollars
)
Default: min($100,000 × 0.15, $25,000) = $15,000
```

**Per-Order Maximum:**
```
Max Order Size = Position Limit - Current Position Value
```

**Configuration Parameters:**
- `max_position_pct`: Maximum position as percentage of account (default: 0.15 = 15%)
- `max_position_dollars`: Absolute maximum position size (default: $25,000)
- `min_position_dollars`: Minimum viable position size (default: $500)

### 2.2 Strategy-Level Hard Limits

#### 2.2.1 Strategy Allocation Limit
```
Max Strategy Capital = Account Value × strategy_allocation_pct
Default per strategy: $100,000 × 0.40 = $40,000
```

**Purpose:** Prevent any single strategy from dominating portfolio

**Configuration Parameters:**
- `strategy_allocation_pct`: Maximum capital per strategy (default: 0.40 = 40%)
- `max_strategies_active`: Maximum concurrent strategies (default: 3)

#### 2.2.2 Strategy Position Count Limit
```
Max Positions per Strategy: 10
Max Total Positions: 20
```

**Purpose:** Prevent over-diversification and maintain manageable portfolio

### 2.3 Security-Level Hard Limits

#### 2.3.1 Minimum Price Threshold
```
Min Stock Price: $5.00
```

**Purpose:** Avoid penny stocks with extreme volatility and low liquidity

#### 2.3.2 Maximum Price Threshold
```
Max Stock Price: $2,000
```

**Purpose:** Prevent overconcentration in high-priced securities (e.g., BRK.A)

### 2.4 Temporal Hard Limits

#### 2.4.1 Order Frequency Limits
```
Max Orders per Symbol per Day: 5
Max Orders per Strategy per Day: 50
Max Orders per Account per Day: 100
```

**Purpose:** Prevent runaway trading algorithms and excessive transaction costs

#### 2.4.2 Position Holding Period
```
Min Holding Period: 24 hours
Max Holding Period: 180 days (alert only, not enforced)
```

**Purpose:** Discourage day trading behavior, encourage position conviction

## 3. Soft Limits System

Soft limits trigger **warnings and gradualtrade size reductions** but don't block orders entirely. They implement progressive risk reduction.

### 3.1 Concentration Limits

#### 3.1.1 Sector Concentration
```
Warning Threshold: 40% of portfolio in single sector
Reduction Threshold: 50% of portfolio in single sector

Action at Reduction Threshold:
- New orders in concentrated sector: size × 0.5
- Prioritize diversification in order queue
```

**Sector Classification:** Use GICS sectors (11 primary sectors)

#### 3.1.2 Correlated Positions
```
Warning Threshold: Positions with correlation > 0.7 exceed 30% of portfolio
Reduction Threshold: Positions with correlation > 0.7 exceed 40% of portfolio

Action:
- Calculate rolling 60-day correlation matrix
- Flag clusters of highly correlated positions
- Reduce order sizes for additions to correlated clusters
```

#### 3.1.3 Congressional Representative Concentration
```
Warning Threshold: 30% of positions based on single representative's trades
Reduction Threshold: 40% of positions based on single representative's trades

Action:
- Track source attribution for each trade signal
- Diversify signal sources
- Reduce confidence in overused sources
```

### 3.2 Volatility Limits

#### 3.2.1 Position Volatility Threshold
```
Warning: Position volatility > 40% annualized
Reduction: Position volatility > 60% annualized

Action at Reduction:
- Reduce target position size by (volatility - 40%) × 2
- Example: 50% vol → 20% reduction, 70% vol → 60% reduction
```

**Volatility Calculation:**
- Use 30-day rolling standard deviation of daily returns
- Annualize: σ_daily × √252

#### 3.2.2 Portfolio Volatility Target
```
Target Portfolio Volatility: 20% annualized
Warning Threshold: 25% annualized
Reduction Threshold: 30% annualized

Action:
- Calculate portfolio volatility using position correlations
- Scale down all new position sizes proportionally
- Scaling factor = min(1.0, target_vol / current_vol)
```

### 3.3 Drawdown-Based Limits

#### 3.3.1 Strategy Drawdown Limits
```
Warning: Strategy down 10% from peak
Reduction: Strategy down 15% from peak
Halt: Strategy down 20% from peak

Actions:
- Warning: Flag for review, no automatic action
- Reduction: Reduce new position sizes by 50%
- Halt: Pause new positions, maintain existing positions
```

#### 3.3.2 Account Drawdown Limits
```
Warning: Account down 10% from peak
Reduction: Account down 15% from peak
Emergency Halt: Account down 25% from peak

Actions:
- Warning: Email notification, dashboard alert
- Reduction: Reduce all position sizes by 30%
- Emergency: Halt all new positions, consider liquidation
```

### 3.4 Liquidity-Based Limits

#### 3.4.1 Average Daily Volume (ADV) Constraint
```
Max Position Size = min(
    Standard Limits,
    ADV_30day × 0.01  // 1% of average daily volume
)

Warning: Position > 0.005 × ADV (0.5%)
Reduction: Position > 0.01 × ADV (1.0%)
```

**Purpose:** Ensure positions can be liquidated without excessive market impact

## 4. Portfolio-Wide Aggregation

### 4.1 Risk Aggregation Architecture

```
┌─────────────────────────────────────────┐
│         Risk Aggregation Engine          │
│                                          │
│  ┌────────────────────────────────────┐ │
│  │     Real-Time Position Monitor     │ │
│  │  - All open positions              │ │
│  │  - Pending orders                  │ │
│  │  - Strategy attributions           │ │
│  └────────────────────────────────────┘ │
│                                          │
│  ┌────────────────────────────────────┐ │
│  │      Exposure Calculator            │ │
│  │  - Gross exposure (sum of longs)   │ │
│  │  - Net exposure (longs - shorts)   │ │
│  │  - Sector exposures                │ │
│  │  - Factor exposures (beta, etc.)   │ │
│  └────────────────────────────────────┘ │
│                                          │
│  ┌────────────────────────────────────┐ │
│  │       Risk Metrics Engine           │ │
│  │  - Portfolio volatility            │ │
│  │  - Value at Risk (VaR)             │ │
│  │  - Correlation matrix              │ │
│  │  - Concentration metrics           │ │
│  └────────────────────────────────────┘ │
│                                          │
│  ┌────────────────────────────────────┐ │
│  │      Limit Enforcement Engine       │ │
│  │  - Check all limits pre-trade      │ │
│  │  - Apply soft limit adjustments    │ │
│  │  - Reject hard limit violations    │ │
│  └────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

### 4.2 Cross-Strategy Risk Coordination

#### 4.2.1 Position Overlap Detection
```
Problem: Multiple strategies want to trade the same symbol
Solution: Aggregate signals and create unified position

Combined Signal Strength = weighted_average(
    strategy_signals,
    weights = strategy_performance_scores
)

Combined Position Size = min(
    sum(individual_strategy_sizes),
    symbol_position_limit
)
```

#### 4.2.2 Conflict Resolution Priority
1. **Hard limits:** Always enforced first
2. **Existing positions:** Reduce first before blocking new entries
3. **Strategy performance:** Higher-performing strategies get priority
4. **Time priority:** First signal gets priority if tie-breaking needed

### 4.3 Real-Time Risk Dashboard Metrics

**Critical Metrics to Display:**
1. Cash available / Buying power used
2. Number of positions / Position limit
3. Gross exposure / Net exposure
4. Portfolio volatility (30-day rolling)
5. Largest position (% of portfolio)
6. Current drawdown from peak
7. Sector concentration (top 3 sectors)
8. Active limit warnings/violations

## 5. Dynamic Adjustment System

### 5.1 Performance-Based Adjustments

#### 5.1.1 Strategy Performance Tiers
```
Tier 1 (Excellent): Sharpe > 1.5, Drawdown < 10%
  → Allocation: 1.0× baseline (no adjustment)

Tier 2 (Good): Sharpe > 0.8, Drawdown < 15%
  → Allocation: 0.8× baseline

Tier 3 (Acceptable): Sharpe > 0.3, Drawdown < 20%
  → Allocation: 0.5× baseline

Tier 4 (Poor): Below Tier 3 thresholds
  → Allocation: 0.0× baseline (paused)
```

**Evaluation Period:** Rolling 30-day window, updated daily

#### 5.1.2 Adaptive Position Sizing
```
Adjusted Position Size = Base Size × Performance Multiplier × Volatility Adjustment

Performance Multiplier = sqrt(max(0.1, sharpe_ratio_30d / 1.0))
  - Scales position size with risk-adjusted returns
  - Minimum 0.1× to avoid complete shutdown
  - Normalized to 1.0 at Sharpe = 1.0

Volatility Adjustment = sqrt(target_vol / realized_vol)
  - Inverse volatility scaling
  - Higher vol → smaller positions
```

### 5.2 Market Regime Detection

#### 5.2.1 Volatility Regime Classification
```
Low Volatility: VIX < 15
  → Risk multiplier: 1.2×

Normal Volatility: VIX 15-25
  → Risk multiplier: 1.0×

High Volatility: VIX 25-35
  → Risk multiplier: 0.7×

Extreme Volatility: VIX > 35
  → Risk multiplier: 0.4×
```

**Application:** Multiply all soft limits by regime multiplier

#### 5.2.2 Market Trend Detection
```
Strong Uptrend: SPY 50-day MA > 200-day MA, both rising
  → Net exposure target: 0% to +40% (allow more long bias)

Neutral: Mixed signals
  → Net exposure target: -10% to +10%

Strong Downtrend: SPY 50-day MA < 200-day MA, both falling
  → Net exposure target: -40% to 0% (allow more short bias)
```

### 5.3 Feedback Loop Integration

#### 5.3.1 Slippage and Execution Quality Monitoring
```
If realized slippage > expected slippage by 50%:
  → Reduce position sizes for that symbol
  → Flag low liquidity warning
  → Tighten ADV constraints
```

#### 5.3.2 Correlation Breakdown Detection
```
If realized correlation differs from predicted by > 0.3:
  → Recalculate correlation matrix
  → Adjust portfolio volatility estimates
  → Recompute concentration limits
```

## 6. Learning vs Safety Balance

### 6.1 Paper Trading Philosophy

**Primary Objective:** Maximize learning while maintaining realistic constraints

**Key Principle:** Risk limits should be loose enough to allow strategies to make mistakes and learn, but tight enough that learnings translate to real trading.

### 6.2 Progressive Risk Allowance

#### 6.2.1 Three-Phase Risk Expansion
```
Phase 1 (Initial - Days 1-30):
  - Conservative limits: 50% of full limits
  - Focus: Validate basic functionality, catch bugs
  - Max position: $7,500
  - Max strategies: 2

Phase 2 (Validation - Days 31-90):
  - Standard limits: 100% of specified limits
  - Focus: Strategy optimization, parameter tuning
  - Max position: $15,000
  - Max strategies: 3

Phase 3 (Advanced - Days 91+):
  - Relaxed limits: 150% of standard limits
  - Focus: Stress testing, edge case exploration
  - Max position: $22,500
  - Max strategies: 5
  - Enable short selling (if supported)
```

#### 6.2.2 Graduation Criteria
```
Advance from Phase 1 to Phase 2 if:
  ✓ No system errors in 14 days
  ✓ At least 30 trades executed
  ✓ No hard limit violations
  ✓ Drawdown < 15%

Advance from Phase 2 to Phase 3 if:
  ✓ Positive Sharpe ratio > 0.5
  ✓ At least 100 trades executed
  ✓ Max drawdown < 20%
  ✓ Less than 5 soft limit warnings per week
```

### 6.3 Learning-Oriented Exceptions

#### 6.3.1 Exploration Mode
```
Optional "Exploration Mode" for specific experiments:
  - Time-limited (e.g., 1 week)
  - Isolated capital allocation ($10k max)
  - Relaxed position limits (+50%)
  - Enhanced logging for analysis
  - Automatic termination after period
```

**Use Cases:**
- Testing new strategy variations
- Exploring unusual market conditions
- Validating edge cases

#### 6.3.2 Strategy Sandbox
```
New strategies start in isolated sandbox:
  - Separate capital allocation ($5k)
  - Cannot interact with main portfolio
  - Standard limits apply within sandbox
  - Promotion to main portfolio after 50+ trades with Sharpe > 0.3
```

## 7. Implementation Considerations

### 7.1 Data Requirements

**Position Data:**
- Symbol, quantity, entry price, current price
- Entry timestamp, strategy ID
- Cost basis, unrealized P&L
- Market data: last price, bid/ask, volume

**Market Data:**
- Real-time prices (for paper trading simulation)
- Historical volatility (30-day, 60-day)
- Average daily volume (30-day)
- Sector classification
- Correlation data (rolling 60-day)

**Risk Metrics:**
- Account equity (updated real-time)
- Cash balance
- Buying power
- Gross/net exposure
- Portfolio volatility
- Drawdown from peak
- VaR estimate

### 7.2 Performance Requirements

**Latency:**
- Pre-trade risk check: < 50ms (p99)
- Portfolio risk calculation: < 200ms (p99)
- Risk dashboard update: < 1 second

**Accuracy:**
- Position valuation: Real-time or 1-minute delayed
- Volatility estimates: Daily updates acceptable
- Correlation matrix: Daily updates acceptable

### 7.3 Monitoring and Alerts

**Alert Levels:**
1. **INFO:** Soft limit warning threshold reached
2. **WARN:** Soft limit reduction threshold reached
3. **ERROR:** Hard limit violation attempted
4. **CRITICAL:** Emergency halt triggered

**Alert Channels:**
- Dashboard visual indicators
- Log files (structured JSON)
- Email notifications (daily summary + critical alerts)
- Slack/Discord integration (optional)

### 7.4 Configuration Management

**Principles:**
- All limits stored in configuration file (YAML/TOML)
- Version controlled
- Hot-reload supported (with validation)
- Override mechanism for emergencies
- Audit log of all configuration changes

**Example Configuration Structure:**
```yaml
risk_framework:
  version: "1.0"

  hard_limits:
    account:
      initial_capital: 100000
      margin_multiplier: 1.0
      reserve_ratio: 0.05

    position:
      max_position_pct: 0.15
      max_position_dollars: 25000
      min_position_dollars: 500

    strategy:
      max_strategy_allocation_pct: 0.40
      max_strategies_active: 3
      max_positions_per_strategy: 10

  soft_limits:
    concentration:
      sector_warning_pct: 0.40
      sector_reduction_pct: 0.50

    volatility:
      position_volatility_warning: 0.40
      position_volatility_reduction: 0.60
      portfolio_volatility_target: 0.20

  dynamic_adjustments:
    performance_tiers_enabled: true
    market_regime_enabled: true
    volatility_regime_enabled: true

  learning_phases:
    current_phase: 1
    phase_1_duration_days: 30
    phase_2_duration_days: 60
```

### 7.5 Testing Strategy

**Unit Tests:**
- Individual limit calculations
- Aggregation logic
- Tier classification
- Configuration parsing

**Integration Tests:**
- Multi-strategy position aggregation
- Limit enforcement pipeline
- Dynamic adjustment calculations

**Simulation Tests:**
- Historical backtest with risk limits
- Stress testing with extreme scenarios
- Limit violation scenarios

**Manual Testing:**
- Dashboard usability
- Alert notification flow
- Configuration hot-reload

## 8. Future Enhancements

### 8.1 Short-Term (Phase 2/3)
1. **Short selling support** with separate risk limits
2. **Options trading** with delta-adjusted position sizing
3. **Stop-loss automation** integrated with risk limits
4. **Tax-loss harvesting** coordination

### 8.2 Long-Term (Post Paper Trading)
1. **Real money integration** with broker APIs
2. **Regulatory compliance** (pattern day trader rules, etc.)
3. **Multi-account support** (IRA, taxable, etc.)
4. **Advanced risk metrics** (CVaR, Expected Shortfall)
5. **Machine learning** for dynamic limit optimization

## 9. Risk Framework API Specification

### 9.1 Core Interfaces

```go
// RiskManager is the main interface for risk management
type RiskManager interface {
    // Pre-trade validation
    ValidateOrder(order Order) (ValidationResult, error)

    // Position management
    GetAvailableBuyingPower() (float64, error)
    GetPositionLimit(symbol string) (float64, error)

    // Portfolio metrics
    GetPortfolioMetrics() (PortfolioMetrics, error)
    GetStrategyMetrics(strategyID string) (StrategyMetrics, error)

    // Dynamic adjustments
    GetCurrentPhase() Phase
    GetRiskMultiplier(strategyID string) (float64, error)

    // Configuration
    UpdateConfig(config RiskConfig) error
    GetConfig() RiskConfig
}

type ValidationResult struct {
    Approved     bool
    AdjustedSize float64  // If soft limits triggered
    Warnings     []string
    Rejections   []string
    Metrics      map[string]float64
}

type PortfolioMetrics struct {
    TotalEquity      float64
    CashBalance      float64
    BuyingPower      float64
    GrossExposure    float64
    NetExposure      float64
    NumPositions     int
    Volatility30D    float64
    CurrentDrawdown  float64
    PeakEquity       float64
    SectorExposures  map[string]float64
    LargestPosition  PositionSummary
}

type Order struct {
    Symbol      string
    Quantity    int
    Side        Side  // Buy, Sell, Short, Cover
    OrderType   OrderType
    LimitPrice  *float64
    StrategyID  string
    Timestamp   time.Time
}
```

### 9.2 Integration Points

**Pre-Trade Hook:**
```
Strategy → Order Generation → RiskManager.ValidateOrder() → Execution Engine
                                      ↓
                                  Rejected/Adjusted
                                      ↓
                                Strategy Feedback
```

**Post-Trade Hook:**
```
Execution Confirmation → Portfolio Update → RiskManager.RecalculateMetrics()
                                                    ↓
                                            Update Dashboard
                                            Check Soft Limits
                                            Trigger Alerts
```

**Periodic Updates:**
```
Market Data Feed → Volatility Calculator → Correlation Matrix → Risk Metrics
                                                                      ↓
                                                              Dashboard Update
                                                              Limit Recalculation
```

## 10. Success Metrics

### 10.1 Risk Framework Effectiveness

**Primary Metrics:**
1. **Hard Limit Violation Rate:** Target < 0.01% (should be near zero)
2. **Soft Limit Warning Rate:** Target 2-5% (enough to guide, not overwhelming)
3. **Maximum Drawdown:** Target < 25% (emergency halt threshold)
4. **Strategy Halt Rate:** Target < 10% of strategies halted

**Secondary Metrics:**
5. **Risk-Adjusted Returns:** Sharpe ratio improvement vs. no risk limits
6. **Diversification Score:** Effective number of positions (portfolio entropy)
7. **Limit Override Frequency:** Manual overrides should be rare (<1/month)
8. **Configuration Stability:** Changes to risk config < 1/week after initial tuning

### 10.2 Learning Effectiveness

**Learning Metrics:**
1. **Strategy Graduation Rate:** >50% of strategies advance from Phase 1 to Phase 2
2. **Exploration Efficiency:** Positive discoveries in exploration mode
3. **Parameter Convergence:** Risk parameters stabilize over time
4. **Incident Learning:** Post-incident adjustments reduce similar violations

## Appendix A: Risk Event Response Playbook

### A.1 Hard Limit Violation
```
Event: Order rejected due to hard limit
Response:
  1. Log detailed rejection reason
  2. Notify strategy (if automated)
  3. Dashboard alert (INFO level)
  4. No further action needed (system working as designed)
```

### A.2 Soft Limit Threshold Breach
```
Event: Portfolio concentration exceeds soft limit
Response:
  1. Calculate adjustment factor
  2. Apply to subsequent orders automatically
  3. Dashboard warning (WARN level)
  4. Daily summary email
  5. Review if persists >3 days
```

### A.3 Strategy Drawdown Halt
```
Event: Strategy hits 20% drawdown (halt threshold)
Response:
  1. Immediately pause new positions for strategy
  2. Maintain existing positions (no forced liquidation)
  3. Email alert to operator
  4. Dashboard CRITICAL alert
  5. Require manual review and approval to resume
  6. Document incident and root cause
```

### A.4 Account Emergency Halt
```
Event: Account hits 25% drawdown
Response:
  1. HALT all new positions across all strategies
  2. Do NOT automatically liquidate (preserve for analysis)
  3. Immediate email + Slack alert
  4. Generate incident report with:
     - Timeline of losses
     - Strategy attribution
     - Positions breakdown
     - Risk metrics history
  5. Manual review required to resume
  6. Consider reducing limits for restart
```

### A.5 Configuration Change
```
Event: Risk configuration updated
Response:
  1. Validate new configuration (schema + sanity checks)
  2. Log change with diff and operator identity
  3. Hot-reload if validation passes
  4. Recalculate all risk metrics with new parameters
  5. Notify all active strategies of parameter changes
  6. Monitor for unexpected behavior in next hour
```

## Appendix B: Risk Calculation Examples

### B.1 Position Limit Calculation
```
Account Value: $100,000
Max Position %: 15%
Max Position $: $25,000

Calculation:
  Limit = min($100,000 × 0.15, $25,000)
        = min($15,000, $25,000)
        = $15,000

For symbol AAPL @ $180:
  Max Shares = floor($15,000 / $180)
             = floor(83.33)
             = 83 shares
  Actual Position Value = 83 × $180 = $14,940
```

### B.2 Volatility-Adjusted Position Sizing
```
Symbol: NVDA
Base Position Limit: $15,000
30-day Volatility: 50% annualized
Soft Limit Reduction Threshold: 60%
Target Volatility: 40%

Since 50% < 60%, no soft limit triggered
But dynamic adjustment applies:

Volatility Adjustment = sqrt(0.40 / 0.50)
                     = sqrt(0.80)
                     = 0.894

Adjusted Limit = $15,000 × 0.894
              = $13,410

At price $500:
  Max Shares = floor($13,410 / $500)
             = 26 shares
```

### B.3 Multi-Strategy Position Aggregation
```
Symbol: MSFT @ $400
Strategy A wants to buy: 30 shares ($12,000)
Strategy B wants to buy: 25 shares ($10,000)

Account position limit: $15,000
Current position: 0 shares

Naive sum: 55 shares × $400 = $22,000 (EXCEEDS LIMIT)

Solution 1 - Proportional Reduction:
  Total desired: $22,000
  Reduction factor: $15,000 / $22,000 = 0.682

  Strategy A gets: 30 × 0.682 = 20 shares
  Strategy B gets: 25 × 0.682 = 17 shares
  Total: 37 shares × $400 = $14,800 ✓

Solution 2 - Performance-Weighted:
  Strategy A Sharpe: 1.2
  Strategy B Sharpe: 0.8

  Weight A: 1.2 / (1.2 + 0.8) = 0.60
  Weight B: 0.8 / (1.2 + 0.8) = 0.40

  Strategy A gets: $15,000 × 0.60 = $9,000 = 22 shares
  Strategy B gets: $15,000 × 0.40 = $6,000 = 15 shares
  Total: 37 shares × $400 = $14,800 ✓
```

## Appendix C: Glossary

**Annualized Volatility:** Standard deviation of returns scaled to one year (daily_vol × √252)

**Average Daily Volume (ADV):** Mean number of shares traded per day over a period (typically 30 days)

**Buying Power:** Capital available for new positions (cash + margin - required reserves)

**Correlation:** Statistical measure of how two securities move together (-1 to +1)

**Drawdown:** Decline from peak equity to trough, expressed as percentage

**Gross Exposure:** Sum of absolute values of all positions (|longs| + |shorts|)

**Hard Limit:** Strict boundary that cannot be exceeded; violations result in order rejection

**Net Exposure:** Difference between long and short positions (longs - shorts)

**Sharpe Ratio:** Risk-adjusted return metric = (return - risk_free_rate) / volatility

**Slippage:** Difference between expected execution price and actual execution price

**Soft Limit:** Warning threshold that triggers gradual risk reduction, not rejection

**Value at Risk (VaR):** Estimated maximum loss over a time period at a confidence level

**Volatility:** Measure of price fluctuation, typically standard deviation of returns

---

**End of Risk Management Framework Specification**
