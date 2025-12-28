# 🤖 TRADING BOT PROJECT SPECIFICATION

---

## PROJECT: End-to-End ML Trading System

**Project ID:** TRADING-ROBOT-001
**Difficulty:** 8/10  
**Type:** Full integration of Data Engineering + MLOps

---

### 📋 EXECUTIVE SUMMARY

Build production-grade algorithmic trading system from data ingestion to live model serving. Learn MLOps + data engineering in financial domain with real constraints:
- Real market data (OHLCV)
- Real risk metrics (Sharpe, drawdown)
- Real-time predictions
- Portfolio optimization

---

### 🎯 BUSINESS CONTEXT

**Problem:**
- Manual trading is slow, emotional, inconsistent
- Markets move 24/7, humans need sleep
- Consistent alpha (outperformance) requires systematic approach

**Goal:**
- Build model that predicts 1-day market movements (up/down)
- Trade automatically
- Target Sharpe ratio > 1.5 (good risk-adjusted returns)
- Minimize drawdown (max losing period)

**Reality Check:**
- Most trading bots underperform market
- Overfitting to history is huge risk
- This project teaches why via real data

---

### 🏗️ ARCHITECTURE

```
Data Layer (Data Engineering)
  ↓
Feature Engineering (MLOps)
  ↓
Model Training & Selection (MLOps)
  ↓
Backtesting (Finance metrics)
  ↓
Live Deployment (Trading execution)
  ↓
Monitoring & Retraining (Continuous learning)
```

---

# PHASE 1: DATA INFRASTRUCTURE (Weeks 1-3, 24 hours)

---

## Step 1.1: Market Data Ingestion

**Real Data Sources:**

```python
# Option A: Free (Recommended for learning)
import yfinance as yf
import pandas as pd

# Download real OHLCV data
data = yf.download(
    tickers=['SPY', 'QQQ', 'IWM'],  # S&P500, Tech, Russell 2000
    start='2020-01-01',
    end='2024-01-01',
    interval='1d'  # Daily data
)

# data structure:
#             Open     High      Low    Close    Volume
# Date
# 2020-01-01  324.80  324.94  322.54  323.17  47000000
```

**Option B: Professional (For production)**
```
- Alpaca API (commission-free, real-time)
- Polygon.io (comprehensive financial data)
- Alternative.me (options flow, sentiment)
```

**What You'll Learn:**
- Data quality (handling splits, dividends)
- Time-series data handling
- Aligned multi-asset data

---

## Step 1.2: Feature Engineering for Trading

```python
# File: features/market_features.py

class TradingFeatureEngineer:
    def compute_features(self, price_data: pd.DataFrame) -> pd.DataFrame:
        """
        Create features that might predict next day's return
        
        Critical: NO LOOK-AHEAD BIAS
        (All features use only past data)
        """
        
        df = price_data.copy()
        
        # 1. MOMENTUM (past performance)
        df['rsi_14'] = self.relative_strength_index(df['Close'], 14)
        # RSI > 70 = overbought (might go down)
        # RSI < 30 = oversold (might go up)
        
        df['macd'] = self.macd_signal(df['Close'])
        # MACD > signal = bullish momentum
        
        # 2. VOLATILITY (market uncertainty)
        df['volatility_20'] = df['Close'].pct_change().rolling(20).std()
        # High volatility = higher risk
        
        # 3. MEAN REVERSION (price goes back to average)
        df['price_vs_sma'] = df['Close'] / df['Close'].rolling(20).mean() - 1
        # Positive = above average (might fall)
        # Negative = below average (might rise)
        
        # 4. VOLUME (buying/selling pressure)
        df['volume_ratio'] = df['Volume'] / df['Volume'].rolling(20).mean()
        # High volume = strong conviction
        
        # 5. MARKET REGIME (bull vs bear)
        df['in_uptrend'] = df['Close'] > df['Close'].rolling(50).mean()
        # True = bull market, False = bear market
        
        # TARGET: Tomorrow's return (what we want to predict)
        df['target'] = df['Close'].pct_change().shift(-1)
        # Shift(-1) = TOMORROW's return (not today)
        
        return df
```

**Critical Concept: Data Leakage**

