
#!/usr/bin/env python3
"""
Polymarket BTC 5-Minute Up/Down - Paper Trading Bot
نسخه بهبود یافته بر اساس تحلیل تکنیکال ۱۹ نوامبر ۲۰۲۶
ریسک ثابت: ۱۵ دلار به ازای هر پنجره
"""

import time
import json
import requests
from datetime import datetime, timezone
from collections import deque
import os

# ====================== تنظیمات ======================
RISK_PER_TRADE = 15.0          # دلار ریسک به ازای هر پنجره
POLL_INTERVAL = 3             # ثانیه بین هر آپدیت
LOOKBACK_SECONDS = 30         # برای محاسبه مومنتوم
MIN_EDGE = 0.03               # حداقل edge برای سیگنال
LOG_FILE = "paper_trades.jsonl"

# سطوح کلیدی بر اساس تحلیل امروز (۱۹ نوامبر ۲۰۲۶)
SUPPORT_1 = 63300             # حمایت اصلی کوتاه‌مدت
SUPPORT_2 = 62200
RESISTANCE_1 = 64000          # مقاومت اصلی
RESISTANCE_2 = 64600

GAMMA = "https://gamma-api.polymarket.com"
CLOB = "https://clob.polymarket.com"
BINANCE = "https://api.binance.com/api/v3/ticker/price?symbol=BTCUSDT"

# =====================================================

class PaperPortfolio:
    def __init__(self):
        self.balance = 1000.0
        self.positions = []
        self.daily_pnl = 0.0
        self.total_trades = 0
        self.wins = 0

    def open_position(self, side: str, price: float, size_usd: float, market_slug: str):
        shares = size_usd / price
        cost = size_usd
        self.balance -= cost
        pos = {
            "id": len(self.positions) + 1,
            "slug": market_slug,
            "side": side,
            "entry_price": price,
            "shares": shares,
            "cost": cost,
            "open_time": datetime.now(timezone.utc).isoformat(),
            "status": "open"
        }
        self.positions.append(pos)
        self.total_trades += 1
        self._log(pos)
        return pos

    def close_all_resolved(self, resolved_side: str, market_slug: str):
        for pos in self.positions:
            if pos["status"] == "open" and pos["slug"] == market_slug:
                if pos["side"] == resolved_side:
                    pnl = pos["shares"] * 1.0 - pos["cost"]
                    self.wins += 1
                else:
                    pnl = -pos["cost"]
                pos["pnl"] = pnl
                pos["status"] = "closed"
                pos["close_time"] = datetime.now(timezone.utc).isoformat()
                self.balance += pos["cost"] + pnl
                self.daily_pnl += pnl
                self._log(pos)

    def _log(self, pos):
        with open(LOG_FILE, "a") as f:
            f.write(json.dumps(pos, ensure_ascii=False) + "\n")

    def summary(self):
        open_pos = [p for p in self.positions if p["status"] == "open"]
        return {
            "balance": round(self.balance, 2),
            "daily_pnl": round(self.daily_pnl, 2),
            "total_trades": self.total_trades,
            "wins": self.wins,
            "open_positions": len(open_pos),
            "winrate": round(self.wins / self.total_trades * 100, 1) if self.total_trades else 0
        }


