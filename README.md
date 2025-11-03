import requests
import pandas as pd
import numpy as np
import time
import datetime

# ======== Discord設定 ========
WEBHOOK_BTC = "https://discord.com/api/webhooks/xxxxxx"
WEBHOOK_GOLD = "https://discord.com/api/webhooks/yyyyyy"

INTERVAL = "1h"
LIMIT = 500
RSI_PERIOD = 14
MA_SHORT = 20
MA_LONG = 50


# ======== データ取得 ========
def get_data(symbol, interval=INTERVAL, limit=LIMIT):
    if symbol == "BTCUSDT":
        url = f"https://api.binance.com/api/v3/klines?symbol={symbol}&interval={interval}&limit={limit}"
        data = requests.get(url).json()
        df = pd.DataFrame(data, columns=[
            "timestamp", "open", "high", "low", "close", "volume",
            "close_time", "quote_asset_volume", "trades",
            "taker_base", "taker_quote", "ignore"
        ])
        df["timestamp"] = pd.to_datetime(df["timestamp"], unit="ms")
        df["close"] = df["close"].astype(float)
        df["volume"] = df["volume"].astype(float)
        return df

    elif symbol == "XAUUSD":
        # metals.liveから金のスポット価格を取得
        data = requests.get("https://api.metals.live/v1/spot/gold").json()
        df = pd.DataFrame([{"timestamp": datetime.datetime.now(), "close": float(data[0]["price"]), "volume": np.nan}])
        return df
    return None


# ======== テクニカル指標 ========
def calc_indicators(df):
    df["MA20"] = df["close"].rolling(MA_SHORT).mean()
    df["MA50"] = df["close"].rolling(MA_LONG).mean()

    delta = df["close"].diff()
    gain = np.maximum(delta, 0)
    loss = np.abs(np.minimum(delta, 0))
    avg_gain = gain.rolling(RSI_PERIOD).mean()
    avg_loss = loss.rolling(RSI_PERIOD).mean()
    rs = avg_gain / avg_loss
    df["RSI"] = 100 - (100 / (1 + rs))

    ema12 = df["close"].ewm(span=12, adjust=False).mean()
    ema26 = df["close"].ewm(span=26, adjust=False).mean()
    df["MACD"] = ema12 - ema26
    df["Signal"] = df["MACD"].ewm(span=9, adjust=False).mean()
    df["MACD_Cross"] = np.where(
        (df["MACD"].shift(1) < df["Signal"].shift(1)) & (df["MACD"] > df["Signal"]), "BUY",
        np.where((df["MACD"].shift(1) > df["Signal"].shift(1)) & (df["MACD"] < df["Signal"]), "SELL", None)
    )
    return df


# ======== バックテストロジック ========
def backtest(df, lookahead=4):
    results = []
    for i in range(len(df) - lookahead):
        row = df.iloc[i]
        future = df.iloc[i + lookahead]
        price_now = row["close"]
        price_future = future["close"]

        if row["RSI"] < 30:
            signal = "BUY_RSI"
        elif row["RSI"] > 70:
            signal = "SELL_RSI"
        elif row["MA20"] > row["MA50"] and df["MA20"].shift(1).iloc[i] < df["MA50"].shift(1).iloc[i]:
            signal = "BUY_MAcross"
        elif row["MA20"] < row["MA50"] and df["MA20"].shift(1).iloc[i] > df["MA50"].shift(1).iloc[i]:
            signal = "SELL_MAcross"
        elif row["MACD_Cross"] == "BUY":
            signal = "BUY_MACD"
        elif row["MACD_Cross"] == "SELL":
            signal = "SELL_MACD"
        else:
            continue

        change = (price_future - price_now) / price_now * 100
        results.append({"Time": row["timestamp"], "Signal": signal, "Change%": change})

    df_result = pd.DataFrame(results)
    summary = df_result.groupby("Signal")["Change%"].agg(["count", "mean", "median", "std"])
    summary["WinRate(%)"] = (df_result.groupby("Signal")["Change%"].apply(lambda x: (x > 0).mean()) * 100)
    return summary.round(2)


# ======== Discord送信 ========
def send_report_to_discord(symbol, webhook, summary):
    today = datetime.date.today()
    text = f"📊 **{symbol} バックテストレポート** ({today})\n\n"
    text += "```" + summary.to_string() + "```\n"
    text += "⚠️ 教育目的・参考目的の情報提供です。実際の取引判断は自己責任で行ってください。"
    requests.post(webhook, json={"content": text})


# ======== メイン処理 ========
def run_reports():
    for symbol, webhook in [("BTCUSDT", WEBHOOK_BTC), ("XAUUSD", WEBHOOK_GOLD)]:
        df = get_data(symbol)
        df = calc_indicators(df)
        summary = backtest(df)
        send_report_to_discord(symbol, webhook, summary)
        print(f"{symbol} のバックテストレポートを送信しました。")


if __name__ == "__main__":
    while True:
        now = datetime.datetime.now()

        # 例：毎週月曜の午前9時に実行
        if now.weekday() == 0 and now.hour == 9 and now.minute == 0:
            run_reports()
            time.sleep(60)  # 1分待機（多重送信防止）

        time.sleep(30)