```python
# ❌ WRONG (uses future data):
df['target'] = df['Close'].pct_change()  # Today's return
# Now you can predict today using today's data!
# This is useless for trading (can't predict past)

# ✅ CORRECT (shift for tomorrow):
df['target'] = df['Close'].pct_change().shift(-1)
# Tomorrow's return
# You predict it today using only today's features
```

**Data Pipeline (Airflow):**

```python
# dags/daily_market_data.py

from airflow import DAG
from airflow.operators.python import PythonOperator

dag = DAG('daily_market_data', schedule_interval='0 17 * * MON-FRI')
# Run at 5pm every weekday (after market closes)

download_task = PythonOperator(
    task_id='download_market_data',
    python_callable=download_ohlcv,
    op_kwargs={'date': '{{ ds }}'}
)

feature_task = PythonOperator(
    task_id='compute_features',
    python_callable=compute_trading_features,
)

store_task = PythonOperator(
    task_id='store_to_warehouse',
    python_callable=store_features,
)

download_task >> feature_task >> store_task
```

---

## Step 1.3: Feature Store Setup

```python
# Store features in PostgreSQL for quick access

CREATE TABLE market_features (
    date DATE NOT NULL,
    ticker VARCHAR(10) NOT NULL,
    close DECIMAL(10, 2),
    rsi_14 DECIMAL(5, 2),
    macd DECIMAL(10, 6),
    volatility_20 DECIMAL(10, 6),
    price_vs_sma DECIMAL(10, 6),
    volume_ratio DECIMAL(10, 2),
    in_uptrend BOOLEAN,
    target DECIMAL(10, 6),  -- Tomorrow's return
    created_at TIMESTAMP,
    PRIMARY KEY (date, ticker)
);

CREATE INDEX idx_market_features_date_ticker 
ON market_features(date, ticker);
```

---

# PHASE 2: MODEL DEVELOPMENT (Weeks 4-8, 40 hours)

---

## Step 2.1: Data Preparation

```python
# File: ml/data_prep.py

import pandas as pd
from sklearn.preprocessing import StandardScaler

class DataPreparation:
    def prepare_for_training(self, df: pd.DataFrame):
        """
        Split data properly for time-series
        
        Critical: TEMPORAL SPLIT (not random!)
        """
        
        df = df.dropna()  # Remove NaN rows
        
        # TIME-BASED SPLIT (correct for time-series!)
        split_date = '2023-01-01'  # Train on older data
        
        train_df = df[df.index < split_date]
        test_df = df[df.index >= split_date]
        
        # Separate features and target
        X_train = train_df.drop('target', axis=1)
        y_train = train_df['target']
        
        X_test = test_df.drop('target', axis=1)
        y_test = test_df['target']
        
        # Scale features (learned from TRAIN only!)
        scaler = StandardScaler()
        X_train_scaled = scaler.fit_transform(X_train)
        X_test_scaled = scaler.transform(X_test)
        
        return X_train_scaled, X_test_scaled, y_train, y_test, scaler
```

---

## Step 2.2: Model Training

```python
# File: ml/model.py

import mlflow
from sklearn.ensemble import GradientBoostingRegressor
from sklearn.model_selection import cross_val_score

def train_trading_model(X_train, y_train):
    """
    Train model to predict next day's return
    """
    
    with mlflow.start_run(run_name="trading_model_v1"):
        model = GradientBoostingRegressor(
            n_estimators=100,
            learning_rate=0.05,
            max_depth=5,
            random_state=42,
            subsample=0.8,
        )
        
        # Cross-validation (test on 5 time periods)
        cv_scores = cross_val_score(
            model, 
            X_train, 
            y_train, 
            cv=5,
            scoring='r2'  # R² score
        )
        
        # Train final model
        model.fit(X_train, y_train)
        
        # Log metrics
        mlflow.log_metric("cv_r2_mean", cv_scores.mean())
        mlflow.log_metric("cv_r2_std", cv_scores.std())
        mlflow.sklearn.log_model(model, "model")
        
        return model

# Training on real data (2020-2023)
# Testing on held-out data (2023-2024)
```

---

## Step 2.3: Backtesting (Financial Evaluation)

