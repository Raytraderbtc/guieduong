import yfinance as yf
import pandas as pd
import pandas_ta as ta
import numpy as np

class BreakoutPyramidStrategy:
    def __init__(self, df, initial_capital=100000):
        self.df = df.copy()
        self.initial_capital = initial_capital
        # Parameters
        self.lookback_period = 25
        self.ma_period = 20
        self.atr_length = 14
        self.rsi_period = 14
        self.pyramid_levels = 4
        self.atr_spacing = 0.5
        self.risk_percent = 0.01 

    def calculate_indicators(self):
        df = self.df
        df['hh'] = df['High'].rolling(window=self.lookback_period).max()
        df['ll'] = df['Low'].rolling(window=self.lookback_period).min()
        df['ma20'] = ta.sma(df['Close'], length=self.ma_period)
        df['atr'] = ta.atr(df['High'], df['Low'], df['Close'], length=self.atr_length)
        df['atr_ma'] = ta.sma(df['atr'], length=20)
        df['rsi'] = ta.rsi(df['Close'], length=self.rsi_period)
        macd = ta.macd(df['Close'])
        df['macd_line'] = macd.iloc[:, 0]
        df['macd_signal'] = macd.iloc[:, 1]
        return df

    def backtest(self):
        df = self.calculate_indicators().dropna()
        equity = self.initial_capital
        position = 0
        entry_price = 0
        stop_loss = 0
        pyramid_count = 0
        trade_log = []

        for i in range(1, len(df)):
            row = df.iloc[i]
            prev_row = df.iloc[i-1]

            # EXIT LOGIC
            if position > 0:
                tp_price = entry_price + (entry_price - stop_loss) * 2
                if row['Low'] <= stop_loss or row['High'] >= tp_price or row['Close'] < row['ma20']:
                    exit_p = stop_loss if row['Low'] <= stop_loss else row['Close']
                    pnl = (exit_p - entry_price) * position
                    equity += pnl
                    trade_log.append({'type': 'EXIT', 'pnl': pnl, 'equity': equity})
                    position = 0
                    pyramid_count = 0

            # ENTRY LOGIC
            elif row['Close'] > prev_row['hh'] and row['Volume'] > prev_row['Volume'] and row['Close'] > row['ma20']:
                entry_price = row['Close']
                stop_loss = row['ll']
                risk_amt = (equity * self.risk_percent) / self.pyramid_levels
                risk_per_unit = abs(entry_price - stop_loss)
                if risk_per_unit > 0:
                    position = risk_amt / risk_per_unit
                    pyramid_count = 1
                    trade_log.append({'type': 'ENTRY', 'price': entry_price})

        return trade_log, equity

# --- ĐOẠN CHẠY RA KẾT QUẢ ---
print("--- Đang tải dữ liệu BTC từ Yahoo Finance ---")
data = yf.download("BTC-USD", start="2024-01-01", interval="1d")

if not data.empty:
    model = BreakoutPyramidStrategy(data)
    logs, final_equity = model.backtest()
    
    # Tính toán thống kê
    total_trades = [t for t in logs if t['type'] == 'EXIT']
    wins = [t for t in total_trades if t['pnl'] > 0]
    
    print("\n" + "="*30)
    print(f"KẾT QUẢ BACKTEST")
    print("="*30)
    print(f"Vốn ban đầu: 100,000 USD")
    print(f"Vốn cuối cùng: {final_equity:,.2f} USD")
    print(f"Tổng số lệnh đóng: {len(total_trades)}")
    if len(total_trades) > 0:
        print(f"Tỉ lệ thắng (Win Rate): {(len(wins)/len(total_trades))*100:.2f}%")
    print(f"Lợi nhuận ròng: {(final_equity - 100000):,.2f} USD")
    print("="*30)
else:
    print("Không tải được dữ liệu. Hãy kiểm tra kết nối mạng/VPN.")
    
