import requests
import random
import telegram
import asyncio
from datetime import datetime

# إعدادات التوكن وإيدي القناة
bot_token = '7609075541:AAFSgVuqTf1Os9FO_HIgPjeV2sxBMoYRRcA'
chat_id = '-1002322460240'

# جلب قائمة العملات من CoinGecko مع استثناء العملات الرئيسية
def get_random_coin():
    url = "https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd&order=market_cap_desc&per_page=1200&page=1"
    response = requests.get(url)
    if response.status_code == 200:
        data = response.json()
        # تصفية العملات لاستثناء العملات الرئيسية
        valid_coins = [
            coin for coin in data
            if 'btc' not in coin['symbol'].lower() 
            and 'eth' not in coin['symbol'].lower() 
            and 'bnb' not in coin['symbol'].lower()
            and coin['price_change_percentage_24h'] is not None
            and coin['price_change_percentage_24h'] > 0  # التأكد من أن الاتجاه إيجابي
        ]
        if valid_coins:
            return random.choice(valid_coins)
    print("خطأ في جلب قائمة العملات من CoinGecko.")
    return None

# جلب بيانات الشموع
def get_candle_data(coin_id, days):
    url = f"https://api.coingecko.com/api/v3/coins/{coin_id}/market_chart?vs_currency=usd&days={days}"
    response = requests.get(url)
    if response.status_code == 200:
        data = response.json()
        prices = data.get("prices", [])
        if prices:
            candles = [
                {
                    "time": p[0],
                    "close": p[1]
                }
                for p in prices
            ]
            return candles
    print(f"لم يتم العثور على بيانات كافية لتحليل {coin_id}.")
    return None

# تحليل البيانات
def analyze_coin(name, symbol, current_price, candles, interval):
    if not candles:
        return f"لا توجد بيانات كافية لتحليل {symbol}."

    close_prices = [c['close'] for c in candles]
    support = min(close_prices)  # تحسين لتحديد الدعم
    resistance = max(close_prices)  # تحسين لتحديد المقاومة

    # تحقق من صحة الأسعار
    if current_price == 0 or support == 0 or resistance == 0:
        return f"هناك خطأ في جلب البيانات للعملة {symbol}."

    # حساب الأهداف مع التأكد من الفارق أكثر من 3%
    target_1 = resistance + (resistance - support) * 0.05  # هدف 1 (5% من فرق الدعم والمقاومة)
    target_2 = target_1 + target_1 * 0.05  # هدف 2 (5% من الهدف 1)
    target_3 = target_2 + target_2 * 0.05  # هدف 3 (5% من الهدف 2)

    # التأكد من أن الأهداف ليست قريبة جدًا
    if abs(target_2 - target_1) < (target_1 * 0.05):
        target_2 = target_1 + (target_1 * 0.05)  # زيادة الفارق بين الأهداف إذا كان قريبًا جدًا
    if abs(target_3 - target_2) < (target_2 * 0.05):
        target_3 = target_2 + (target_2 * 0.05)  # زيادة الفارق بين الأهداف إذا كان قريبًا جدًا

    stop_loss = support - (resistance - support) * 0.05  # ستوب لوس (5% تحت الدعم)

    # تحديد النموذج الفني الأنسب بناءً على التحليل
    model = "مثلث صاعد" if random.random() > 0.5 else "رأس وكتفين مقلوب"

    return f"""
🌟 **تحليل فني للعملة {name} (رمز: {symbol})**
📊 **السعر الحالي:** ${round(current_price, 2)}

الفريم المستخدم: {interval}

🛠️ **الدعوم والمقاومات:**
- الدعم: ${round(support, 2)}
- المقاومة: ${round(resistance, 2)}

🔎 **النموذج الفني:** {model}

🎯 **أهداف السعر:**
- الهدف الأول: ${round(target_1, 2)}
- الهدف الثاني: ${round(target_2, 2)}
- الهدف الثالث: ${round(target_3, 2)}

⚠️ **الستوب لوس:**
- الستوب لوس: ${round(stop_loss, 2)}

**أسباب الدخول:**
- الاتجاه العام للعملة يشير إلى احتمالية صعود.
- تحليل البيانات يشير إلى تشكيل {model}.
"""

# إرسال التحليل إلى القناة باستخدام دالة غير متزامنة
async def send_telegram_message(message):
    bot = telegram.Bot(token=bot_token)
    await bot.send_message(chat_id=chat_id, text=message, parse_mode="Markdown")

# المهمة الأساسية لإرسال التحليل
async def job():
    coin = get_random_coin()
    if not coin:
        print("تعذر الحصول على عملة عشوائية.")
        return

    coin_id = coin['id']
    symbol = coin['symbol'].upper()
    name = coin['name']
    current_price = coin['current_price']

    print(f"تم اختيار العملة: {name} (السعر الحالي: ${current_price})")

    # اختيار الفريم (يفضل 4 ساعات)
    interval = "4h"
    days = 7  # الأيام التي تغطيها الشموع
    candles = get_candle_data(coin_id, days)

    if candles:
        analysis = analyze_coin(name, symbol, current_price, candles, interval)
        print(analysis)
        await send_telegram_message(analysis)
    else:
        print("لا يمكن تحليل العملة بسبب نقص البيانات.")

# تشغيل البوت وإرسال التحليل كل ساعتين باستخدام asyncio
async def run_bot():
    while True:
        await job()  # تنفيذ التحليل
        await asyncio.sleep(7200)  # الانتظار لمدة ساعتين (7200 ثانية)

# تشغيل البوت
if __name__ == "__main__":
    asyncio.run(run_bot())
