def trading_loop(symbol, amount, duration=60):
    while True:
        # Step 1: Get market data
        market_data = get_market_data(symbol)
        
        # Step 2: Calculate moving averages
        short_ma = calculate_moving_average(market_data['close'], 5)  # 5-period MA
        long_ma = calculate_moving_average(market_data['close'], 20)  # 20-period MA
        
        # Step 3: Check if a signal to trade has been triggered
        action = check_trade_signal(short_ma, long_ma)
        
        if action:
            # Step 4: Place trade
            trade_response = place_trade(symbol, amount, action, duration)
            print(f"Placed {action} order. Response: {trade_response}")
            
            # Step 5: Manage risk (stop-loss or take-profit)
            entry_price = market_data['close'][-1]
            while True:
                current_price = get_current_price(symbol)
                risk_status = manage_risk(entry_price, current_price)
                if risk_status in ['stop-loss', 'take-profit']:
                    print(f"Exit signal received: {risk_status}")
                    exit_trade(symbol)
                    break
                time.sleep(10)  # Check price every 10 seconds

        time.sleep(60)  # Run every minute (or adjust as needed)
STOP_LOSS_PERCENTAGE = 0.02  # 2% stop loss
TAKE_PROFIT_PERCENTAGE = 0.05  # 5% take profit

def manage_risk(entry_price, current_price):
    loss_threshold = entry_price * (1 - STOP_LOSS_PERCENTAGE)
    profit_threshold = entry_price * (1 + TAKE_PROFIT_PERCENTAGE)
    
    if current_price <= loss_threshold:
        return 'stop-loss'
    elif current_price >= profit_threshold:
        return 'take-profit'
    return None
import numpy as np
import pandas as pd

def calculate_moving_average(prices, window):
    return np.convolve(prices, np.ones(window)/window, mode='valid')

def check_trade_signal(short_ma, long_ma):
    if short_ma[-1] > long_ma[-1]:
        return 'buy'
    elif short_ma[-1] < long_ma[-1]:
        return 'sell'
    return None

def get_market_data(symbol, duration=60):
    # This function would ideally fetch historical data from the Deriv API or any other source
    # For simplicity, let's assume `get_historical_data()` is a function that gets the last n prices
    data = get_historical_data(symbol, duration)  # Function to get historical data
    return data
import requests
import time

API_KEY = 'your_api_key_here'
BASE_URL = 'https://api.deriv.com/api/v2/'

# Example of getting account information
def get_account_info():
    url = BASE_URL + 'get_account_info'
    params = {'api_key': API_KEY}
    response = requests.get(url, params=params)
    return response.json()

# Example of placing a trade
def place_trade(symbol, amount, action, duration):
    url = BASE_URL + 'place_order'
    params = {
        'api_key': API_KEY,
        'symbol': symbol,
        'amount': amount,
        'action': action,  # 'buy' or 'sell'
        'duration': duration,  # In seconds or ticks
    }
    response = requests.post(url, params=params)
    return response.json()
pip install requests numpy pandas matplotlib
import ccxt
import time
import pandas as pd
import talib as ta

# Define exchange and initialize with your API credentials
exchange = ccxt.binance({
    'apiKey': 'your_api_key',
    'secret': 'your_api_secret',
    'enableRateLimit': True,  # Prevents hitting rate limit
})

# Function to fetch historical data (OHLCV)
def fetch_data(symbol, timeframe='5m', limit=100):
    ohlcv = exchange.fetch_ohlcv(symbol, timeframe=timeframe, limit=limit)
    df = pd.DataFrame(ohlcv, columns=['timestamp', 'open', 'high', 'low', 'close', 'volume'])
    df['timestamp'] = pd.to_datetime(df['timestamp'], unit='ms')
    return df

# Basic strategy: Moving Average Crossover
def moving_average_crossover(df):
    df['sma_short'] = ta.SMA(df['close'], timeperiod=50)  # Short-term moving average
    df['sma_long'] = ta.SMA(df['close'], timeperiod=200)  # Long-term moving average
    
    # Signal: Buy when short MA crosses above long MA, sell when vice versa
    df['signal'] = 0
    df.loc[df['sma_short'] > df['sma_long'], 'signal'] = 1  # Buy signal
    df.loc[df['sma_short'] < df['sma_long'], 'signal'] = -1  # Sell signal

    return df

# Function to place an order
def place_order(symbol, side, amount, price=None):
    if side == 'buy':
        if price:
            order = exchange.create_limit_buy_order(symbol, amount, price)
        else:
            order = exchange.create_market_buy_order(symbol, amount)
    elif side == 'sell':
        if price:
            order = exchange.create_limit_sell_order(symbol, amount, price)
        else:
            order = exchange.create_market_sell_order(symbol, amount)
    return order

# Function to trade based on strategy
def trade(symbol, amount):
    df = fetch_data(symbol)
    df = moving_average_crossover(df)

    latest_signal = df.iloc[-1]['signal']
    if latest_signal == 1:  # Buy signal
        print("Placing Buy Order...")
        place_order(symbol, 'buy', amount)
    elif latest_signal == -1:  # Sell signal
        print("Placing Sell Order...")
        place_order(symbol, 'sell', amount)

# Main loop to run the bot every few minutes
symbol = 'BTC/USDT'  # Example: BTC/USDT perpetual futures
amount = 0.01  # Amount to trade

while True:
    try:
        trade(symbol, amount)
    except Exception as e:
        print(f"Error: {e}")
    time.sleep(60)  # Wait for the next cycle