```python
# File: ml/backtest.py

class TradingBacktest:
    def backtest_strategy(self, predictions, prices, target_values):
        """
        Simulate trading using model predictions
        
        Metrics that matter:
        - Sharpe ratio (risk-adjusted returns)
        - Maximum drawdown (worst losing period)
        - Win rate (% of profitable trades)
        """
        
        # Generate trading signals
        signals = (predictions > 0).astype(int)  # 1=buy, 0=sell
        
        # Calculate returns
        strategy_returns = signals.shift(1) * target_values
        # We trade TOMORROW based on TODAY's prediction
        
        # Metrics
        total_return = (1 + strategy_returns).prod() - 1
        annual_return = total_return / len(strategy_returns) * 252  # 252 trading days/year
        volatility = strategy_returns.std() * np.sqrt(252)
        
        sharpe_ratio = annual_return / volatility if volatility > 0 else 0
        
        # Drawdown (worst losing period)
        cumulative_returns = (1 + strategy_returns).cumprod()
        running_max = cumulative_returns.expanding().max()
        drawdown = (cumulative_returns - running_max) / running_max
        max_drawdown = drawdown.min()
        
        # Win rate
        profitable_days = (strategy_returns > 0).sum()
        win_rate = profitable_days / len(strategy_returns)
        
        return {
            'total_return': total_return,
            'annual_return': annual_return,
            'sharpe_ratio': sharpe_ratio,
            'max_drawdown': max_drawdown,
            'win_rate': win_rate,
            'cumulative_returns': cumulative_returns,
        }

# Result Example (REALISTIC):
# Sharpe ratio: 0.8 (baseline = 0, good = 1+, excellent = 2+)
# Max drawdown: -12% (some losing periods expected)
# Win rate: 52% (slightly better than 50% coin flip)
```

---

# PHASE 3: DEPLOYMENT & MONITORING (Weeks 9-12, 24 hours)

---

## Step 3.1: Real-Time Prediction API

```python
# File: serving/app.py

from fastapi import FastAPI
import joblib
import numpy as np
from datetime import datetime, timedelta

app = FastAPI()

model = joblib.load("models/trading_model.pkl")
scaler = joblib.load("models/scaler.pkl")

@app.post("/predict-next-day")
async def predict_next_day(features: dict):
    """
    Input: Today's market features
    Output: Predicted tomorrow's return + signal
    """
    
    # Extract and scale
    X = np.array([[
        features['rsi_14'],
        features['macd'],
        features['volatility_20'],
        # ... all features
    ]])
    
    X_scaled = scaler.transform(X)
    
    # Predict
    predicted_return = model.predict(X_scaled)[0]
    
    # Signal
    if predicted_return > 0.01:  # Predict > 1% gain
        signal = 'BUY'
    elif predicted_return < -0.01:  # Predict > 1% loss
        signal = 'SELL'
    else:
        signal = 'HOLD'
    
    return {
        'timestamp': datetime.now().isoformat(),
        'predicted_return': predicted_return,
        'signal': signal,
        'confidence': abs(predicted_return),  # How sure are we?
        'model_version': 'v1.2',
    }

@app.get("/backtest-metrics")
async def get_backtest_metrics():
    """
    Return latest backtest performance
    """
    return {
        'sharpe_ratio': 0.87,
        'max_drawdown': -0.12,
        'win_rate': 0.52,
        'data_period': '2023-2024',
    }
```

---

## Step 3.2: Live Trading Execution

```python
# File: trading/executor.py

import alpaca_trade_api as tradeapi

class TradingExecutor:
    def __init__(self, api_key, api_secret):
        self.api = tradeapi.REST(api_key, api_secret)
    
    def execute_signal(self, signal: str, ticker: str, quantity: int):
        """
        Execute trade based on model signal
        
        With RISK LIMITS:
        """
        
        # Get portfolio (ensure we have money)
        account = self.api.get_account()
        
        if account.buying_power < 0:
            print("No buying power! Skipping trade.")
            return
        
        # Execute
        if signal == 'BUY':
            order = self.api.submit_order(
                symbol=ticker,
                qty=quantity,
                side='buy',
                type='market',
                time_in_force='day'
            )
        elif signal == 'SELL':
            order = self.api.submit_order(
                symbol=ticker,
                qty=quantity,
                side='sell',
                type='market',
                time_in_force='day'
            )
        
        return order

# Daily execution (Airflow task)
dag_daily_trading = {
    'predict_signals': 'Call API, get predictions',
    'execute_trades': 'Place orders (with risk limits)',
    'log_trades': 'Record everything for analysis',
    'monitor_positions': 'Check overnight risks',
}
```