def get_current_window_start() -> int:
    now = int(time.time())
    return (now // 300) * 300


def get_market_by_slug(slug: str):
    try:
        r = requests.get(f"{GAMMA}/events", params={"slug": slug}, timeout=8)
        r.raise_for_status()
        data = r.json()
        if data and len(data) > 0:
            return data[0]
    except Exception as e:
        print(f"[Scout] خطا در گرفتن بازار: {e}")
    return None


def get_orderbook(token_id: str):
    try:
        r = requests.get(f"{CLOB}/book", params={"token_id": token_id}, timeout=5)
        r.raise_for_status()
        return r.json()
    except:
        return None


def get_btc_price():
    try:
        r = requests.get(BINANCE, timeout=3)
        return float(r.json()["price"])
    except:
        return None


def best_prices(book):
    if not book:
        return None, None
    bids = book.get("bids", [])
    asks = book.get("asks", [])
    best_bid = float(bids[-1]["price"]) if bids else None
    best_ask = float(asks[-1]["price"]) if asks else None
    return best_bid, best_ask


def clear_screen():
    os.system("cls" if os.name == "nt" else "clear")


def main():
    portfolio = PaperPortfolio()
    price_history = deque(maxlen=20)
    current_slug = None
    last_signal = None

    print("🚀 BTC 5-Minute Paper Bot (نسخه تحلیل‌محور) شروع شد...")
    print(f"ریسک به ازای هر معامله: ${RISK_PER_TRADE}")
    print("سطوح کلیدی: حمایت 63300 | مقاومت 64000")
    print("حالت: فقط Paper (بدون سفارش واقعی)\n")
    time.sleep(2)

    while True:
        try:
            window_start = get_current_window_start()
            slug = f"btc-updown-5m-{window_start}"

            if slug != current_slug:
                current_slug = slug
                last_signal = None
                print(f"\n🔄 پنجره جدید: {slug}")

            market = get_market_by_slug(slug)
            btc_price = get_btc_price()

            if btc_price:
                price_history.append(btc_price)

            # محاسبه مومنتوم
            momentum = 0.0
            if len(price_history) >= 5:
                old = price_history[0]
                momentum = (btc_price - old) / old * 10000  # basis point

            up_token = down_token = None
            up_bid = up_ask = down_bid = down_ask = None

            if market:
                try:
                    tokens = json.loads(market.get("clobTokenIds") or "[]")
                    if len(tokens) >= 2:
                        up_token, down_token = tokens[0], tokens[1]
                except:
                    pass

                if up_token:
                    book_up = get_orderbook(up_token)
                    up_bid, up_ask = best_prices(book_up)
                if down_token:
                    book_down = get_orderbook(down_token)
                    down_bid, down_ask = best_prices(book_down)

            # ====================== منطق سیگنال بهبود یافته ======================
            signal = None
            seconds_left = 300 - (int(time.time()) - window_start)

            if btc_price and seconds_left > 50:  # فقط در نیمه اول پنجره سیگنال بده
                # سیگنال Long (Up) - ریباند از حمایت + مومنتوم مثبت
                if (btc_price <= SUPPORT_1 + 150) and momentum > 6 and up_ask and up_ask < 0.58:
                    signal = "Up"

                # سیگنال Short (Down) - رد شدن از مقاومت + مومنتوم منفی
                elif (btc_price >= RESISTANCE_1 - 150) and momentum < -6 and down_ask and down_ask < 0.58:
                    signal = "Down"

                # سیگنال جایگزین: مومنتوم خیلی قوی
                elif momentum > 12 and up_ask and up_ask < 0.55:
                    signal = "Up"
                elif momentum < -12 and down_ask and down_ask < 0.55:
                    signal = "Down"
            # ==================================================================

            # نمایش ترمینال
            clear_screen()
            now = datetime.now(timezone.utc).strftime("%Y-%m-%d %H:%M:%S UTC")

            print("=" * 70)
            print(f"  POLYMARKET BTC 5m PAPER BOT (تحلیل‌محور)  |  {now}")
            print("=" * 70)
            print(f"پنجره فعلی     : {slug}")
            print(f"زمان باقی‌مانده : {seconds_left:3d} ثانیه")
            print(f"قیمت BTC       : {btc_price:,.2f}" if btc_price else "قیمت BTC       : ---")
            print(f"مومنتوم ۳۰ث    : {momentum:+.1f} bp")
            print(f"حمایت اصلی     : {SUPPORT_1}   |   مقاومت اصلی: {RESISTANCE_1}")
            print("-" * 70)
            print(f"Up   → Bid: {up_bid or '-':.3f}   Ask: {up_ask or '-':.3f}")
            print(f"Down → Bid: {down_bid or '-':.3f}   Ask: {down_ask or '-':.3f}")
            print("-" * 70)

            summ = portfolio.summary()
            print(f"موجودی Paper   : ${summ['balance']}")
            print(f"PnL امروز      : ${summ['daily_pnl']:+.2f}")
            print(f"معاملات / برد  : {summ['total_trades']} / {summ['wins']}  (Winrate {summ['winrate']}%)")
            print(f"پوزیشن باز     : {summ['open_positions']}")
            print("-" * 70)

            if signal and signal != last_signal and seconds_left > 50:
                entry_price = up_ask if signal == "Up" else down_ask
                if entry_price and entry_price > 0.05:
                    pos = portfolio.open_position(signal, entry_price, RISK_PER_TRADE, slug)
                    last_signal = signal
                    print(f"✅ سیگنال {signal} اجرا شد @ {entry_price:.3f}  |  ریسک ${RISK_PER_TRADE}")
            else:
                print(f"سیگنال فعلی   : {signal or 'هیچ'}   (آخرین: {last_signal or '-'})")

            print("=" * 70)
            print("Ctrl+C برای خروج | لاگ‌ها در paper_trades.jsonl ذخیره می‌شوند")

            time.sleep(POLL_INTERVAL)

        except KeyboardInterrupt:
            print("\n\nبات متوقف شد.")
            print(portfolio.summary())
            break
        except Exception as e:
            print(f"خطا: {e}")
            time.sleep(5)


if __name__ == "__main__":
    main()