---

## Step 3.3: Monitoring & Retraining

```python
# File: monitoring/dashboard.py

class TradingMonitoring:
    def track_performance(self):
        """
        Monitor trading system health
        """
        
        metrics = {
            'live_pnl': self.calculate_pnl(),  # Daily profit/loss
            'win_rate_live': self.win_rate_recent_30days(),
            'drawdown_current': self.current_drawdown(),
            'model_age': self.days_since_retrain(),
        }
        
        alerts = []
        
        # Alert: Model getting old
        if metrics['model_age'] > 30:
            alerts.append("Model > 30 days old, retrain recommended")
        
        # Alert: Performance degrading
        if metrics['win_rate_live'] < 0.48:
            alerts.append("Win rate < 48%, possible regime change")
        
        # Alert: Drawdown too deep
        if metrics['drawdown_current'] < -0.15:
            alerts.append("Drawdown > 15%, reduce position size")
        
        return metrics, alerts

# Monthly retraining
dag_monthly_retrain = {
    'collect_recent_trades': 'Get trades from last 30 days',
    'add_new_labels': 'What actually happened?',
    'train_new_model': 'Train on extended history',
    'evaluate': 'Better than current model?',
    'a_b_test': '50% old model, 50% new for 1 week',
    'promote_if_better': 'If new model wins, promote',
}
```

---

# 🛠️ MODERN TECH STACK

---

## Package Management: uv (NOT poetry)

**Why uv over poetry?**
- ✅ 10x faster (written in Rust)
- ✅ Simpler (less magic)
- ✅ Better for data science (fewer lock issues)
- ✅ Modern standard (2024+)

```bash
# Install uv
pip install uv

# Create project
uv init trading-bot
cd trading-bot

# Add dependencies
uv add pandas numpy scikit-learn fastapi uvicorn

# Install all
uv sync

# Run
uv run python app.py

# Clean
uv pip cache purge
```

**pyproject.toml:**

```toml
[project]
name = "trading-bot"
version = "0.1.0"
description = "ML-powered trading system"
requires-python = ">=3.11"

dependencies = [
    "pandas>=2.0",
    "numpy>=1.24",
    "scikit-learn>=1.3",
    "fastapi>=0.104",
    "uvicorn>=0.24",
    "sqlalchemy>=2.0",
    "yfinance>=0.2.32",
    "alpaca-trade-api>=3.0",
    "mlflow>=2.9",
    "pytest>=7.4",
    "black>=23.11",
    "ruff>=0.1",
]

[tool.uv]
python-version = "3.11"
```

---

## Modern Stack

| Layer | Tool | Why |
|-------|------|-----|
| **Package Mgmt** | uv | Fast, reliable, modern |
| **Python** | 3.11+ | Latest stable |
| **Data** | Polars | 5-10x faster than pandas |
| **ML Training** | scikit-learn | Proven, interpretable |
| **Deep Learning** | PyTorch | If needed, but likely overkill here |
| **Serving** | FastAPI | Modern, fast, async |
| **Database** | PostgreSQL | Production-ready |
| **Orchestration** | Airflow | Standard (or Dagster if budget) |
| **Monitoring** | Prometheus + Grafana | Industry standard |
| **Testing** | pytest | De facto standard |
| **Code Quality** | Ruff + Black | Fastest linters |

---

# 📊 DELIVERABLES

```
trading-bot/
├── data/
│   ├── raw/                          # Raw OHLCV data
│   ├── processed/                    # Features computed
│   └── backtest/                     # Backtest results
├── features/
│   ├── __init__.py
│   └── market_features.py            # Feature engineering
├── ml/
│   ├── data_prep.py                  # Train/test split
│   ├── model.py                      # Model training
│   ├── backtest.py                   # Backtesting
│   └── test_model.py                 # Tests
├── serving/
│   ├── app.py                        # FastAPI
│   ├── test_api.py
│   └── requirements.txt
├── trading/
│   ├── executor.py                   # Order execution
│   ├── risk_manager.py               # Position sizing
│   └── test_executor.py
├── dags/
│   ├── daily_market_data.py          # Data ingestion
│   ├── model_training.py             # Monthly retrain
│   └── trading_execution.py          # Daily trades
├── monitoring/
│   ├── dashboard.py                  # Metrics tracking
│   └── alerts.py                     # Alerting logic
├── notebooks/
│   ├── 01_eda.ipynb                  # Exploration
│   ├── 02_feature_analysis.ipynb
│   └── 03_backtest_analysis.ipynb
├── tests/
│   ├── test_features.py
│   ├── test_model.py
│   ├── test_api.py
│   └── test_backtest.py
├── docker-compose.yml                # Postgres + Redis
├── pyproject.toml                    # UV config
├── uv.lock                           # Lock file
└── README.md
```

---

# ✅ SUCCESS CRITERIA

**Functional:**
- ✅ Fetch real market data daily
- ✅ Compute features without leakage
- ✅ Train model on historical data
- ✅ Backtest shows Sharpe > 0.8
- ✅ API predicts next day < 100ms
- ✅ Execute trades automatically
- ✅ Monitor performance
- ✅ Monthly retraining

**Code Quality:**
- ✅ 90%+ test coverage
- ✅ Type hints on all functions
- ✅ Ruff/Black compliance
- ✅ No data leakage
- ✅ Reproducible (seed set)

**Financial:**
- ✅ Realistic expectations (Sharpe 0.8-1.5, not 5.0)
- ✅ Account for transaction costs
- ✅ Proper risk management
- ✅ Documented assumptions
- ✅ Known limitations

---

# 🎯 REAL DATA SOURCES

## Free (Recommended for learning):

```python
import yfinance as yf

# Major indexes
SPY = yf.download('SPY', start='2020-01-01', end='2024-01-01')
QQQ = yf.download('QQQ', start='2020-01-01', end='2024-01-01')
IWM = yf.download('IWM', start='2020-01-01', end='2024-01-01')

# Individual stocks
AAPL = yf.download('AAPL', start='2020-01-01', end='2024-01-01')
MSFT = yf.download('MSFT', start='2020-01-01', end='2024-01-01')

# Crypto (bonus)
BTC = yf.download('BTC-USD', start='2020-01-01', end='2024-01-01')
```

## Professional (For production):

```
- Alpaca API: Real-time, commission-free
- Polygon.io: Comprehensive historical data
- IQFeed: Professional-grade
- CoinGecko: Free crypto data
```

---

# 📅 REALISTIC TIMELINE

```
Week 1-2:   Data pipeline + EDA = 12h
Week 3-4:   Feature engineering = 12h
Week 5-6:   Model training + tuning = 12h
Week 7-8:   Backtesting + analysis = 12h
Week 9-10:  API serving + testing = 12h
Week 11-12: Live deployment + monitoring = 16h

Total: 96 hours = 12-16 weeks at 6-8h/week
```

---

# ⚠️ REALISTIC EXPECTATIONS

```
❌ "I'll build a bot that beats Warren Buffett"
   → Most bots underperform buy-and-hold

✅ "I'll build a disciplined system that matches S&P500"
   → Realistic (if you achieve this, you've won)

❌ "Sharpe ratio 3.0"
   → Legendary (Buffett achieves 0.76)

✅ "Sharpe ratio 0.8-1.2"
   → Good (beat market in risk-adjusted terms)

❌ "No drawdowns"
   → Impossible (all strategies have losing periods)

✅ "Max drawdown < 15%"
   → Realistic (accept losses, manage them)
```

**This project teaches:**
- How hard it is to generate alpha
- Why most traders fail
- Importance of risk management
- Value of systematic thinking
- Real-world ML constraints

---

# 🎓 LEARNING OUTCOMES

After 12-16 weeks, you'll master:

```
DATA ENGINEERING:
✅ Real-time data pipelines
✅ Feature store design
✅ Handling financial time-series
✅ Data quality for trading

MLOPS:
✅ Production model serving
✅ Continuous retraining
✅ A/B testing in production
✅ Monitoring systems

FINANCE:
✅ Backtesting methodology
✅ Risk metrics (Sharpe, Drawdown)
✅ Order execution
✅ Portfolio management

SOFTWARE:
✅ Full-stack ML system
✅ Real-time constraints
✅ Error handling
✅ Production thinking
```

This is **equivalent to 1-2 years of fintech experience in one project**.

