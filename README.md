#!/usr/bin/env python3
"""
DARKHYDRA V4 — Multi-Strategy Optimized (Turbo Mode)
Final with: strategy customization + configurable MTG steps (separate for signals/backtest) + chart screenshots
Run:
    pip install requests colorama termcolor pytz pyfiglet
    python3 darkhydra_v4_professional_debug_with_custom.py
"""
import asyncio
import datetime
import time
import sys
import select
import requests
import html
import pytz
import math
import shutil
from typing import List, Dict, Optional, Tuple
from colorama import Fore, Style, init
from termcolor import colored
import random
import os
from datetime import timedelta
import pyfiglet


expiry_str = "08/02/2026"  # DD/MM/YYYY
expiry_date = datetime.datetime.strptime(expiry_str, "%d/%m/%Y")

# --- Current time ---
now = datetime.datetime.now()

# --- Check expiration ---
if now >= expiry_date:
    print("❌ Script expired! Exiting...")
    sys.exit()  # Script stop hobe ekhane

# --- Script continue if not expired ---
print("✅ Script is valid. Continuing...")

# =============== Init ===============
init(autoreset=True)
TIMEZONE = pytz.timezone('Asia/Dhaka')

# ===== Endpoints =====
OTC_API_URL = "https://vps2.arxonxpro.top/cheat2.php"
HIST_API_URL = "https://vps2.arxonxpro.top/30daydata.php"
CHART_BASE_URL = "https://vps2.arxonxpro.top/chart2.php"

# =============== Config & Defaults ===============
class Config:
    EMA_PERIODS = [10, 20, 50, 100, 200]  # default multi EMA set
    REQUEST_TIMEOUT = 90
    TELEGRAM_TIMEOUT = 90
    DEBUG = True

    # RSI defaults
    RSI_PERIOD = 14
    RSI_OVERBOUGHT = 70
    RSI_OVERSOLD = 30

    # Support/Resistance wick settings (defaults)
    LOOKBACK = 10
    WICK_RATIO = 1.2

    # Backtest
    MAX_RETRIES = 3
    RETRY_DELAY = 1.0


STRAT_PARAMS: Dict[str, Dict] = {
    "EMAPULBACK": {
        "ema_periods": [20, 50, 100, 200],
        "touch_pct_threshold": 0.002,
        "atr_lookback": 10,
        "atr_multiplier": 0.5,
        "min_slope_pct": 0.01,
        "require_full_stack": True,
        "trend_mode": "multi",
        "trend_ema_periods": [20,50,100,200]
    },
    "EMASLOPE": {
        "trend_mode": "multi",
        "trend_ema_periods": [20,50,100,200],
        "require_trend": True,
        "slope_min_pct": 0.01,
        "engulfing_required": True,
        "engulfing_min_body_pct": 0.15
    }
}

# ===== runtime globals =====
TRADE_HISTORY: List[Dict] = []
live_candle_cache: Dict[str, List[Dict]] = {}
hist_candle_cache: Dict[str, List[Dict]] = {}
SEND_CHARTS = False
# Separate martingale settings: one used for actual live signal attempts, one used by the backtest filter
MARTINGALE_STEPS_SIGNAL = 1  # used when processing a live/sent signal
MARTINGALE_STEPS_BACKTEST = 1  # used when backtesting/filtering
STRAT_PARAMS: Dict[str, Dict] = {}  # will store per-strategy customized params

# Safety-margin defaults (can prompt user to enable/change)
USE_SAFETY_MARGIN = False
SAFETY_WICK_THRESHOLD = 0.80   # e.g., 0.80 == 80%
SAFETY_BODY_THRESHOLD = 0.20   # e.g., 0.20 == 20%

# =============== UI / logging helpers (same as before) ===============
RESET = Style.RESET_ALL
BRIGHT_YELLOW = Fore.YELLOW + Style.BRIGHT
BRIGHT_GREEN = Fore.GREEN + Style.BRIGHT
BRIGHT_RED = Fore.RED + Style.BRIGHT
BRIGHT_CYAN = Fore.CYAN + Style.BRIGHT
WHITE = Fore.WHITE

def _timestamp():
    return datetime.datetime.now(TIMEZONE).strftime('%H:%M:%S')

def log_debug(msg: str):
    if not Config.DEBUG: return
    print(f"[{_timestamp()}] {BRIGHT_CYAN}{msg}{RESET}")

def log_info(msg: str):
    print(f"[{_timestamp()}] {BRIGHT_GREEN}{msg}{RESET}")

def log_warn(msg: str):
    print(f"[{_timestamp()}] {BRIGHT_YELLOW}{msg}{RESET}")

def log_err(msg: str):
    print(f"[{_timestamp()}] {BRIGHT_RED}{msg}{RESET}")

def print_box(title: str, lines: List[str], title_color: str = BRIGHT_CYAN):
    if not lines:
        lines = ["(none)"]
    width = max(len(title), *(len(l) for l in lines)) + 4
    border = '━' * width
    print(title_color + f'┏{border}┓' + RESET)
    print(title_color + f'┃  {title.ljust(width-2)}┃' + RESET)
    print(title_color + f'┣{border}┫' + RESET)
    for l in lines:
        print(f'┃  {l.ljust(width-2)}┃')
    print(title_color + f'┗{border}┛' + RESET)

async def _run_with_spinner(coro, message: str):
    stop = asyncio.Event()
    async def _spinner():
        chars = '|/-\\'
        i = 0
        while not stop.is_set():
            sys.stdout.write(f"\r{BRIGHT_YELLOW}{message} {chars[i % len(chars)]}{RESET}")
            sys.stdout.flush()
            i += 1
            await asyncio.sleep(0.12)
        sys.stdout.write('\r' + ' ' * (len(message) + 6) + '\r')
        sys.stdout.flush()
    spinner_task = asyncio.create_task(_spinner())
    try:
        result = await coro
    finally:
        stop.set()
        await spinner_task
    return result
    
def typing_print(text: str, char_delay: float = 0.008, line_delay: float = 0.06):
    for ch in text:
        sys.stdout.write(ch)
        sys.stdout.flush()
        time.sleep(char_delay)
    sys.stdout.write("\n")
    sys.stdout.flush()
    time.sleep(line_delay)


def show_loading_spinner(duration: float = 1.2, message: str = "Loading configuration"):
    spinner = "|/-\\"
    end_time = time.time() + duration
    i = 0
    while time.time() < end_time:
        sys.stdout.write(
            f"\r{Style.BRIGHT}{Fore.YELLOW}{message} {spinner[i % len(spinner)]}{RESET}"
        )
        sys.stdout.flush()
        i += 1
        time.sleep(0.09)
    sys.stdout.write("\r" + " " * (len(message) + 8) + "\r")
    sys.stdout.flush()    

# =============== Helpers & UI small functions ===============
MONO_MAP = str.maketrans(
    "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-:. ",
    "𝙰𝙱𝙲𝙳𝙴𝙵𝙶𝙷𝙸𝙹𝙺𝙻𝙼𝙽𝙾𝙿𝚀𝚁𝚂𝚃𝚄𝚅𝚆𝚇𝚈𝚉"
    "𝚊𝚋𝚌𝚍𝚎𝚏𝚐𝚑𝚒𝚓𝚔𝚕𝚖𝚗𝚘𝚙𝚚𝚛𝚜𝚝𝚞𝚟𝚠𝚡𝚢𝚣"
    "𝟶𝟷𝟸𝟹𝟺𝟻𝟼𝟽𝟾𝟿﹘∶． "
)
def to_mono(text: str) -> str:
    return str(text).translate(MONO_MAP)

# =============== User Inputs (initial) ===============
# === Colors ===
YELLOW_BOLD = Fore.YELLOW + Style.BRIGHT
RESET = Style.RESET_ALL

# Define colors using ANSI escape sequences
RESET = "\033[0m"
BOLD = "\033[1m"
BRIGHT_GREEN = "\033[92m"
BRIGHT_MAGENTA = "\033[95m"
BRIGHT_BLUE = "\033[94m"
BRIGHT_CYAN = "\033[96m"
BRIGHT_YELLOW = "\033[93m"
WHITE = "\033[97m"
BRIGHT_RED = "\033[91m"
BROWN = "\033[38;5;94m"  # ANSI code for brown color

# ANSI color codes
BLUE = "\033[94m"
GREEN = "\033[92m"
YELLOW = "\033[93m"
RED = "\033[91m"
CYAN = "\033[96m"
WHITE = "\033[97m"
RESET = "\033[0m"

# Gold ANSI color using 256-color mode (approximate gold shades)
gold_ansi_256 = "\033[38;5;220m"  # Gold-like color (orange-yellow shade)
reset = "\033[0m"

# Gold ANSI color using 256-color mode (approximate gold shades)
gold_ansi_256 = "\033[38;5;220m"  # Gold-like color (orange-yellow shade)
reset = "\033[0m"

def to_monospace(text):
    result = ''
    for char in text:
        if 'A' <= char <= 'Z':
            result += chr(ord(char) + 0x1D670 - ord('A'))
        elif 'a' <= char <= 'z':
            result += chr(ord(char) + 0x1D68A - ord('a'))
        elif '0' <= char <= '9':
            result += chr(ord(char) + 0x1D7F6 - ord('0'))
        else:
            result += char
    return result

# Clear the screen function
def clear_screen():
    os.system('cls' if os.name == 'nt' else 'clear')

# Animated typing effect
def type_banner(text, delay=0.005):
    clear_screen()
    for char in text:
        print(char, end='', flush=True)
        time.sleep(delay)
    print()  # Move to a new line at the end
    
def typing_effect(text, delay=0.009):
    for char in text:
        sys.stdout.write(char)
        sys.stdout.flush()
        time.sleep(delay)
    print()    
    
def to_monospace(text):
    result = ''
    for char in text:
        if 'A' <= char <= 'Z':
            result += chr(ord(char) + 0x1D670 - ord('A'))
        elif 'a' <= char <= 'z':
            result += chr(ord(char) + 0x1D68A - ord('a'))
        elif '0' <= char <= '9':
            result += chr(ord(char) + 0x1D7F6 - ord('0'))
        else:
            result += char
    return result

def typing_effect(text, delay=0.009):
    for char in text:
        sys.stdout.write(char)
        sys.stdout.flush()
        time.sleep(delay)
    print() 
    
# Function to show animated loading
def animated_loading(message, duration=3):
    chars = ["|", "-", "|", "-"]
    start_time = time.time()
    while time.time() - start_time < duration:
        for char in chars:
            sys.stdout.write(f"\r{BRIGHT_YELLOW}{BOLD}{message} {char}{RESET}")
            sys.stdout.flush()
            time.sleep(0.2)
    sys.stdout.write("\r" + " " * (len(message) + 2) + "\r")  # Clear line    

WELCOME_HEADER = f"""
{Fore.RED + Style.BRIGHT}
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━😈𝗪𝗘𝗟𝗟𝗖𝗢𝗠𝗘 𝗧𝗢 𝗦𝗢𝗙𝗧😈━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
{Fore.GREEN}
  ██╗    ██╗███████╗██╗     ██╗      ██████╗ ██████╗ ███╗   ███╗███████╗    
  ██║    ██║██╔════╝██║     ██║     ██╔════╝██╔═══██╗████╗ ████║██╔════╝    
  ██║ █╗ ██║█████╗  ██║     ██║     ██║     ██║   ██║██╔████╔██║█████╗      
  ██║███╗██║██╔══╝  ██║     ██║     ██║     ██║   ██║██║╚██╔╝██║██╔══╝      
  ╚███╔███╔╝███████╗███████╗███████╗╚██████╗╚██████╔╝██║ ╚═╝ ██║███████╗    
   ╚══╝╚══╝ ╚══════╝╚══════╝╚══════╝ ╚═════╝ ╚═════╝ ╚═╝     ╚═╝╚══════╝    
{Fore.RED + Style.BRIGHT}
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━😈𝗪𝗘𝗟𝗟𝗖𝗢𝗠𝗘 𝗧𝗢 𝗦𝗢𝗙𝗧😈━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
{RESET}
"""

def clear_screen():
    os.system('cls' if os.name == 'nt' else 'clear')


def get_terminal_size():
    try:
        columns, rows = shutil.get_terminal_size()
        return max(columns, 60), max(rows, 20)
    except:
        return 60, 20


def virtual_reality_visualization():
    clear_screen()
    width, height = get_terminal_size()
    is_mobile = width < 80

    ascii_height = 3
    chart_height = min(10, height - ascii_height - 2)
    price_data = []
    current_price = 100.0
    data_points = 80 if is_mobile else 150

    for _ in range(data_points):
        movement = random.normalvariate(0, 1) * 0.5
        if random.random() < 0.55:
            movement += 0.05
        current_price = max(50, current_price + movement)
        price_data.append(current_price)

    candles = []
    candle_period = 5
    for i in range(0, len(price_data), candle_period):
        period_data = price_data[i:i + candle_period]
        if period_data:
            candles.append({
                'open': period_data[0],
                'high': max(period_data),
                'low': min(period_data),
                'close': period_data[-1],
                'bullish': period_data[-1] >= period_data[0]
            })

    max_candles = min(width // 3 - 5, 15)
    start_idx = 0

    for frame in range(30):
        clear_screen()
        print(WELCOME_HEADER)
        buffer_height = min(height, ascii_height + chart_height + 2)
        buffer = [[' ' for _ in range(width)] for _ in range(buffer_height)]

        visible_candles = candles[start_idx:start_idx + max_candles]
        if visible_candles:
            min_price = min(c['low'] for c in visible_candles) * 0.99
            max_price = max(c['high'] for c in visible_candles) * 1.01
            price_range = max_price - min_price
            for y in range(ascii_height + 1, buffer_height):
                if (y - (ascii_height + 1)) % 3 == 0:
                    price_str = f"{max_price - (y - (ascii_height + 1)) / chart_height * price_range:.1f}"
                    for x, char in enumerate(price_str[:5]):
                        buffer[y][x] = Fore.YELLOW + char + RESET
                    for x in range(7, width):
                        buffer[y][x] = Fore.BLUE + '·' + RESET

            x_start = 10
            candle_spacing = 3
            for i, candle in enumerate(visible_candles):
                x = x_start + i * candle_spacing
                y_positions = {
                    'open': candle['open'],
                    'close': candle['close'],
                    'high': candle['high'],
                    'low': candle['low']
                }
                for key in y_positions:
                    y_positions[key] = int(ascii_height + 1 + chart_height - (
                            (y_positions[key] - min_price) / price_range) * chart_height)
                    y_positions[key] = max(ascii_height + 1, min(buffer_height - 1, y_positions[key]))

                for y in range(y_positions['high'], y_positions['low'] + 1):
                    if ascii_height + 1 <= y < buffer_height and 0 <= x < width:
                        buffer[y][x] = Fore.WHITE + '│' + RESET

                for y in range(min(y_positions['open'], y_positions['close']),
                               max(y_positions['open'], y_positions['close']) + 1):
                    color = Fore.GREEN if candle['bullish'] else Fore.RED
                    if ascii_height + 1 <= y < buffer_height and 0 <= x < width:
                        buffer[y][x] = color + '█' + RESET

        for row in buffer:
            print(''.join(row))
        if frame % 2 == 0 and start_idx < len(candles) - max_candles:
            start_idx += 1
        time.sleep(0.05)


def main():
    virtual_reality_visualization()


if __name__ == "__main__":
    main()

import sys
import time
import os

# ======================
# YOUR MAIN PASSWORD HERE
# ======================
MAIN_PASSWORD = "9905"   # ← এখানে আপনার পাসওয়ার্ড দিন


def check_password():
    print("┏━━━━━━━━━━━━━━━━━━━━━━ 🔐 AUTHENTICATION 🔐 ━━━━━━━━━━━━━━━━━━━━━━┓")
    print("        PASSWORD REQUIRED TO ACCESS THE SOFTWARE")
    print("┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛\n")

    pw = input("🔑 Enter Password: ").strip()

    if pw == MAIN_PASSWORD:
        print("\n✅ Password Correct! Access Granted.")
        time.sleep(1)
        clear_screen()
        return True
    else:
        print("\n❌ Wrong Password! Access Denied.")
        time.sleep(2)
        sys.exit(1)


def main_program():
    print("🔰 Welcome to TOWSIF SOFTWARE!")
    print("📊 Main Software Running Now...")
    # ---- এখানে আপনার মেইন কোড রান হবে ----
    time.sleep(1)


if __name__ == "__main__":
    if check_password():
        main_program()
        
        
# =============== Small UI helpers ===============
def rainbow(text: str) -> str:
    cols = [Fore.RED, Fore.YELLOW, Fore.GREEN, Fore.CYAN, Fore.BLUE, Fore.MAGENTA]
    return ''.join(cols[i % len(cols)] + ch for i, ch in enumerate(text)) + Style.RESET_ALL

def white_bold(text: str) -> str:
    return Style.BRIGHT + Fore.WHITE + text + Style.RESET_ALL

def cyan_bold(text: str) -> str:
    return Style.BRIGHT + Fore.CYAN + text + Style.RESET_ALL

def big_banner(text: str):
    ascii_art = pyfiglet.figlet_format(text)
    print(white_bold(ascii_art))

def banner(text: str):
    print(cyan_bold(text))

# =============== HTTP helpers ===============
HEADERS = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"}
REQ_TIMEOUT = 15

def http_get_json(url, params=None, timeout=REQ_TIMEOUT):
    for attempt in range(1, Config.MAX_RETRIES + 1):
        try:
            r = requests.get(url, params=params, headers=HEADERS, timeout=timeout)
            r.raise_for_status()
            return r.json()
        except Exception:
            if attempt == Config.MAX_RETRIES:
                raise
            time.sleep(Config.RETRY_DELAY)

# =============== Live candles fetch ===============
def fetch_otc_candles(instrument: str, count: int = 220) -> List[Dict]:
    try:
        params = {"pair": instrument, "count": count}
        r = requests.get(OTC_API_URL, params=params, timeout=Config.REQUEST_TIMEOUT)
        data = r.json()
        if isinstance(data, dict) and "data" in data and isinstance(data["data"], list):
            candles = []
            for item in data["data"]:
                dt_str = item.get("time", "")
                if len(dt_str) == 16:
                    dt_str += ":00"
                candle = {
                    "time": dt_str,
                    "mid": {
                        "o": str(item.get("open", "0")),
                        "h": str(item.get("high", "0")),
                        "l": str(item.get("low", "0")),
                        "c": str(item.get("close", "0"))
                    }
                }
                candles.append(candle)
            return candles
        if isinstance(data, dict) and "data" in data:
            container = data["data"]
            if isinstance(container, list):
                return container
        return []
    except Exception:
        return []

# =============== Candle helpers & indicators ===============
def candle_direction(candle) -> str:
    try:
        o = float(candle["mid"]["o"]) ; c = float(candle["mid"]["c"])
        if c > o: return "CALL"
        if c < o: return "PUT"
    except Exception:
        pass
    return "FLAT"

def calculate_ema_series_from_mid(candles: List[Dict], period: int) -> Optional[List[Optional[float]]]:
    closes: List[float] = []
    for c in candles:
        try:
            closes.append(float(c["mid"]["c"]))
        except Exception:
            return None
    n = len(closes)
    if n < period:
        return None
    emas: List[Optional[float]] = [None] * n
    seed = sum(closes[0:period]) / period
    emas[period - 1] = seed
    multiplier = 2 / (period + 1)
    for i in range(period, n):
        prev = emas[i - 1]
        if prev is None:
            prev = seed
        ema = (closes[i] - prev) * multiplier + prev
        emas[i] = ema
    return emas

def calculate_rsi_from_mid(candles: List[Dict], period: int) -> Optional[float]:
    closes: List[float] = []
    for c in candles:
        try:
            closes.append(float(c["mid"]["c"]))
        except Exception:
            return None
    if len(closes) < period + 1:
        return None
    gains: List[float] = []
    losses: List[float] = []
    for i in range(1, period + 1):
        change = closes[i] - closes[i-1]
        gains.append(max(0.0, change))
        losses.append(max(0.0, -change))
    avg_gain = sum(gains) / period
    avg_loss = sum(losses) / period
    for i in range(period + 1, len(closes)):
        change = closes[i] - closes[i-1]
        gain = max(0.0, change)
        loss = max(0.0, -change)
        avg_gain = (avg_gain * (period - 1) + gain) / period
        avg_loss = (avg_loss * (period - 1) + loss) / period
    if avg_loss == 0:
        return 100.0
    rs = avg_gain / avg_loss
    return 100.0 - (100.0 / (1 + rs))

# =============== Historical fetch & parsing ===============
def normalize_pair_for_api(pair: str) -> str:
    if not pair: return ""
    p = ''.join(ch for ch in str(pair) if ch.isalnum()).upper()
    if p.endswith("OTC"): p = p[:-3]
    if not p: return ""
    return f"{p}_otc"

def fetch_candles_for_pair(pair, days, debug=False):
    params = {"pair": pair, "days": days}
    try:
        data = http_get_json(HIST_API_URL, params=params)
    except Exception:
        return []
    candles = []
    if isinstance(data, dict) and "data" in data:
        container = data["data"]
        if isinstance(container, dict):
            def norm(x): return ''.join(ch for ch in str(x) if ch.isalnum()).upper()
            target = norm(pair)
            found = False
            for k, v in container.items():
                if isinstance(v, list) and norm(k) == target:
                    candles = v; found = True; break
            if not found:
                for k, v in container.items():
                    if isinstance(v, list) and v:
                        if isinstance(v[0], dict) and ("time" in v[0] or "datetime" in v[0]):
                            candles = v; break
        elif isinstance(container, list):
            candles = container
    if not candles and isinstance(data, dict):
        for k_candidate in (pair, pair.upper(), pair.lower()):
            if k_candidate in data and isinstance(data[k_candidate], list):
                candles = data[k_candidate]; break
    if not candles and isinstance(data, list):
        candles = data
    if not candles and isinstance(data, dict) and "assets" in data and "data" in data and isinstance(data["data"], dict):
        container = data["data"]
        for k, v in container.items():
            if isinstance(v, list):
                if ''.join(ch for ch in str(k) if ch.isalnum()).upper() == ''.join(ch for ch in str(pair) if ch.isalnum()).upper():
                    candles = v; break
    normalized = []
    for c in (candles or []):
        if not isinstance(c, dict): continue
        t = c.get("time") or c.get("datetime") or c.get("timestamp") or c.get("Time")
        if not t:
            if "date" in c and "time" in c:
                t = f"{c.get('date')} {c.get('time')}"
            else:
                continue
        nc = dict(c)
        nc["time"] = t
        if "direction" in nc:
            try: nc["direction"] = str(nc["direction"]).upper()
            except: pass
        normalized.append(nc)
    def parse_time_val(t):
        for fmt in ("%Y-%m-%d %H:%M:%S", "%Y-%m-%d %H:%M"):
            try: return datetime.datetime.strptime(t, fmt)
            except: continue
        try:
            ts = int(float(t)); return datetime.datetime.fromtimestamp(ts)
        except:
            return None
    parsed = []
    for c in normalized:
        dt = parse_time_val(str(c["time"]))
        if dt:
            c["_parsed_time"] = dt
            parsed.append(c)
    parsed.sort(key=lambda x: x["_parsed_time"])
    return parsed

def build_candle_map(candles):
    cmap = {}
    for c in candles:
        dt = c.get("_parsed_time")
        if not dt: continue
        key = dt.strftime("%Y-%m-%d %H:%M")
        cmap[key] = c
    return cmap
    
    
    

# =============== Backtest helpers (unchanged logic but used with MARTINGALE_STEPS_BACKTEST) ===============
def generate_positive_offsets(max_step):
    yield 0
    for s in range(1, max_step + 1):
        yield s

def find_candle_for_base_positive_in_cmap(cmap, base_dt, max_offset):
    for off in generate_positive_offsets(max_offset):
        key = (base_dt + timedelta(minutes=off)).strftime("%Y-%m-%d %H:%M")
        if key in cmap:
            return off, cmap[key]
    return None, None

def backtest_with_cmap(cmap, trade_time_hm: str, direction: str, backtest_days: int, martingale_steps: int, skip_recent: bool, debug: bool=False) -> Tuple[bool, List[Dict]]:
    start_offset = 2 if skip_recent else 1
    day_results = []
    all_days_win = True
    for i in range(start_offset, start_offset + backtest_days):
        date_str = (datetime.datetime.now(TIMEZONE) - timedelta(days=i)).strftime("%Y-%m-%d")
        base_str = f"{date_str} {trade_time_hm}"
        try:
            base_dt = datetime.datetime.strptime(base_str, "%Y-%m-%d %H:%M")
        except Exception:
            day_results.append({"date": date_str, "status": "ERROR", "offset": None})
            all_days_win = False
            continue
        off, candle = find_candle_for_base_positive_in_cmap(cmap, base_dt, martingale_steps)
        if candle is None:
            day_results.append({"date": date_str, "status": "LOSS", "offset": None})
            all_days_win = False
            continue
        if is_win(candle, direction):
            status = "WIN (exact)" if off == 0 else f"WIN (mtg +{off})"
            day_results.append({"date": date_str, "status": status, "offset": off})
            continue
        found_win = False
        for extra_off in generate_positive_offsets(martingale_steps):
            if extra_off == off:
                continue
            key = (base_dt + timedelta(minutes=extra_off)).strftime("%Y-%m-%d %H:%M")
            if key in cmap:
                c2 = cmap[key]
                if is_win(c2, direction):
                    status = "WIN (exact)" if extra_off == 0 else f"WIN (mtg +{extra_off})"
                    day_results.append({"date": date_str, "status": status, "offset": extra_off})
                    found_win = True
                    break
        if not found_win:
            day_results.append({"date": date_str, "status": "LOSS", "offset": None})
            all_days_win = False
    return all_days_win, day_results

def is_win(candle, signal_direction):
    try:
        cand_dir = None
        for k in ("direction", "Direction", "dir"):
            if k in candle and candle[k] is not None:
                cand_dir = str(candle[k]).upper(); break
        if cand_dir:
            return cand_dir == signal_direction
        o = None; c = None
        for k in ("open","o","Open","O","OpenPrice"):
            if k in candle:
                try: o = float(candle[k]); break
                except: pass
        for k in ("close","c","Close","C","ClosePrice"):
            if k in candle:
                try: c = float(candle[k]); break
                except: pass
        if o is None or c is None: return False
        return (c > o) if signal_direction == "CALL" else (c < o)
    except:
        return False


def safety_margin_check(
    candle: Dict,
    signal_direction: str,
    wick_thresh: float = SAFETY_WICK_THRESHOLD,   # e.g. 0.80 (80%)
    body_thresh: float = SAFETY_BODY_THRESHOLD,   # e.g. 0.20 (20%)
    tiny_body_thresh: float = 0.05                # <=5% body = auto safety
) -> bool:
    """
    Strong & flexible Safety Margin Check:
      ✔ Primary safety -> body <= body_thresh AND rejection-wick >= wick_thresh
      ✔ Tiny-body/Doji override -> body_ratio <= tiny_body_thresh => SAFETY
      ✔ Extreme-wick override -> wick almost full candle (>=95%) => SAFETY
      ✔ Works for both CALL/PUT
    """
    try:
        # Extract OHLC flexibly
        o = h = l = c = None
        for k in ("open","o","Open","O","OpenPrice"):
            if k in candle:
                try: o = float(candle[k]); break
                except: pass
        for k in ("high","h","High","H"):
            if k in candle:
                try: h = float(candle[k]); break
                except: pass
        for k in ("low","l","Low","L"):
            if k in candle:
                try: l = float(candle[k]); break
                except: pass
        for k in ("close","c","Close","C","ClosePrice"):
            if k in candle:
                try: c = float(candle[k]); break
                except: pass

        if None in (o, h, l, c):
            return False

        # Candle metrics
        total = h - l
        if total <= 0:
            return False

        body = abs(c - o)
        upper_wick = h - max(c, o)
        lower_wick = min(c, o) - l

        # Normalize
        body_ratio = body / total
        upper_ratio = upper_wick / total
        lower_ratio = lower_wick / total

        # Clamp thresholds safe-range
        tiny_body_thresh = max(0.0, min(0.2, float(tiny_body_thresh)))
        body_thresh = max(0.0, min(1.0, float(body_thresh)))
        wick_thresh = max(0.0, min(1.0, float(wick_thresh)))

        # -------------------------
        # 1) Tiny body → DOJI/FLAT → auto Safety
        # -------------------------
        if body_ratio <= tiny_body_thresh:
            return True

        # -------------------------
        # 2) Main strict rule
        # -------------------------
        if signal_direction == "CALL":
            if lower_ratio >= wick_thresh and body_ratio <= body_thresh:
                return True
        else:  # PUT
            if upper_ratio >= wick_thresh and body_ratio <= body_thresh:
                return True

        # -------------------------
        # 3) Extreme-wick override (almost full rejection wick)
        # -------------------------
        extreme_limit = 1.0 - tiny_body_thresh   # e.g., 1.0 - 0.05 = 0.95

        if signal_direction == "CALL":
            if lower_ratio >= extreme_limit and body_ratio <= (tiny_body_thresh * 2):
                return True
        else:
            if upper_ratio >= extreme_limit and body_ratio <= (tiny_body_thresh * 2):
                return True

        return False

    except Exception:
        return False
        
        
# =============== Strategies (adapted to read STRAT_PARAMS where applicable) ===============
def get_param(strategy: str, key: str, default):
    return STRAT_PARAMS.get(strategy, {}).get(key, default)

def calculate_ema_from_values(values: List[float], period: int) -> List[Optional[float]]:
    """Simple EMA generator for a plain list of numeric values.
    Returns a list of same length where first (period-1) entries are None and later entries are EMA values.
    """
    n = len(values)
    if n < period or period <= 0:
        return [None] * n
    emas: List[Optional[float]] = [None] * n
    seed = sum(values[0:period]) / period
    emas[period - 1] = seed
    multiplier = 2 / (period + 1)
    for i in range(period, n):
        prev = emas[i - 1] if emas[i - 1] is not None else seed
        emas[i] = (values[i] - prev) * multiplier + prev
    return emas


def calculate_macd_series_from_mid(candles: List[Dict], fast: int = 12, slow: int = 26, signal: int = 9) -> Optional[Dict]:
    """Compute MACD line, signal line and histogram for the provided candles.
    Expects the usual candle format used elsewhere (c['mid']['c']).
    Returns dict with keys: 'macd', 'signal', 'hist', 'closes'.
    """
    try:
        closes = [float(c["mid"]["c"]) for c in candles]
    except Exception:
        return None
    if len(closes) < max(slow, fast, signal) + 1:
        # still compute where possible, but require at least slow points for stable macd
        pass
    ema_fast = calculate_ema_from_values(closes, fast)
    ema_slow = calculate_ema_from_values(closes, slow)

    macd = [None] * len(closes)
    for i, (f, s) in enumerate(zip(ema_fast, ema_slow)):
        if f is None or s is None:
            macd[i] = None
        else:
            macd[i] = f - s

    # For signal EMA we need a numeric series; replace None by 0 temporarily so EMA routine can run
    macd_for_signal = [v if v is not None else 0.0 for v in macd]
    signal_line = calculate_ema_from_values(macd_for_signal, signal)

    hist = [None] * len(closes)
    for i, (m, sig) in enumerate(zip(macd, signal_line)):
        if m is None or sig is None:
            hist[i] = None
        else:
            hist[i] = m - sig

    return {"macd": macd, "signal": signal_line, "hist": hist, "closes": closes}


def _split_window_indices(length: int, lookback: int):
    """Return left/right window indices for splitting a lookback into two halves.
    Ensures at least 2 indices in each half when possible.
    """
    if lookback < 4:
        left = lookback // 2
    else:
        left = max(2, lookback // 2)
    right = lookback - left
    return left, right


def detect_macd_divergence(closes: List[float], macd_vals: List[Optional[float]], base_index: int, lookback: int = 8):
    """Detect bullish/bearish divergence ending at base_index (inclusive).

    Returns (direction, score) where direction is 'CALL'|'PUT'|None and score is float confidence metric.
    """
    n = len(closes)
    if base_index < 1 or base_index >= n:
        return None, 0.0

    # limit lookback to available data
    lookback = max(2, min(lookback, base_index + 1))
    start = max(0, base_index - lookback + 1)

    window_closes = closes[start: base_index + 1]
    window_macd = macd_vals[start: base_index + 1]

    # require at least 4 points (2 in each half) for a sensible split
    if len(window_closes) < 4:
        return None, 0.0

    # split into two halves (left / right) using helper
    left_len, right_len = _split_window_indices(len(window_closes), lookback)
    # safety: ensure both halves non-empty
    if left_len < 1 or right_len < 1:
        return None, 0.0

    left_vals = window_closes[:left_len]
    right_vals = window_closes[left_len:]
    left_macd = window_macd[:left_len]
    right_macd = window_macd[left_len:]

    # extra safety: lengths must match
    if not left_vals or not right_vals:
        return None, 0.0

    try:
        # find extremum indices within each half (choose first occurrence if ties)
        left_min_idx = min(range(len(left_vals)), key=lambda i: left_vals[i])
        right_min_idx = min(range(len(right_vals)), key=lambda i: right_vals[i])
        left_max_idx = max(range(len(left_vals)), key=lambda i: left_vals[i])
        right_max_idx = max(range(len(right_vals)), key=lambda i: right_vals[i])
    except Exception:
        return None, 0.0

    # convert to absolute indices in the original closes/macd arrays
    lmin_abs = start + left_min_idx
    rmin_abs = start + left_len + right_min_idx
    lmax_abs = start + left_max_idx
    rmax_abs = start + left_len + right_max_idx

    # ensure these indices are in range
    for idx in (lmin_abs, rmin_abs, lmax_abs, rmax_abs):
        if idx < 0 or idx >= n:
            return None, 0.0

    # fetch prices & macd values (may be None)
    lmin_p = closes[lmin_abs]; rmin_p = closes[rmin_abs]
    lmin_m = macd_vals[lmin_abs]; rmin_m = macd_vals[rmin_abs]
    lmax_p = closes[lmax_abs]; rmax_p = closes[rmax_abs]
    lmax_m = macd_vals[lmax_abs]; rmax_m = macd_vals[rmax_abs]

    bullish = False
    bearish = False
    bullish_score = 0.0
    bearish_score = 0.0

    # bullish divergence: price lower-low + MACD higher-low
    if lmin_m is not None and rmin_m is not None:
        if lmin_p > rmin_p and lmin_m < rmin_m:
            price_diff_pct = (lmin_p - rmin_p) / max(abs(lmin_p), 1e-8) * 100.0
            macd_diff = (rmin_m - lmin_m)
            bullish_score = max(0.0, min(99.9, 20.0 + min(79.9, abs(price_diff_pct) * 2.0 + macd_diff * 15.0)))
            bullish = True

    # bearish divergence: price higher-high + MACD lower-high
    if lmax_m is not None and rmax_m is not None:
        if lmax_p < rmax_p and lmax_m > rmax_m:
            price_diff_pct = (rmax_p - lmax_p) / max(abs(lmax_p), 1e-8) * 100.0
            macd_diff = (lmax_m - rmax_m)
            bearish_score = max(0.0, min(99.9, 20.0 + min(79.9, abs(price_diff_pct) * 2.0 + macd_diff * 15.0)))
            bearish = True

    if bullish and bullish_score >= bearish_score:
        return "CALL", round(bullish_score, 1)
    if bearish and bearish_score > bullish_score:
        return "PUT", round(bearish_score, 1)
    return None, 0.0


def strategy_macd_divergence_from_candles(asset: str, candles: List[Dict]) -> Optional[Dict]:
    """MACD Divergence strategy compatible with the rest of DARKHYDRA.
    - Uses the second-last candle (candles[-2]) as the base evaluation candle (same convention as other strategies).
    - Returns None if no divergence detected or insufficient data.

    Configurable params via STRAT_PARAMS['MACD']:
      - fast (int, default 12)
      - slow (int, default 26)
      - signal (int, default 9)
      - lookback (int, default 8)  # number of candles to inspect ending at base candle

    Returns a dict matching other strategy outputs: {"strategy":"MACD", "asset":asset, "direction":..., "confidence":..., "candle": base_candle}
    """
    try:
        params = {
            "fast": int(get_param("MACD", "fast", 12)),
            "slow": int(get_param("MACD", "slow", 26)),
            "signal": int(get_param("MACD", "signal", 9)),
            "lookback": int(get_param("MACD", "lookback", 8)),
        }
    except Exception:
        params = {"fast":12, "slow":26, "signal":9, "lookback":8}

    macd_res = calculate_macd_series_from_mid(candles, fast=params["fast"], slow=params["slow"], signal=params["signal"])
    if not macd_res:
        return None

    closes = macd_res["closes"]
    macd_vals = macd_res["macd"]

    # pick base index = len(candles)-2 consistent with other strategies
    base_index = len(candles) - 2
    if base_index < 3:
        return None

    direction, confidence = detect_macd_divergence(closes, macd_vals, base_index, lookback=params["lookback"])
    if not direction:
        return None

    base_candle = candles[-2]
    return {"strategy":"MACD","asset":asset,"direction":direction,"confidence":confidence,"candle":base_candle}

def strategy_emapulback_from_candles(asset: str, candles: List[Dict]) -> Optional[Dict]:
    """
    EMAPULBACK — EMA Pullback strategy (final, robust)

    Logic summary:
      1) Determine major trend using EMA 20,50,100,200 (stacking).
         - up_stack: 20 > 50 > 100 > 200
         - down_stack: 20 < 50 < 100 < 200
         (we allow a tiny epsilon tolerance)
      2) Compute EMA5 and EMA5 previous -> slope_pct (percent).
      3) Compute ATR-like volatility (avg true range) for normalization.
      4) Consider EMA5 'touched' if:
           - EMA5 is inside base candle (low <= EMA5 <= high) OR
           - distance(close, EMA5) <= max(pct_threshold * close, atr * atr_multiplier)
      5) Require EMA5 slope direction to match trend (up -> positive slope, down -> negative slope).
      6) Produce CALL on uptrend pullback / PUT on downtrend pullback with a combined confidence score.

    Config via STRAT_PARAMS['EMAPULBACK']:
      - ema_periods: [20,50,100,200] (list)
      - touch_pct_threshold: 0.002  (relative price fraction, e.g. 0.002 = 0.2%)
      - atr_lookback: 10
      - atr_multiplier: 0.5   (distance threshold = max(pct_threshold*price, atr * atr_multiplier))
      - min_slope_pct: 0.01   (minimum EMA5 slope percent required, e.g. 0.01 = 0.01%)
      - require_full_stack: True | False
    """
    try:
        params = STRAT_PARAMS.get("EMAPULBACK", {})
        major_periods = params.get("ema_periods", [20, 50, 100, 200])
        touch_pct_threshold = float(params.get("touch_pct_threshold", 0.002))
        atr_lookback = int(params.get("atr_lookback", 10))
        atr_multiplier = float(params.get("atr_multiplier", 0.5))
        min_slope_pct = float(params.get("min_slope_pct", 0.01))
        require_full_stack = bool(params.get("require_full_stack", True))
    except Exception:
        major_periods = [20, 50, 100, 200]
        touch_pct_threshold = 0.002
        atr_lookback = 10
        atr_multiplier = 0.5
        min_slope_pct = 0.01
        require_full_stack = True

    # periods we need (5 + majors)
    all_periods = [5] + [int(p) for p in major_periods]
    maxp = max(all_periods)

    # need enough candles (use max period + 2 for prev/current)
    if not candles or len(candles) < maxp + 2:
        return None

    try:
        # compute EMAs using calculate_ema_series_from_mid (shared helper)
        ema_map = {}
        for p in all_periods:
            series = calculate_ema_series_from_mid(candles, int(p))
            if series is None:
                return None
            ema_map[int(p)] = series

        last_idx = len(candles) - 2   # base candle index (consistent with other strategies)
        prev_idx = last_idx - 1
        if prev_idx < 0:
            return None

        # EMA5 values and slope
        ema5 = ema_map[5][last_idx]
        ema5_prev = ema_map[5][prev_idx]
        if ema5 is None or ema5_prev is None:
            return None
        slope_pct = (ema5 - ema5_prev) / max(abs(ema5_prev), 1e-12) * 100.0

        # major EMA values at base index
        major_vals = {p: ema_map[p][last_idx] for p in major_periods}
        if any(v is None for v in major_vals.values()):
            return None

        # stack detection (strict)
        eps = 1e-12
        up_stack = all(major_vals[major_periods[i]] > major_vals[major_periods[i+1]] + eps for i in range(len(major_periods)-1))
        down_stack = all(major_vals[major_periods[i]] < major_vals[major_periods[i+1]] - eps for i in range(len(major_periods)-1))

        if not (up_stack or down_stack):
            if require_full_stack:
                return None
            else:
                # tolerant stacking: count pairwise agreements
                up_pairs = sum(1 for i in range(len(major_periods)-1) if major_vals[major_periods[i]] > major_vals[major_periods[i+1]])
                down_pairs = sum(1 for i in range(len(major_periods)-1) if major_vals[major_periods[i]] < major_vals[major_periods[i+1]])
                up_stack = (up_pairs >= down_pairs and up_pairs >= 2)
                down_stack = (down_pairs > up_pairs and down_pairs >= 2)
                if not (up_stack or down_stack):
                    return None

        # base candle OHLC for touch test
        base = candles[last_idx]
        try:
            o = float(base["mid"]["o"]); c = float(base["mid"]["c"]); h = float(base["mid"]["h"]); l = float(base["mid"]["l"])
        except Exception:
            return None

        # compute ATR-like average true range for normalization
        tr_vals = []
        start_tr_index = max(1, last_idx - atr_lookback + 1)
        for i in range(start_tr_index, last_idx + 1):
            try:
                high_i = float(candles[i]["mid"]["h"])
                low_i = float(candles[i]["mid"]["l"])
                prev_close = float(candles[i-1]["mid"]["c"])
                tr = max(high_i - low_i, abs(high_i - prev_close), abs(low_i - prev_close))
                tr_vals.append(tr)
            except Exception:
                continue
        atr = (sum(tr_vals) / len(tr_vals)) if tr_vals else max( (h - l), abs(c - o), 1e-9 )

        # touch test: either EMA5 inside base candle OR very near by pct or ATR threshold
        pct_thr = touch_pct_threshold * max(abs(c), 1.0)   # relative to price
        atr_thr = max(1e-12, atr * atr_multiplier)
        distance = abs(c - ema5)
        touched = (l <= ema5 <= h) or (distance <= max(pct_thr, atr_thr))

        if not touched:
            return None

        # require slope direction consistent with stack
        # small hysteresis: for up require slope > min_slope_pct (positive); for down require slope < -min_slope_pct
        if up_stack and slope_pct <= min_slope_pct:
            return None
        if down_stack and slope_pct >= -min_slope_pct:
            return None

        # Determine direction
        direction = "CALL" if up_stack else "PUT"

        # Build confidence score (0-99.9)
        # components: stack_strength (0-40), slope_strength (0-30), proximity (0-30)
        # stack_strength: fraction of correct pairwise orders * 40
        pair_count = len(major_periods) - 1
        correct_pairs = 0
        for i in range(pair_count):
            a = major_vals[major_periods[i]]; b = major_vals[major_periods[i+1]]
            if up_stack and a > b: correct_pairs += 1
            if down_stack and a < b: correct_pairs += 1
        stack_score = (correct_pairs / max(1, pair_count)) * 40.0

        # slope strength: proportional to abs(slope_pct), scaled but capped
        slope_score = min(30.0, abs(slope_pct) * 2.5)  # e.g., 0.01% -> small, larger slopes increase score

        # proximity: how close close->EMA5 relative to ATR (closer => better)
        prox_raw = 1.0 - min(1.0, distance / (atr + 1e-12))  # 1.0 if distance=0, 0 if distance>=atr
        prox_score = prox_raw * 30.0

        confidence = round(min(99.9, 20.0 + stack_score + slope_score + prox_score), 1)

        base_candle = base
        return {
            "strategy": "EMAPULBACK",
            "asset": asset,
            "direction": direction,
            "confidence": confidence,
            "candle": base_candle
        }

    except Exception:
        return None

def strategy_ema_from_candles(asset: str, candles: List[Dict]) -> Optional[Dict]:
    # get periods either from custom params or Config default
    periods = get_param("EMA", "periods", Config.EMA_PERIODS)
    # allow single-value list
    if not periods or len(candles) < min(periods)+2:
        return None
    # If single period provided -> simpler logic: trend vs ema and ema slope
    try:
        if len(periods) == 1:
            p = int(periods[0])
            series = calculate_ema_series_from_mid(candles, p)
            if series is None: return None
            last_index = len(candles) - 2
            prev_index = last_index - 1 if last_index - 1 >= 0 else None
            last_ema = series[last_index] if len(series) > last_index else None
            prev_ema = series[prev_index] if prev_index is not None and len(series) > prev_index else None
            if last_ema is None: return None
            last_close = float(candles[last_index]["mid"]["c"])
            direction = None
            if last_close > last_ema and (prev_ema is None or last_ema > prev_ema):
                direction = "CALL"
            elif last_close < last_ema and (prev_ema is None or last_ema < prev_ema):
                direction = "PUT"
            if direction is None: return None
            separation = abs(last_close - last_ema) / max(abs(last_ema), 1e-8) * 100.0
            confidence = round(min(99.9, 50.0 + min(49.9, separation)), 1)
            base_candle = candles[-2]
            return {"strategy":"EMA","asset":asset,"direction":direction,"confidence":confidence,"candle":base_candle}
        # multiple periods logic (expect ordering)
        periods = sorted([int(x) for x in periods])
        maxp = max(periods)
        if len(candles) < maxp + 2:
            return None
        ema_series = {}
        for p in periods:
            series = calculate_ema_series_from_mid(candles, p)
            if series is None:
                return None
            ema_series[p] = series
        last_index = len(candles) - 2
        prev_index = last_index - 1 if last_index - 1 >= 0 else None
        last_emas = {}
        prev_emas = {}
        for p in periods:
            val = ema_series[p][last_index] if len(ema_series[p]) > last_index else None
            if val is None: return None
            last_emas[p] = val
            prev_emas[p] = ema_series[p][prev_index] if prev_index is not None and len(ema_series[p]) > prev_index else None
        last_close = float(candles[last_index]["mid"]["c"])
        pairs = list(zip(periods[:-1], periods[1:]))
        up_pairs = sum(1 for short,long in pairs if last_emas[short] > last_emas[long])
        down_pairs = sum(1 for short,long in pairs if last_emas[short] < last_emas[long])
        required_pairs = len(pairs)
        direction = None
        if up_pairs == required_pairs and last_close > last_emas[periods[0]]:
            prev_fast = prev_emas.get(periods[0])
            if prev_fast is None or last_emas[periods[0]] > prev_fast:
                direction = "CALL"
        elif down_pairs == required_pairs and last_close < last_emas[periods[0]]:
            prev_fast = prev_emas.get(periods[0])
            if prev_fast is None or last_emas[periods[0]] < prev_fast:
                direction = "PUT"
        if direction is None: return None
        ema_fast = last_emas[periods[0]]
        ema_slow = last_emas[periods[-1]]
        separation = abs(ema_fast - ema_slow) / max(abs(ema_slow), 1e-8) * 100.0
        alignment_ratio = max(up_pairs, down_pairs) / (len(periods) - 1) * 100.0
        confidence = alignment_ratio * 0.6 + min(40.0, separation * 0.8)
        confidence = round(min(99.9, confidence), 1)
        base_candle = candles[-2]
        return {"strategy":"EMA","asset":asset,"direction":direction,"confidence":confidence,"candle":base_candle}
    except Exception:
        return None

def strategy_rsi_from_candles(asset: str, candles: List[Dict]) -> Optional[Dict]:
    period = int(get_param("RSI", "period", Config.RSI_PERIOD))
    overbought = int(get_param("RSI", "overbought", Config.RSI_OVERBOUGHT))
    oversold = int(get_param("RSI", "oversold", Config.RSI_OVERSOLD))
    if not candles or len(candles) < period + 2: return None
    rsi = calculate_rsi_from_mid(candles, period)
    if rsi is None: return None
    if rsi <= oversold:
        direction = "CALL"
    elif rsi >= overbought:
        direction = "PUT"
    else:
        return None
    confidence = round(min(99.9, abs(rsi - 50.0) * 2.0), 1)
    base_candle = candles[-2] if len(candles) >= 2 else candles[-1]
    return {"strategy":"RSI","asset":asset,"direction":direction,"confidence":confidence,"rsi":round(rsi,2),"candle":base_candle}

def support_resistance_signal_from_candles(candles: List[Dict], lookback:int, wick_ratio:float) -> Optional[str]:
    if not candles or len(candles) < lookback + 1: return None
    opens, highs, lows, closes = [], [], [], []
    for c in candles:
        try:
            opens.append(float(c["mid"]["o"]))
            highs.append(float(c["mid"]["h"]))
            lows.append(float(c["mid"]["l"]))
            closes.append(float(c["mid"]["c"]))
        except Exception:
            return None
    i = len(opens) - 1
    open_, close, high, low = opens[i], closes[i], highs[i], lows[i]
    prev_slice_start = max(0, i - lookback)
    prev_slice_end = i
    prev_lows = lows[prev_slice_start:prev_slice_end]
    prev_highs = highs[prev_slice_start:prev_slice_end]
    if len(prev_lows) < lookback or len(prev_highs) < lookback: return None
    support = min(prev_lows)
    resistance = max(prev_highs)
    body = abs(close - open_)
    upper_wick = high - max(open_, close)
    lower_wick = min(open_, close) - low
    if low <= support * 1.0002 and close > open_ and lower_wick > body * wick_ratio:
        return "CALL"
    if high >= resistance * 0.9998 and close < open_ and upper_wick > body * wick_ratio:
        return "PUT"
    return None

def strategy_sr_from_candles(asset: str, candles: List[Dict]) -> Optional[Dict]:
    lookback = int(get_param("SR", "lookback", Config.LOOKBACK))
    wick_ratio = float(get_param("SR", "wick_ratio", Config.WICK_RATIO))
    if not candles or len(candles) < lookback + 1: return None
    direction = support_resistance_signal_from_candles(candles, lookback, wick_ratio)
    if direction is None: return None
    last = candles[-1]
    try:
        o = float(last["mid"]["o"]) ; c = float(last["mid"]["c"]) ; h = float(last["mid"]["h"]) ; l = float(last["mid"]["l"])
    except Exception:
        return None
    body = abs(c - o)
    upper_wick = h - max(o, c)
    lower_wick = min(o, c) - l
    wick_strength = max(upper_wick, lower_wick)
    confidence = round(min(99.9, (wick_strength / (body + 1e-9)) * 100.0), 1) if body > 0 else 50.0
    base_candle = candles[-2] if len(candles) >= 2 else candles[-1]
    return {"strategy":"SR","asset":asset,"direction":direction,"confidence":confidence,"candle":base_candle}

def strategy_ma_from_candles(asset: str, candles: List[Dict]) -> Optional[Dict]:
    short = int(get_param("MA", "short", 9))
    longp = int(get_param("MA", "long", 21))
    if not candles or len(candles) < max(short, longp): return None
    try:
        closes = [float(c["mid"]["c"]) for c in candles]
    except Exception:
        return None
    def ma(period): return sum(closes[-period:]) / period
    short_ma = ma(short); long_ma = ma(longp)
    if short_ma > long_ma:
        return {"strategy":"MA","asset":asset,"direction":"CALL","confidence":round(min(99.9, (short_ma-long_ma)/long_ma*100),1),"candle":candles[-2]}
    if short_ma < long_ma:
        return {"strategy":"MA","asset":asset,"direction":"PUT","confidence":round(min(99.9, (long_ma-short_ma)/long_ma*100),1),"candle":candles[-2]}
    return None

def strategy_fib_from_candles(asset: str, candles: List[Dict]) -> Optional[Dict]:
    if not candles or len(candles) < 5: return None
    try:
        highs = [float(c["mid"]["h"]) for c in candles]
        lows = [float(c["mid"]["l"]) for c in candles]
        closes = [float(c["mid"]["c"]) for c in candles]
    except Exception:
        return None
    high = max(highs); low = min(lows)
    level_618 = low + (high - low) * 0.618
    close_price = closes[-2]
    if close_price > level_618:
        return {"strategy":"FIB","asset":asset,"direction":"PUT","confidence":round(min(99.9, abs(close_price-level_618)/high*100),1),"candle":candles[-2]}
    elif close_price < level_618:
        return {"strategy":"FIB","asset":asset,"direction":"CALL","confidence":round(min(99.9, abs(close_price-level_618)/high*100),1),"candle":candles[-2]}
    return None

def strategy_trend_from_candles(asset: str, candles: List[Dict]) -> Optional[Dict]:
    # allow param for length to check (default 4 closed candles)
    length = int(get_param("TREND", "length", 4))
    need = length + 1
    if not candles or len(candles) < need:
        return None
    try:
        last_group = candles[-(length+1):-1]  # last 'length' closed candles
        colors = []
        for c in last_group:
            o = float(c["mid"]["o"]); cl = float(c["mid"]["c"]) 
            if cl > o: colors.append("GREEN")
            elif cl < o: colors.append("RED")
            else: colors.append("FLAT")
    except Exception:
        return None
    if any(col == "FLAT" for col in colors): return None
    if all(col == "GREEN" for col in colors):
        direction = "CALL"
    elif all(col == "RED" for col in colors):
        direction = "PUT"
    else:
        return None
    try:
        start_close = float(last_group[0]["mid"]["c"])
        end_close = float(last_group[-1]["mid"]["c"])
        change_pct = abs((end_close - start_close) / max(abs(start_close), 1e-9)) * 100.0
        confidence = round(min(99.9, 50.0 + min(change_pct, 49.9)), 1)
    except Exception:
        confidence = 75.0
    base_candle = candles[-2]
    return {"strategy":"TREND","asset":asset,"direction":direction,"confidence":confidence,"candle":base_candle}

def strategy_tcm_from_candles(asset: str, candles: List[Dict]) -> Optional[Dict]:
    # params: body_thresh, wick_thresh, lookback_bodies
    body_thresh = float(get_param("TCM", "body_thresh", 1.3))
    wick_thresh = float(get_param("TCM", "wick_thresh", 0.6))
    lookback_bodies = int(get_param("TCM", "recent_bodies_count", 10))
    if not candles or len(candles) < 6: return None
    try:
        last3 = candles[-4:-1]
        dirs = []; bodies = []; upper_wicks = []; lower_wicks = []
        for c in last3:
            o = float(c["mid"]["o"]); cl = float(c["mid"]["c"]); h = float(c["mid"]["h"]); l = float(c["mid"]["l"])
            bodies.append(abs(cl - o))
            upper_wicks.append(h - max(o, cl))
            lower_wicks.append(min(o, cl) - l)
            if cl > o: dirs.append("GREEN")
            elif cl < o: dirs.append("RED")
            else: dirs.append("FLAT")
        if any(d == "FLAT" for d in dirs): return None
        recent_closed = candles[-(lookback_bodies+1):-1] if len(candles) >= (lookback_bodies+1) else candles[:-1]
        recent_bodies = [abs(float(c["mid"]["c"]) - float(c["mid"]["o"])) for c in recent_closed if "mid" in c]
        avg_body = (sum(recent_bodies)/len(recent_bodies)) if recent_bodies else (sum(bodies)/len(bodies))
        if not all(b > max(1e-9, avg_body * body_thresh) for b in bodies): return None
        last_up = upper_wicks[-1]; last_low = lower_wicks[-1]; last_body = bodies[-1]
        if last_body <= 1e-9: return None
        if max(last_up, last_low) > last_body * wick_thresh: return None
        if all(d == "GREEN" for d in dirs): direction = "PUT"
        elif all(d == "RED" for d in dirs): direction = "CALL"
        else: return None
        strength = sum(bodies) / (avg_body * 3 + 1e-9)
        confidence = round(min(99.9, 60 + min(39.9, (strength - 1.0) * 50)), 1)
        base_candle = candles[-2]
        return {"strategy":"TCM","asset":asset,"direction":direction,"confidence":confidence,"candle":base_candle}
    except Exception:
        return None

def strategy_wts_from_candles(asset: str, candles: List[Dict]) -> Optional[Dict]:
    wick_factor = float(get_param("WTS", "wick_factor", 1.5))
    lookback = int(get_param("WTS", "lookback", Config.LOOKBACK))
    if not candles or len(candles) < lookback + 3: return None
    try:
        last = candles[-2]
        o = float(last["mid"]["o"]); c = float(last["mid"]["c"]); h = float(last["mid"]["h"]); l = float(last["mid"]["l"])
        body = abs(c - o); upper_wick = h - max(o, c); lower_wick = min(o, c) - l
        if body <= 0: return None
        comp_slice = candles[-(lookback+3):-2] if len(candles) >= (lookback+4) else candles[:-2]
        highs = [float(cc["mid"]["h"]) for cc in comp_slice if "mid" in cc]
        lows  = [float(cc["mid"]["l"]) for cc in comp_slice if "mid" in cc]
        if not highs or not lows: return None
        resistance = max(highs); support = min(lows)
        if upper_wick > body * wick_factor and h > resistance * 0.9999 and c < resistance * 0.9995:
            confidence = round(min(99.9, (upper_wick / (body + 1e-9)) * 30 + 60), 1)
            base_candle = candles[-2]
            return {"strategy":"WTS","asset":asset,"direction":"PUT","confidence":confidence,"candle":base_candle}
        if lower_wick > body * wick_factor and l < support * 1.0001 and c > support * 1.0005:
            confidence = round(min(99.9, (lower_wick / (body + 1e-9)) * 30 + 60), 1)
            base_candle = candles[-2]
            return {"strategy":"WTS","asset":asset,"direction":"CALL","confidence":confidence,"candle":base_candle}
        return None
    except Exception:
        return None

def strategy_pis_from_candles(asset: str, candles: List[Dict]) -> Optional[Dict]:
    lookback = int(get_param("PIS", "lookback", 5))
    if not candles or len(candles) < lookback + 2: return None
    try:
        lastN = candles[-(lookback+1):-1]
        greens = 0; reds = 0; bodies = []
        for c in lastN:
            o = float(c["mid"]["o"]); cl = float(c["mid"]["c"]) 
            bodies.append(abs(cl - o))
            if cl > o: greens += 1
            elif cl < o: reds += 1
        if greens >= (lookback//2 + 1) and greens > reds:
            direction = "CALL"
        elif reds >= (lookback//2 + 1) and reds > greens:
            direction = "PUT"
        else:
            return None
        avg_body = sum(bodies)/len(bodies) if bodies else 0.0
        strength = sum(sorted(bodies, reverse=True)[:3]) / (avg_body*3 + 1e-9)
        confidence = round(min(99.9, 55 + min(44.9, (strength - 1.0) * 30)), 1)
        base_candle = candles[-2]
        return {"strategy":"PIS","asset":asset,"direction":direction,"confidence":confidence,"candle":base_candle}
    except Exception:
        return None

def strategy_srf_from_candles(asset: str, candles: List[Dict]) -> Optional[Dict]:
    """SRF - Signal Reversell Flow
    Heuristics:
      - Look at previous candle (the one before the latest closed candle).
      - Check "big trend" by comparing mean of recent closes vs older closes.
      - If big trend -> continuation (signal same direction as previous candle)
      - If trend not big -> favor reversal, but check SNR/round-number rejection: if rejection occurred, reverse.
      - Provide a confidence score (0-100)
    Returns a dict like: {"strategy":"SRF","direction":"CALL"/"PUT","confidence": float }
    """
    try:
        if not candles or len(candles) < 3:
            return None
        # assume candles[0] is latest closed, candles[1] is previous, etc.
        latest = candles[0]
        prev = candles[1]
        # extract closes for trend check
        closes = []
        for c in candles[:60]:
            try:
                for k in ("close","c","Close","C","ClosePrice"):
                    if k in c:
                        closes.append(float(c[k])); break
            except:
                continue
        if len(closes) < 10:
            return None
        recent_mean = sum(closes[:10]) / 10.0
        older_mean = sum(closes[10:30]) / max(1, len(closes[10:30]))
        trend_strength = (recent_mean - older_mean) / older_mean if older_mean != 0 else 0.0
        big_trend = abs(trend_strength) > 0.003  # >0.3% recent move considered "big"
        # prev candle direction
        try:
            o = float(prev.get('o') or prev.get('open'))
            cval = float(prev.get('c') or prev.get('close'))
        except:
            return None
        prev_dir = 'CALL' if cval > o else 'PUT'
        # check SNR/round-number rejection on prev
        # quick round-number check: close near integer or .50
        rn_checks = []
        for chk in (round(cval), round(cval*2)/2.0):
            rn_checks.append(abs(cval - chk) / max(1.0, abs(cval)) )
        close_to_round = min(rn_checks) < 0.0008  # within ~0.08%
        # wick rejection check on prev
        try:
            ph = float(prev.get('h') or prev.get('high'))
            pl = float(prev.get('l') or prev.get('low'))
            body = abs(cval - o)
            total = ph - pl if ph - pl != 0 else 1e-9
            upper = ph - max(cval,o)
            lower = min(cval,o) - pl
            rejection_up = upper / total > 0.6
            rejection_down = lower / total > 0.6
        except:
            rejection_up = rejection_down = False
        # Decide
        if big_trend:
            # follow prev candle direction
            direction = prev_dir
            confidence = min(95.0, 70.0 + abs(trend_strength) * 1000)
        else:
            # no big trend -> reversal favored
            # but if prev had clear rejection (at round-number or wick), reversal more likely
            if (prev_dir == 'CALL' and rejection_up) or (prev_dir == 'PUT' and rejection_down) or close_to_round:
                # reversal
                direction = 'PUT' if prev_dir == 'CALL' else 'CALL'
                confidence = 75.0
            else:
                # ambiguous: choose continuation but with lower confidence
                direction = prev_dir
                confidence = 50.0
        return {"strategy":"SRF","direction":direction,"confidence":round(confidence,1)}
    except Exception:
        return None

# lightweight input helpers used in interactive customization
def read_int(prompt: str, default: int = 0) -> int:
    try:
        v = input(prompt).strip()
        return int(v) if v != "" else int(default)
    except Exception:
        return int(default)

def read_float(prompt: str, default: float = 0.0) -> float:
    try:
        v = input(prompt).strip()
        return float(v) if v != "" else float(default)
    except Exception:
        return float(default)

def read_comma_ints(prompt: str, expected_count: Optional[int] = None, min_val: int = 1) -> Optional[List[int]]:
    try:
        v = input(prompt).strip()
        if not v:
            return None
        parts = [p.strip() for p in v.split(',') if p.strip()]
        ints = [int(x) for x in parts]
        if expected_count is not None and len(ints) != expected_count:
            print(f"Expected {expected_count} values but got {len(ints)}; ignoring.")
            return None
        ints = [max(min_val, i) for i in ints]
        return ints
    except Exception:
        return None

# --------------------
# EMA helper: compute EMA series from candle mid prices
# --------------------

def calculate_ema_series_from_mid(candles: List[Dict], period: int) -> Optional[List[Optional[float]]]:
    """
    Compute EMA series over `candles` using mid price = (o+h+l+c)/4.
    Returns a list of the same length as candles. Values before enough history are None.
    """
    if not candles or period < 1:
        return None
    mids: List[float] = []
    for c in candles:
        try:
            o = float(c['mid']['o']); h = float(c['mid']['h']); l = float(c['mid']['l']); cl = float(c['mid']['c'])
            mids.append((o + h + l + cl) / 4.0)
        except Exception:
            mids.append(None)
    # if any early None values, we still process but skip where necessary
    n = len(mids)
    ema: List[Optional[float]] = [None] * n
    # find first index where we have `period` valid mids
    valid_indices = [i for i, v in enumerate(mids) if v is not None]
    if len(valid_indices) < period:
        return ema
    # build initial SMA over first `period` valid values in order
    first_valid_index = valid_indices[0]
    # Collect the first `period` valid mid values (they must be consec? well accept first encountered)
    collected = []
    idx = first_valid_index
    while idx < n and len(collected) < period:
        if mids[idx] is not None:
            collected.append(mids[idx])
        idx += 1
    if len(collected) < period:
        return ema
    sma = sum(collected) / len(collected)
    # place SMA at the index of the last collected value
    seed_index = idx - 1
    ema[seed_index] = sma
    alpha = 2.0 / (period + 1.0)
    # compute EMA forward
    prev = sma
    for j in range(seed_index + 1, n):
        if mids[j] is None:
            ema[j] = None
            continue
        prev = alpha * mids[j] + (1 - alpha) * prev
        ema[j] = prev
    # earlier indices before seed_index remain None
    return ema

# --------------------
# EMAPULBACK strategy (adapted from the function you provided)
# --------------------

def strategy_emapulback_from_candles(asset: str, candles: List[Dict]) -> Optional[Dict]:
    """
    EMAPULBACK — EMA Pullback strategy (final, robust)
    """
    try:
        params = STRAT_PARAMS.get("EMAPULBACK", {})
        major_periods = params.get("ema_periods", [20, 50, 100, 200])
        touch_pct_threshold = float(params.get("touch_pct_threshold", 0.002))
        atr_lookback = int(params.get("atr_lookback", 10))
        atr_multiplier = float(params.get("atr_multiplier", 0.5))
        min_slope_pct = float(params.get("min_slope_pct", 0.01))
        require_full_stack = bool(params.get("require_full_stack", True))
    except Exception:
        major_periods = [20, 50, 100, 200]
        touch_pct_threshold = 0.002
        atr_lookback = 10
        atr_multiplier = 0.5
        min_slope_pct = 0.01
        require_full_stack = True

    # periods we need (5 + majors)
    all_periods = [5] + [int(p) for p in major_periods]
    maxp = max(all_periods)

    # need enough candles (use max period + 2 for prev/current)
    if not candles or len(candles) < maxp + 2:
        return None

    try:
        # compute EMAs using calculate_ema_series_from_mid (shared helper)
        ema_map = {}
        for p in all_periods:
            series = calculate_ema_series_from_mid(candles, int(p))
            if series is None:
                return None
            ema_map[int(p)] = series

        last_idx = len(candles) - 2   # base candle index (consistent with other strategies)
        prev_idx = last_idx - 1
        if prev_idx < 0:
            return None

        # EMA5 values and slope
        ema5 = ema_map[5][last_idx]
        ema5_prev = ema_map[5][prev_idx]
        if ema5 is None or ema5_prev is None:
            return None
        slope_pct = (ema5 - ema5_prev) / max(abs(ema5_prev), 1e-12) * 100.0

        # major EMA values at base index
        major_vals = {p: ema_map[p][last_idx] for p in major_periods}
        if any(v is None for v in major_vals.values()):
            return None

        # stack detection (strict)
        eps = 1e-12
        up_stack = all(major_vals[major_periods[i]] > major_vals[major_periods[i+1]] + eps for i in range(len(major_periods)-1))
        down_stack = all(major_vals[major_periods[i]] < major_vals[major_periods[i+1]] - eps for i in range(len(major_periods)-1))

        if not (up_stack or down_stack):
            if require_full_stack:
                return None
            else:
                # tolerant stacking: count pairwise agreements
                up_pairs = sum(1 for i in range(len(major_periods)-1) if major_vals[major_periods[i]] > major_vals[major_periods[i+1]])
                down_pairs = sum(1 for i in range(len(major_periods)-1) if major_vals[major_periods[i]] < major_vals[major_periods[i+1]])
                up_stack = (up_pairs >= down_pairs and up_pairs >= 2)
                down_stack = (down_pairs > up_pairs and down_pairs >= 2)
                if not (up_stack or down_stack):
                    return None

        # base candle OHLC for touch test
        base = candles[last_idx]
        try:
            o = float(base["mid"]["o"]); c = float(base["mid"]["c"]); h = float(base["mid"]["h"]); l = float(base["mid"]["l"])
        except Exception:
            return None

        # compute ATR-like average true range for normalization
        tr_vals = []
        start_tr_index = max(1, last_idx - atr_lookback + 1)
        for i in range(start_tr_index, last_idx + 1):
            try:
                high_i = float(candles[i]["mid"]["h"])
                low_i = float(candles[i]["mid"]["l"])
                prev_close = float(candles[i-1]["mid"]["c"])
                tr = max(high_i - low_i, abs(high_i - prev_close), abs(low_i - prev_close))
                tr_vals.append(tr)
            except Exception:
                continue
        atr = (sum(tr_vals) / len(tr_vals)) if tr_vals else max( (h - l), abs(c - o), 1e-9 )

        # touch test: either EMA5 inside base candle OR very near by pct or ATR threshold
        pct_thr = touch_pct_threshold * max(abs(c), 1.0)   # relative to price
        atr_thr = max(1e-12, atr * atr_multiplier)
        distance = abs(c - ema5)
        touched = (l <= ema5 <= h) or (distance <= max(pct_thr, atr_thr))

        if not touched:
            return None

        # require slope direction consistent with stack
        if up_stack and slope_pct <= min_slope_pct:
            return None
        if down_stack and slope_pct >= -min_slope_pct:
            return None

        # Determine direction
        direction = "CALL" if up_stack else "PUT"

        # Build confidence score (0-99.9)
        pair_count = len(major_periods) - 1
        correct_pairs = 0
        for i in range(pair_count):
            a = major_vals[major_periods[i]]; b = major_vals[major_periods[i+1]]
            if up_stack and a > b: correct_pairs += 1
            if down_stack and a < b: correct_pairs += 1
        stack_score = (correct_pairs / max(1, pair_count)) * 40.0

        slope_score = min(30.0, abs(slope_pct) * 2.5)

        prox_raw = 1.0 - min(1.0, distance / (atr + 1e-12))
        prox_score = prox_raw * 30.0

        confidence = round(min(99.9, 20.0 + stack_score + slope_score + prox_score), 1)

        base_candle = base
        return {
            "strategy": "EMAPULBACK",
            "asset": asset,
            "direction": direction,
            "confidence": confidence,
            "candle": base_candle
        }

    except Exception:
        return None

# --------------------
# EMASLOPE strategy (new)
# --------------------

def strategy_emaslope_from_candles(asset: str, candles: List[Dict]) -> Optional[Dict]:
    """
    EMASLOPE — Trend filter (single/multi EMA) + EMA5 slope + optional engulfing confirmation.
    """
    try:
        params = STRAT_PARAMS.get("EMASLOPE", {})
        trend_mode = params.get("trend_mode", "multi")  # "single" or "multi"
        trend_ema_periods = params.get("trend_ema_periods", [20, 50, 100, 200])
        require_trend = bool(params.get("require_trend", True))
        slope_min_pct = float(params.get("slope_min_pct", 0.01))
        engulfing_required = bool(params.get("engulfing_required", True))
        engulfing_min_body_pct = float(params.get("engulfing_min_body_pct", 0.15))
    except Exception:
        trend_mode = "multi"
        trend_ema_periods = [20, 50, 100, 200]
        require_trend = True
        slope_min_pct = 0.01
        engulfing_required = True
        engulfing_min_body_pct = 0.15

    all_periods = [5] + [int(p) for p in (trend_ema_periods if isinstance(trend_ema_periods, list) else [trend_ema_periods[0]])]
    maxp = max(all_periods)

    if not candles or len(candles) < maxp + 2:
        return None

    try:
        # compute EMAs
        ema_map = {}
        for p in all_periods:
            series = calculate_ema_series_from_mid(candles, int(p))
            if series is None:
                return None
            ema_map[int(p)] = series

        last_idx = len(candles) - 2
        prev_idx = last_idx - 1
        if prev_idx < 0:
            return None

        # EMA5 slope
        ema5 = ema_map[5][last_idx]
        ema5_prev = ema_map[5][prev_idx]
        if ema5 is None or ema5_prev is None:
            return None
        slope_pct = (ema5 - ema5_prev) / max(abs(ema5_prev), 1e-12) * 100.0

        # determine trend
        up_trend = False
        down_trend = False
        if trend_mode == "single":
            p = int(trend_ema_periods[0]) if isinstance(trend_ema_periods, list) else int(trend_ema_periods)
            ema_single = ema_map.get(p)
            if ema_single is None:
                return None
            ema_val = ema_single[last_idx]
            if ema_val is None:
                return None
            base = candles[last_idx]
            try:
                c = float(base["mid"]["c"])
            except Exception:
                return None
            if c > ema_val:
                up_trend = True
            elif c < ema_val:
                down_trend = True
        else:
            # multi EMA stacking
            trend_ema_periods_list = trend_ema_periods if isinstance(trend_ema_periods, list) else [trend_ema_periods]
            major_vals = {p: ema_map[p][last_idx] for p in trend_ema_periods_list}
            if any(v is None for v in major_vals.values()):
                return None
            eps = 1e-12
            up_stack = all(major_vals[trend_ema_periods_list[i]] > major_vals[trend_ema_periods_list[i+1]] + eps for i in range(len(trend_ema_periods_list)-1))
            down_stack = all(major_vals[trend_ema_periods_list[i]] < major_vals[trend_ema_periods_list[i+1]] - eps for i in range(len(trend_ema_periods_list)-1))
            if up_stack:
                up_trend = True
            elif down_stack:
                down_trend = True
            else:
                up_pairs = sum(1 for i in range(len(trend_ema_periods_list)-1) if major_vals[trend_ema_periods_list[i]] > major_vals[trend_ema_periods_list[i+1]])
                down_pairs = sum(1 for i in range(len(trend_ema_periods_list)-1) if major_vals[trend_ema_periods_list[i]] < major_vals[trend_ema_periods_list[i+1]])
                if up_pairs >= down_pairs and up_pairs >= 1:
                    up_trend = True
                elif down_pairs > up_pairs and down_pairs >= 1:
                    down_trend = True

        if require_trend and not (up_trend or down_trend):
            return None

        # slope direction must match trend
        if up_trend and slope_pct <= slope_min_pct:
            return None
        if down_trend and slope_pct >= -slope_min_pct:
            return None

        # engulfing confirmation (compare base candle vs previous)
        base = candles[last_idx]
        prev = candles[prev_idx]
        try:
            o = float(base["mid"]["o"]); c = float(base["mid"]["c"]); h = float(base["mid"]["h"]); l = float(base["mid"]["l"])
            po = float(prev["mid"]["o"]); pc = float(prev["mid"]["c"]); ph = float(prev["mid"]["h"]); pl = float(prev["mid"]["l"])
        except Exception:
            return None

        def body_low(o_, c_): return min(o_, c_)
        def body_high(o_, c_): return max(o_, c_)
        base_low = body_low(o, c); base_high = body_high(o, c)
        prev_low = body_low(po, pc); prev_high = body_high(po, pc)
        base_body = abs(c - o)

        bullish_engulf = (c > o) and (pc < po) and (base_low <= prev_low) and (base_high >= prev_high)
        bearish_engulf = (c < o) and (pc > po) and (base_low <= prev_low) and (base_high >= prev_high)

        min_body_ok = (base_body >= engulfing_min_body_pct * max(abs(c), 1.0))

        engulfing_ok = (bullish_engulf and min_body_ok) or (bearish_engulf and min_body_ok)

        if engulfing_required and not engulfing_ok:
            return None

        # decide direction
        direction = "CALL" if up_trend else "PUT" if down_trend else None
        if direction is None:
            return None

        # confidence: combine trend strength, slope magnitude, body size / engulfing
        trend_score = 0.0
        if trend_mode == "single":
            trend_score = 15.0
        else:
            pair_count = len(trend_ema_periods_list) - 1
            correct_pairs = 0
            for i in range(pair_count):
                a = ema_map[trend_ema_periods_list[i]][last_idx]; b = ema_map[trend_ema_periods_list[i+1]][last_idx]
                if up_trend and a > b: correct_pairs += 1
                if down_trend and a < b: correct_pairs += 1
            trend_score = (correct_pairs / max(1, pair_count)) * 30.0

        slope_score = min(40.0, abs(slope_pct) * 3.0)
        engulf_score = min(30.0, (base_body / max(1.0, abs(c))) * 100.0) if engulfing_ok else 0.0

        confidence = round(min(99.9, 10.0 + trend_score + slope_score + engulf_score), 1)

        return {
            "strategy": "EMASLOPE",
            "asset": asset,
            "direction": direction,
            "confidence": confidence,
            "candle": base
        }

    except Exception:
        return None        


from typing import List, Dict, Optional

# -------------------- EMA helper (fallback) --------------------
if 'calculate_ema_series_from_mid' not in globals():
    def calculate_ema_series_from_mid(candles: List[Dict], period: int) -> Optional[List[Optional[float]]]:
        """Simple EMA over mid prices (o+h+l+c)/4 ; returns list aligned with candles (None where not available)."""
        if not candles or period < 1:
            return None
        mids = []
        for c in candles:
            try:
                o = float(c['mid']['o']); h = float(c['mid']['h']); l = float(c['mid']['l']); cl = float(c['mid']['c'])
                mids.append((o + h + l + cl) / 4.0)
            except Exception:
                mids.append(None)
        n = len(mids)
        ema = [None] * n
        # seed with SMA of first `period` valid mids
        valid = [v for v in mids if v is not None]
        if len(valid) < period:
            return ema
        collected = []
        idx = 0
        while idx < n and len(collected) < period:
            if mids[idx] is not None:
                collected.append(mids[idx])
            idx += 1
        if len(collected) < period:
            return ema
        seed_index = idx - 1
        sma = sum(collected) / len(collected)
        ema[seed_index] = sma
        alpha = 2.0 / (period + 1.0)
        prev = sma
        for j in range(seed_index+1, n):
            if mids[j] is None:
                ema[j] = None
                continue
            prev = alpha * mids[j] + (1-alpha) * prev
            ema[j] = prev
        return ema

# -------------------- HTF aggregation & ATR --------------------
def aggregate_candles_for_htf(candles: List[Dict], factor: int) -> List[Dict]:
    """Aggregate consecutive `factor` candles into 1 higher-timeframe candle."""
    if factor is None or factor <= 1:
        return list(candles)
    out = []
    n = len(candles)
    i = 0
    while i < n:
        group = candles[i:i+factor]
        if not group:
            break
        try:
            o = float(group[0]['mid']['o'])
            c = float(group[-1]['mid']['c'])
            h = max(float(x['mid']['h']) for x in group)
            l = min(float(x['mid']['l']) for x in group)
            t = group[0].get('time', '')
            out.append({'time': t, 'mid': {'o': str(o), 'h': str(h), 'l': str(l), 'c': str(c)}})
        except Exception:
            pass
        i += factor
    return out


def compute_atr_series(candles: List[Dict], period: int) -> List[Optional[float]]:
    """Simple ATR series (SMA of True Range)."""
    n = len(candles)
    tr_list = [None] * n
    for i in range(1, n):
        try:
            high = float(candles[i]['mid']['h'])
            low = float(candles[i]['mid']['l'])
            prev_close = float(candles[i-1]['mid']['c'])
            tr = max(high - low, abs(high - prev_close), abs(low - prev_close))
            tr_list[i] = tr
        except Exception:
            tr_list[i] = None
    atr = [None] * n
    for i in range(n):
        if i < period:
            atr[i] = None
            continue
        window = [v for v in tr_list[i-period+1:i+1] if v is not None]
        atr[i] = (sum(window) / len(window)) if window else None
    return atr

# -------------------- ATR SuperTrend Bidirectional with EMA filter --------------------

def strategy_atr_supert_bidirectional_from_candles(asset: str, candles: List[Dict]) -> Optional[Dict]:
    """
    ATR SuperTrend (bidirectional) with optional EMA filter (single or multi).

    STRAT_PARAMS["ATR_SUPERT"] supports:
      - atr_period: int (default 10)
      - atr_multiplier: float (default 3.0)
      - require_reversal: bool (default True)
      - htf_factor: int (default 1)  # aggregation factor for HTF
      - use_htf_as_filter: bool (default False)  # optional HTF EMA200 filter
      - min_dist_atr_mult: float (default 0.15)
      - ema_filter_mode: str in ("none","single","multi") default 'none'
      - ema_filter_periods: list[int] (if single-> first used; if multi-> list used for stacking)
      - ema_filter_require: bool (if True EMA filter is enforced; if False EMA computed for plot only)
    """
    try:
        params = STRAT_PARAMS.get("ATR_SUPERT", {})
        atr_period = int(params.get("atr_period", 10))
        atr_multiplier = float(params.get("atr_multiplier", 3.0))
        require_reversal = bool(params.get("require_reversal", True))
        htf_factor = int(params.get("htf_factor", 1))
        use_htf_as_filter = bool(params.get("use_htf_as_filter", False))
        min_dist_atr_mult = float(params.get("min_dist_atr_mult", 0.15))
        ema_filter_mode = str(params.get("ema_filter_mode", "none")).lower()
        ema_filter_periods = params.get("ema_filter_periods", []) or []
        ema_filter_require = bool(params.get("ema_filter_require", False))
    except Exception:
        atr_period = 10
        atr_multiplier = 3.0
        require_reversal = True
        htf_factor = 1
        use_htf_as_filter = False
        min_dist_atr_mult = 0.15
        ema_filter_mode = "none"
        ema_filter_periods = []
        ema_filter_require = False

    if not candles or len(candles) < max(atr_period + 4, 30):
        return None

    n = len(candles)
    atr_series = compute_atr_series(candles, atr_period)

    # HL2 & basic bands
    hl2 = [None] * n
    for i in range(n):
        try:
            hi = float(candles[i]['mid']['h']); lo = float(candles[i]['mid']['l'])
            hl2[i] = (hi + lo) / 2.0
        except Exception:
            hl2[i] = None

    basic_upper = [None]*n
    basic_lower = [None]*n
    for i in range(n):
        if hl2[i] is None or atr_series[i] is None:
            basic_upper[i] = basic_lower[i] = None
            continue
        basic_upper[i] = hl2[i] + atr_multiplier * atr_series[i]
        basic_lower[i] = hl2[i] - atr_multiplier * atr_series[i]

    # final upper/lower smoothing & trend detection
    final_upper = [None]*n
    final_lower = [None]*n
    trend = [None]*n  # 'up' or 'down'
    for i in range(n):
        if basic_upper[i] is None or basic_lower[i] is None:
            continue
        if i == 0:
            final_upper[i] = basic_upper[i]
            final_lower[i] = basic_lower[i]
            try:
                c0 = float(candles[i]['mid']['c'])
                trend[i] = 'up' if c0 > hl2[i] else 'down'
            except Exception:
                trend[i] = 'down'
            continue

        prev_fu = final_upper[i-1]
        prev_fl = final_lower[i-1]
        prev_close = None
        try:
            prev_close = float(candles[i-1]['mid']['c'])
        except Exception:
            prev_close = None

        fu = basic_upper[i]
        fl = basic_lower[i]

        # update finals with smoothing rules
        if prev_fu is None:
            final_upper[i] = fu
        else:
            if fu < prev_fu or (prev_close is not None and prev_close > prev_fu):
                final_upper[i] = fu
            else:
                final_upper[i] = prev_fu

        if prev_fl is None:
            final_lower[i] = fl
        else:
            if fl > prev_fl or (prev_close is not None and prev_close < prev_fl):
                final_lower[i] = fl
            else:
                final_lower[i] = prev_fl

        # trend flip logic
        try:
            cval = float(candles[i]['mid']['c'])
            if trend[i-1] == 'down' and cval > prev_fu:
                trend[i] = 'up'
            elif trend[i-1] == 'up' and cval < prev_fl:
                trend[i] = 'down'
            else:
                trend[i] = trend[i-1]
        except Exception:
            trend[i] = trend[i-1] if i-1 >= 0 else 'down'

    base_idx = n - 2
    prev_idx = base_idx - 1
    if prev_idx < 0 or base_idx < 0:
        return None

    if trend[base_idx] is None or final_lower[base_idx] is None or final_upper[base_idx] is None:
        return None

    curr_trend = trend[base_idx]
    prev_trend = trend[prev_idx]

    # HTF EMA200 (unchanged)
    htf_ema200 = None
    try:
        htf_candles = aggregate_candles_for_htf(candles, htf_factor)
        if len(htf_candles) >= 200:
            htf_series = calculate_ema_series_from_mid(htf_candles, 200)
            if htf_series:
                for v in reversed(htf_series):
                    if v is not None:
                        htf_ema200 = v
                        break
    except Exception:
        htf_ema200 = None

    try:
        base_close = float(candles[base_idx]['mid']['c'])
    except Exception:
        return None

    # --- EMA filter logic (new) ---
    ema_meta = {'mode': ema_filter_mode, 'periods': ema_filter_periods, 'applied': False, 'values': None}
    ema_values = {}
    ema_ok_call = True
    ema_ok_put = True
    if ema_filter_mode in ('single', 'multi') and ema_filter_periods:
        # compute requested EMA series (on same timeframe as candles)
        periods = [int(p) for p in ema_filter_periods if int(p) > 0]
        for p in periods:
            series = calculate_ema_series_from_mid(candles, p)
            ema_values[p] = series[p and base_idx] if series else None
        ema_meta['values'] = ema_values

        # Evaluate stacking if multi
        if ema_filter_mode == 'single':
            p = periods[0]
            val = None
            try:
                val = calculate_ema_series_from_mid(candles, p)[base_idx]
            except Exception:
                val = None
            ema_values = {p: val}
            if ema_filter_require and val is not None:
                if call_ok and not (base_close > val):
                    ema_ok_call = False
                if put_ok and not (base_close < val):
                    ema_ok_put = False
            ema_meta['applied'] = ema_filter_require
        else:
            # multi: require stacking order for filter if required
            # stacking: periods must be ascending (short -> long)
            # e.g., [20,50,100] => up_stack if EMA20>EMA50>EMA100
            vals = []
            for p in periods:
                try:
                    v = calculate_ema_series_from_mid(candles, p)[base_idx]
                except Exception:
                    v = None
                vals.append((p, v))
            ema_meta['values'] = dict(vals)
            up_stack = all(vals[i][1] is not None and vals[i+1][1] is not None and vals[i][1] > vals[i+1][1] for i in range(len(vals)-1))
            down_stack = all(vals[i][1] is not None and vals[i+1][1] is not None and vals[i][1] < vals[i+1][1] for i in range(len(vals)-1))
            if ema_filter_require:
                if call_ok and not up_stack:
                    ema_ok_call = False
                if put_ok and not down_stack:
                    ema_ok_put = False
            ema_meta['applied'] = ema_filter_require

    # Apply EMA filter results
    if not ema_ok_call and not ema_ok_put:
        # both blocked
        return None
    if not ema_ok_call:
        call_ok = False
    if not ema_ok_put:
        put_ok = False

    # Signal decision
    call_ok = (curr_trend == 'up') and ((not require_reversal) or (prev_trend == 'down'))
    put_ok = (curr_trend == 'down') and ((not require_reversal) or (prev_trend == 'up'))

    # Re-apply EMA blocking (if earlier EMA blocked, ensure remains blocked)
    if ema_meta.get('applied'):
        if not ema_ok_call:
            call_ok = False
        if not ema_ok_put:
            put_ok = False

    # HTF EMA200 filter (optional)
    if use_htf_as_filter and htf_ema200 is not None:
        if call_ok and not (base_close > htf_ema200):
            call_ok = False
        if put_ok and not (base_close < htf_ema200):
            put_ok = False

    # Determine direction
    direction = None
    if call_ok and not put_ok:
        direction = "CALL"
    elif put_ok and not call_ok:
        direction = "PUT"
    elif call_ok and put_ok:
        # tie-breaker prefer reversal
        if prev_trend == 'down' and curr_trend == 'up':
            direction = "CALL"
        elif prev_trend == 'up' and curr_trend == 'down':
            direction = "PUT"
        else:
            direction = None
    else:
        return None

    # compute confidence
    atr_val = atr_series[base_idx] or 1e-9
    if direction == "CALL":
        distance = max(0.0, base_close - final_lower[base_idx])
    else:
        distance = max(0.0, final_upper[base_idx] - base_close)
    dist_ratio = distance / (atr_val + 1e-12)
    score = 10.0
    score += min(45.0, dist_ratio * 12.0)
    if (prev_trend != curr_trend):
        score += 20.0
    if htf_ema200 is not None:
        if (direction == "CALL" and base_close > htf_ema200) or (direction == "PUT" and base_close < htf_ema200):
            score += 10.0
    # small penalty if EMA filter applied but not confirmed (plot-only mode)
    if ema_filter_mode != 'none' and not ema_meta.get('applied'):
        score -= 2.0
    confidence = round(min(99.9, max(0.0, score)), 1)

    meta = {
        'final_upper': final_upper[base_idx],
        'final_lower': final_lower[base_idx],
        'curr_trend': curr_trend,
        'prev_trend': prev_trend,
        'atr': atr_val,
        'dist_ratio': dist_ratio,
        'htf_ema200': htf_ema200,
        'ema_filter': ema_meta
    }

    return {
        "strategy": "ATR_SUPERT",
        "asset": asset,
        "direction": direction,
        "confidence": confidence,
        "candle": candles[base_idx],
        "meta": meta
    }


from typing import List, Dict, Optional

# --- fallback helpers (only define if not present) ---
if 'calculate_ema_series_from_mid' not in globals():
    def calculate_ema_series_from_mid(candles: List[Dict], period: int) -> Optional[List[Optional[float]]]:
        """Fallback EMA on mid price (o+h+l+c)/4. Returns aligned list with None where insufficient history."""
        if not candles or period < 1:
            return None
        mids = []
        for c in candles:
            try:
                o = float(c['mid']['o']); h = float(c['mid']['h']); l = float(c['mid']['l']); cl = float(c['mid']['c'])
                mids.append((o + h + l + cl) / 4.0)
            except Exception:
                mids.append(None)
        n = len(mids)
        ema = [None] * n
        # seed EMA using first 'period' valid values (SMA)
        collected = []
        idx = 0
        while idx < n and len(collected) < period:
            if mids[idx] is not None:
                collected.append(mids[idx])
            idx += 1
        if len(collected) < period:
            return ema
        seed_index = idx - 1
        sma = sum(collected) / len(collected)
        ema[seed_index] = sma
        alpha = 2.0 / (period + 1.0)
        prev = sma
        for j in range(seed_index + 1, n):
            if mids[j] is None:
                ema[j] = None
                continue
            prev = alpha * mids[j] + (1 - alpha) * prev
            ema[j] = prev
        return ema

if 'compute_atr_series' not in globals():
    def compute_atr_series(candles: List[Dict], period: int) -> List[Optional[float]]:
        """Fallback simple ATR (SMA of True Range) aligned with candles."""
        n = len(candles)
        tr = [None] * n
        for i in range(1, n):
            try:
                h = float(candles[i]['mid']['h'])
                l = float(candles[i]['mid']['l'])
                pc = float(candles[i-1]['mid']['c'])
                tr[i] = max(h - l, abs(h - pc), abs(l - pc))
            except Exception:
                tr[i] = None
        atr = [None] * n
        for i in range(n):
            if i < period:
                atr[i] = None
                continue
            window = [v for v in tr[i-period+1:i+1] if v is not None]
            atr[i] = (sum(window) / len(window)) if window else None
        return atr

# -------------------------
# EMA200 + SuperTrend + EMA5 Slope Strategy
# -------------------------
def strategy_ema200_supertrend_from_candles(asset: str, candles: List[Dict]) -> Optional[Dict]:
    """
    EMA200 + SuperTrend + EMA5-slope confirmation strategy.

    STRAT_PARAMS["EMA200_SUPERT"] supports:
      - atr_period: int (default 10)
      - atr_multiplier: float (default 3.0)
      - require_reversal: bool (default True)   # require supertrend flip to trigger (optional)
      - min_ema5_slope_pct: float (default 0.01) # min relative slope required (0.01 == 1%)
      - min_body_pct: float (default 0.12)      # candle body / price minimum for confirmation
      - min_atr_dist_mult: float (default 0.15) # minimum distance from band in ATR multiples
      - require_all_confirmations: bool (default False)  # if True require all confirmations, else any one suffices
    """
    try:
        params = STRAT_PARAMS.get("EMA200_SUPERT", {})
        atr_period = int(params.get("atr_period", 10))
        atr_multiplier = float(params.get("atr_multiplier", 3.0))
        require_reversal = bool(params.get("require_reversal", True))
        min_ema5_slope_pct = float(params.get("min_ema5_slope_pct", 0.01))
        min_body_pct = float(params.get("min_body_pct", 0.12))
        min_atr_dist_mult = float(params.get("min_atr_dist_mult", 0.15))
        require_all_confirmations = bool(params.get("require_all_confirmations", False))
    except Exception:
        atr_period = 10
        atr_multiplier = 3.0
        require_reversal = True
        min_ema5_slope_pct = 0.01
        min_body_pct = 0.12
        min_atr_dist_mult = 0.15
        require_all_confirmations = False

    # need enough history: EMA200 + ATR window + safety
    needed = max(200, atr_period, 5) + 5
    if not candles or len(candles) < needed:
        return None

    n = len(candles)
    base_idx = n - 2    # base candle consistent with your other strategies
    prev_idx = base_idx - 1
    if prev_idx < 0:
        return None

    # --- compute series ---
    ema200_series = calculate_ema_series_from_mid(candles, 200)
    ema5_series = calculate_ema_series_from_mid(candles, 5)
    atr_series = compute_atr_series(candles, atr_period)

    if ema200_series is None or ema5_series is None or atr_series is None:
        return None

    ema200 = ema200_series[base_idx]
    ema5 = ema5_series[base_idx]
    ema5_prev = ema5_series[prev_idx] if prev_idx >= 0 else None
    if ema200 is None or ema5 is None or ema5_prev is None:
        return None

    # --- SuperTrend (final upper/lower & trend) ---
    # compute hl2 and basic bands
    hl2 = [None] * n
    for i in range(n):
        try:
            hi = float(candles[i]['mid']['h']); lo = float(candles[i]['mid']['l'])
            hl2[i] = (hi + lo) / 2.0
        except Exception:
            hl2[i] = None

    basic_up = [None] * n
    basic_lo = [None] * n
    for i in range(n):
        if hl2[i] is None or atr_series[i] is None:
            basic_up[i] = basic_lo[i] = None
            continue
        basic_up[i] = hl2[i] + atr_multiplier * atr_series[i]
        basic_lo[i] = hl2[i] - atr_multiplier * atr_series[i]

    final_up = [None] * n
    final_lo = [None] * n
    trend = [None] * n  # 'up' or 'down'
    for i in range(n):
        if basic_up[i] is None or basic_lo[i] is None:
            continue
        if i == 0:
            final_up[i] = basic_up[i]
            final_lo[i] = basic_lo[i]
            try:
                c0 = float(candles[i]['mid']['c'])
                trend[i] = 'up' if c0 > hl2[i] else 'down'
            except Exception:
                trend[i] = 'down'
            continue

        prev_fu = final_up[i-1]
        prev_fl = final_lo[i-1]
        prev_close = None
        try:
            prev_close = float(candles[i-1]['mid']['c'])
        except Exception:
            prev_close = None

        fu = basic_up[i]; fl = basic_lo[i]
        # final up
        if prev_fu is None:
            final_up[i] = fu
        else:
            if fu < prev_fu or (prev_close is not None and prev_close > prev_fu):
                final_up[i] = fu
            else:
                final_up[i] = prev_fu
        # final lo
        if prev_fl is None:
            final_lo[i] = fl
        else:
            if fl > prev_fl or (prev_close is not None and prev_close < prev_fl):
                final_lo[i] = fl
            else:
                final_lo[i] = prev_fl

        # trend flip
        try:
            cval = float(candles[i]['mid']['c'])
            if trend[i-1] == 'down' and cval > prev_fu:
                trend[i] = 'up'
            elif trend[i-1] == 'up' and cval < prev_fl:
                trend[i] = 'down'
            else:
                trend[i] = trend[i-1]
        except Exception:
            trend[i] = trend[i-1] if i-1 >= 0 else 'down'

    # need trend at base
    if trend[base_idx] is None or final_lo[base_idx] is None or final_up[base_idx] is None:
        return None

    curr_trend = trend[base_idx]
    prev_trend = trend[prev_idx]

    # base candle values
    try:
        base_c = float(candles[base_idx]['mid']['c'])
        base_o = float(candles[base_idx]['mid']['o'])
        base_h = float(candles[base_idx]['mid']['h'])
        base_l = float(candles[base_idx]['mid']['l'])
    except Exception:
        return None

    # --- Big trend via EMA200 ---
    # if ema200 is None skip
    try:
        ema200_val = float(ema200)
    except Exception:
        return None

    big_trend_up = base_c > ema200_val
    big_trend_down = base_c < ema200_val

    # --- Confirmation tests ---
    confirmations = {}

    # 1) EMA5 slope confirmation (relative percent)
    try:
        slope_pct = (ema5 - ema5_prev) / max(abs(ema5_prev), 1e-12)
    except Exception:
        slope_pct = 0.0
    confirmations['ema5_slope_pct'] = slope_pct
    conf_slope_ok = False
    if slope_pct >= min_ema5_slope_pct:
        conf_slope_ok = True
    if slope_pct <= -min_ema5_slope_pct:
        conf_slope_ok = True  # slope in either direction counts (well later compare with direction)

    # 2) Candle body confirmation: body size relative to price
    body = abs(base_c - base_o)
    body_pct = body / max(abs(base_c), 1e-12)
    confirmations['body_pct'] = body_pct
    conf_body_ok = body_pct >= min_body_pct

    # 3) ATR-distance from band: distance / ATR
    atr_val = atr_series[base_idx] or 1e-9
    if curr_trend == 'up':
        distance = max(0.0, base_c - final_lo[base_idx])
    else:
        distance = max(0.0, final_up[base_idx] - base_c)
    dist_ratio = distance / (atr_val + 1e-12)
    confirmations['dist_ratio'] = dist_ratio
    conf_dist_ok = dist_ratio >= min_atr_dist_mult

    # Evaluate confirmation logic
    # If require_all_confirmations=True -> need slope + body + distance (matching direction)
    # Else -> need at least one confirmation that matches the direction
    matched_confirmations = []

    # slope direction match
    slope_dir = 1 if slope_pct > 0 else (-1 if slope_pct < 0 else 0)
    if slope_dir != 0:
        if (slope_dir > 0 and curr_trend == 'up') or (slope_dir < 0 and curr_trend == 'down'):
            if abs(slope_pct) >= min_ema5_slope_pct:
                matched_confirmations.append('ema5_slope')

    if conf_body_ok:
        # body sign must agree with trend: bullish body for up, bearish for down
        if (base_c > base_o and curr_trend == 'up') or (base_c < base_o and curr_trend == 'down'):
            matched_confirmations.append('body')

    if conf_dist_ok:
        matched_confirmations.append('atr_distance')

    # decide confirmation pass
    if require_all_confirmations:
        conf_pass = all(k in matched_confirmations for k in ('ema5_slope', 'body', 'atr_distance'))
    else:
        conf_pass = len(matched_confirmations) >= 1

    # require reversal optional: if set, we want prev_trend != curr_trend
    reversal = (prev_trend != curr_trend)
    if require_reversal and not reversal:
        return None

    # Now check big trend alignment:
    # Only allow CALL if big_trend_up and curr_trend == 'up' and conf_pass and slope_dir positive
    # Only allow PUT if big_trend_down and curr_trend == 'down' and conf_pass and slope_dir negative
    direction = None
    if big_trend_up and curr_trend == 'up' and conf_pass:
        # ensure slope matches up
        if slope_dir >= 0:
            direction = "CALL"
    if big_trend_down and curr_trend == 'down' and conf_pass:
        if slope_dir <= 0:
            direction = "PUT"

    if direction is None:
        return None

    # compute confidence score (0-99.9)
    score = 10.0
    # give bonus for reversal
    if reversal:
        score += 25.0
    # slope magnitude contribution (capped)
    score += min(30.0, abs(slope_pct) * 1000.0)  # scale to visible range; tweak if needed
    # body contribution
    score += min(20.0, body_pct * 100.0)
    # distance contribution
    score += min(20.0, dist_ratio * 10.0)
    # ensure within range
    confidence = round(min(99.9, score), 1)

    meta = {
        "ema200": ema200_val,
        "ema5": ema5,
        "ema5_prev": ema5_prev,
        "atr": atr_val,
        "final_upper": final_up[base_idx],
        "final_lower": final_lo[base_idx],
        "curr_trend": curr_trend,
        "prev_trend": prev_trend,
        "reversal": reversal,
        "confirmations_required_all": require_all_confirmations,
        "matched_confirmations": matched_confirmations,
        "confirmations": confirmations
    }

    base_candle = candles[base_idx]
    return {
        "strategy": "EMA200_SUPERT_SLOPE",
        "asset": asset,
        "direction": direction,
        "confidence": confidence,
        "candle": base_candle,
        "meta": meta
    }

STRATEGY_FUNC_MAP = {
    "SRF": strategy_srf_from_candles,
    "RSI": strategy_rsi_from_candles,
    "SR": strategy_sr_from_candles,
    "MA": strategy_ma_from_candles,
    "FIB": strategy_fib_from_candles,
    "TREND": strategy_trend_from_candles,
    "TCM": strategy_tcm_from_candles,
    "WTS": strategy_wts_from_candles,
    "PIS": strategy_pis_from_candles,
    "MACD": strategy_macd_divergence_from_candles,
    "EMAPULBACK": strategy_emapulback_from_candles,
    "EMASLOPE": strategy_emaslope_from_candles,
    "ATR_SUPERT": strategy_atr_supert_bidirectional_from_candles,
    "EMA200_SUPERTREND": strategy_ema200_supertrend_from_candles,
}

# -------------------- End of module --------------------

# =============== Formatting messages & charts ===============
def format_signal(asset: str, signal_time: datetime.datetime, direction: str, taguserid: str, lead_minutes: int = 1, last_price: str = None) -> str:
    if last_price is None:
        try:
            last_price = get_last_candle_price(asset)
        except Exception:
            last_price = None
    signal_time = signal_time.astimezone(TIMEZONE)
    trade_place_time = (signal_time + datetime.timedelta(minutes=lead_minutes)).strftime("%H:%M")
    direction_icon = '🟢 CALL' if direction == 'CALL' else '🔴 PUT'
    last_price_line = f"🎯 PRICE: {to_mono(str(last_price))}" if last_price is not None else "🎯 PRICE: N/A"
    return f"""
💫 𝗔𝗜 𝗕𝗢𝗧 𝗦𝗜𝗚𝗡𝗔𝗟 💫

==================
📊 {to_mono(asset)}
⏳ {to_mono(str(lead_minutes))} 𝙼𝚒𝚗𝚞𝚝𝚎
⏰ {to_mono(trade_place_time)}
{to_mono(direction_icon)}
{to_mono(last_price_line)}
==================
🎯 USE {to_mono(str(MARTINGALE_STEPS_SIGNAL))} MTG 💰
🏆 UTC,GMT +06:00 

🔊 𝗖𝗢𝗡𝗧𝗔𝗖𝗧: {taguserid}
""".strip()

TOTAL_WINS = 0
TOTAL_LOSSES = 0


def format_result(
    asset: str,
    signal_time: datetime.datetime,
    direction: str,
    outcome: str,          # "WIN", "LOSS", or "S_WIN" (safety win)
    taguserid: str,
    mtg_step: int = 0      # 0 = Non MTG, 1 = MTG 1, etc.
) -> str:
    """
    Format and return trade result message using the new layout requested by the user.
    This function will increment the global counters for wins/losses.
    """

    global TOTAL_WINS, TOTAL_LOSSES

    # --- Time handling ---
    signal_time = signal_time.astimezone(TIMEZONE)
    trade_place_time = (signal_time + datetime.timedelta(minutes=1)).strftime("%H:%M")

    # --- Result line ---
    if outcome == "WIN":
        TOTAL_WINS += 1
        if mtg_step == 0:
            result_line = "🎯 RESULT: ✅ NON-MTG SURESHOT ✅"
        else:
            result_line = f"🎯 RESULT: ✅ MTG {mtg_step} SURESHOT ✅"

    elif outcome == "S_WIN":
        TOTAL_WINS += 1
        result_line = "🎯 RESULT: ✅ SAFETY WIN ✅"

    else:  # LOSS or anything else treated as loss
        TOTAL_LOSSES += 1
        result_line = "🎯 RESULT: 👎 LOSS 👎"

    # --- Stats ---
    total_trades = TOTAL_WINS + TOTAL_LOSSES
    win_rate = round((TOTAL_WINS / total_trades) * 100, 1) if total_trades > 0 else 0.0

    # --- Final message (matches the users requested format) ---
    return f"""
======= 𝗥𝗘𝗦𝗨𝗟𝗧 𝗔𝗟𝗘𝗥𝗧 =======

📊 𝙿𝙰𝙸𝚁:- {to_mono(asset)}
⏰ 𝚃𝚒𝚖𝚎:- {to_mono(trade_place_time)}

{result_line}

🔊 𝙵𝙴𝙴𝙳𝙱𝙰𝙲𝙺: {taguserid}
""".strip()


def get_last_candle_price(instrument: str) -> Optional[str]:
    candles = live_candle_cache.get(instrument)
    if not candles:
        candles = fetch_otc_candles(instrument, count=1)
    if candles and isinstance(candles, list) and len(candles) > 0:
        try: return str(candles[0]["mid"]["c"])
        except Exception: return None
    return None

def format_summary():
    wins = sum(1 for r in TRADE_HISTORY if r.get("outcome") == "WIN")
    losses = sum(1 for r in TRADE_HISTORY if r.get("outcome") == "LOSS")
    total = len(TRADE_HISTORY)
    wr = round((wins / total) * 100, 1) if total > 0 else 0.0
    win_lines = [to_mono(f"{r['time']},{r['asset']},{r['dir']}  ✅") for r in TRADE_HISTORY if r.get("outcome") == "WIN"]
    loss_times = [to_mono(r['time']) for r in TRADE_HISTORY if r.get("outcome") == "LOSS"]
    summary = f"""
{to_mono("𒆜=====『 𝗙𝗜𝗡𝗔𝗟 𝗥𝗘𝗦𝗨𝗟𝗧 』======𒆜️")}

┏====================┓
 {to_mono("📆 - " + datetime.datetime.now(TIMEZONE).strftime('%Y.%m.%d'))}
<====================>
 {to_mono("OTC MARKET")}
┏====================┓

{chr(10).join(win_lines) if win_lines else to_mono("No WIN signals")}
"""
    if loss_times:
        summary += f"""
┏====================┓
❌ {to_mono("LOSS TIMES " + ", ".join(loss_times))}
"""
    summary += f"""
<====================>
📊  {to_mono(f"Total Signal : {total} ✠ Ratio: ({wr}%)")}
<====================>
{to_mono(f"TOTAL WIN : {wins}  TOTAL LOSS: {losses}")}
"""
    return summary

# =============== Chart helpers & Telegram photo send ===============
def format_chart_time_for_message(msg_time: datetime.datetime) -> str:
    if msg_time.tzinfo is None:
        msg_time = msg_time.replace(tzinfo=datetime.timezone.utc)
    local = msg_time.astimezone(TIMEZONE)
    chart_time = local - datetime.timedelta(minutes=1)
    return chart_time.strftime("%H:%M")

def build_chart_url_for_pair(pair: str, msg_time: datetime.datetime) -> str:
    if not pair:
        return f"{CHART_BASE_URL}"
    p = normalize_pair_for_api(pair)
    hm = format_chart_time_for_message(msg_time)
    return f"{CHART_BASE_URL}?pair={p}&time={hm}"

async def send_telegram_with_optional_chart(message: str, pair: str = None, msg_time: datetime.datetime = None):
    if msg_time is None:
        msg_time = datetime.datetime.now(datetime.timezone.utc)
    if SEND_CHARTS and pair:
        chart_url = build_chart_url_for_pair(pair, msg_time)
        url = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendPhoto"
        caption = f"<b>{html.escape(message)}</b>"
        payload = {"chat_id": TELEGRAM_CHAT_ID, "photo": chart_url, "caption": caption, "parse_mode": "HTML"}
        try:
            resp = await asyncio.to_thread(requests.post, url, data=payload, timeout=Config.TELEGRAM_TIMEOUT)
            if resp.status_code != 200:
                log_err(f"Telegram sendPhoto error {resp.status_code}: {resp.text}")
                await send_telegram_message_bold(message)
        except Exception as e:
            log_err(f"Telegram sendPhoto exception: {e}")
            await send_telegram_message_bold(message)
    else:
        await send_telegram_message_bold(message)

async def send_telegram_message_bold(message: str):
    url = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"
    safe_message = f"<b>{html.escape(message)}</b>"
    payload = {"chat_id": TELEGRAM_CHAT_ID, "text": safe_message, "parse_mode": "HTML"}
    try:
        resp = await asyncio.to_thread(requests.post, url, data=payload, timeout=Config.TELEGRAM_TIMEOUT)
        if resp.status_code != 200:
            log_err(f"Telegram send error {resp.status_code}: {resp.text}")
    except Exception as e:
        log_err(f"Telegram send exception: {e}")

# =============== Trade processing with configurable MTG steps ===============
def get_candle_at_time(instrument: str, target_time: datetime.datetime, retries: int = 90, sleep_s: float = 1.0):
    target_str = target_time.astimezone(TIMEZONE).strftime("%Y-%m-%d %H:%M:00")
    for i in range(retries):
        candles = fetch_otc_candles(instrument, count=30)
        if candles:
            for candle in candles:
                candle_time = candle.get("time", "")
                if "T" in candle_time:
                    candle_time = candle_time.replace("T", " ").split(".")[0]
                if target_str in candle_time:
                    return candle
        time.sleep(sleep_s)
    return None

async def process_trade_for_signal(signal_data: dict, signal_time: datetime.datetime):
    asset = signal_data["asset"]
    direction = signal_data["direction"]
    trade_place_time = (signal_time + datetime.timedelta(minutes=1)).replace(second=0, microsecond=0)
    log_info(f"Starting trade checks for {asset} direction {direction} at base {trade_place_time.astimezone(TIMEZONE).strftime('%H:%M')}, MTG steps allowed (signal): {MARTINGALE_STEPS_SIGNAL}")

    # holder for any earlier safety detection (step, attempt_time, candle)
    saved_safety = None  # (step, attempt_time, candle)

    last_step = MARTINGALE_STEPS_SIGNAL
    for step in range(0, last_step + 1):
        attempt_time = trade_place_time + datetime.timedelta(minutes=step)
        log_info(f"Waiting candle at {attempt_time.astimezone(TIMEZONE).strftime('%H:%M')} (mtg +{step}) for {asset} ...")

        candle = get_candle_at_time(asset, attempt_time, retries=120, sleep_s=2.0)
        if not candle:
            log_err(f"{asset}: Candle for mtg+{step} at {attempt_time.astimezone(TIMEZONE).strftime('%H:%M')} not found.")
            # if couldn fetch and this is last step -> decide using saved_safety or LOSS
            if step == last_step:
                if saved_safety:
                    s_step, s_time, s_candle = saved_safety
                    res_msg = format_result(asset, signal_time, direction, "S_WIN", TAG_USER_ID, mtg_step=s_step)
                    await send_telegram_with_optional_chart(res_msg, asset, s_time.astimezone(datetime.timezone.utc))
                    TRADE_HISTORY.append({"time": s_time.astimezone(TIMEZONE).strftime("%H:%M"), "asset": asset, "dir": direction, "outcome": "S_WIN", "mtg_step": s_step, "safety": True})
                    # terminal summary print
                    total_trades = TOTAL_WINS + TOTAL_LOSSES
                    win_rate = round((TOTAL_WINS / total_trades) * 100, 1) if total_trades > 0 else 0.0
                    print("\n---------------------------------")
                    print(f"🏆 Win: {TOTAL_WINS} | Loss: {TOTAL_LOSSES} ✠ ({win_rate}%)")
                    print("---------------------------------\n")
                    log_info(f"{asset}: Sent saved S_WIN (mtg+{s_step}) because final candle missing.")
                    return
                else:
                    res_msg = format_result(asset, signal_time, direction, "LOSS", TAG_USER_ID, mtg_step=step)
                    await send_telegram_with_optional_chart(res_msg, asset, datetime.datetime.now(datetime.timezone.utc))
                    TRADE_HISTORY.append({"time": attempt_time.astimezone(TIMEZONE).strftime("%H:%M"), "asset": asset, "dir": direction, "outcome": "LOSS"})
                    # terminal summary print
                    total_trades = TOTAL_WINS + TOTAL_LOSSES
                    win_rate = round((TOTAL_WINS / total_trades) * 100, 1) if total_trades > 0 else 0.0
                    print("\n---------------------------------")
                    print(f"🏆 Win: {TOTAL_WINS} | Loss: {TOTAL_LOSSES} ✠ ({win_rate}%)")
                    print("---------------------------------\n")
                    return
            else:
                # try next mtg
                continue

        cand_dir = candle_direction(candle)
        log_debug(f"{asset} mtg+{step} candle dir={cand_dir} target={direction}")

        # 1) Direct WIN -> immediate send and return
        if cand_dir == direction:
            res_msg = format_result(asset, signal_time, direction, "WIN", TAG_USER_ID, mtg_step=step)
            await send_telegram_with_optional_chart(res_msg, asset, attempt_time.astimezone(datetime.timezone.utc))
            TRADE_HISTORY.append({"time": attempt_time.astimezone(TIMEZONE).strftime("%H:%M"), "asset": asset, "dir": direction, "outcome": "WIN", "mtg_step": step})
            # terminal summary print
            total_trades = TOTAL_WINS + TOTAL_LOSSES
            win_rate = round((TOTAL_WINS / total_trades) * 100, 1) if total_trades > 0 else 0.0
            print("\n---------------------------------")
            print(f"🏆 Win: {TOTAL_WINS} | Loss: {TOTAL_LOSSES} ✠ ({win_rate}%)")
            print("---------------------------------\n")
            log_info(f"{asset}: WIN at mtg+{step}.")
            return

        # 2) Safety check on this candle
        try:
            use_safety_live = USE_SAFETY_MARGIN
        except NameError:
            use_safety_live = False

        if use_safety_live and safety_margin_check(candle, direction):
            # If not last step: save and dont send yet (postpone)
            if step < last_step:
                saved_safety = (step, attempt_time, candle)
                log_info(f"{asset}: SAFETY detected at mtg+{step} — postponed (will check next mtg).")
                # continue to next mtg to see if it converts to direct WIN
                continue
            else:
                # last step: send S_WIN immediately (no later mtg to wait)
                res_msg = format_result(asset, signal_time, direction, "S_WIN", TAG_USER_ID, mtg_step=step)
                await send_telegram_with_optional_chart(res_msg, asset, attempt_time.astimezone(datetime.timezone.utc))
                TRADE_HISTORY.append({"time": attempt_time.astimezone(TIMEZONE).strftime("%H:%M"), "asset": asset, "dir": direction, "outcome": "S_WIN", "mtg_step": step, "safety": True})
                # terminal summary print
                total_trades = TOTAL_WINS + TOTAL_LOSSES
                win_rate = round((TOTAL_WINS / total_trades) * 100, 1) if total_trades > 0 else 0.0
                print("\n---------------------------------")
                print(f"🏆 Win: {TOTAL_WINS} | Loss: {TOTAL_LOSSES} ✠ ({win_rate}%)")
                print("---------------------------------\n")
                log_info(f"{asset}: S_WIN at final mtg+{step}.")
                return

        # 3) Not direct win, not safety
        if step < last_step:
            log_info(f"{asset}: mtg+{step} DID NOT WIN and no SAFETY → trying mtg+{step+1}")
            continue
        else:
            # final mtg completed and no direct win / no safety in this last candle
            # If we had saved safety earlier, send that saved S_WIN now (prefer earliest saved)
            if saved_safety:
                s_step, s_time, s_candle = saved_safety
                res_msg = format_result(asset, signal_time, direction, "S_WIN", TAG_USER_ID, mtg_step=s_step)
                await send_telegram_with_optional_chart(res_msg, asset, s_time.astimezone(datetime.timezone.utc))
                TRADE_HISTORY.append({"time": s_time.astimezone(TIMEZONE).strftime("%H:%M"), "asset": asset, "dir": direction, "outcome": "S_WIN", "mtg_step": s_step, "safety": True})
                # terminal summary print
                total_trades = TOTAL_WINS + TOTAL_LOSSES
                win_rate = round((TOTAL_WINS / total_trades) * 100, 1) if total_trades > 0 else 0.0
                print("\n---------------------------------")
                print(f"🏆 Win: {TOTAL_WINS} | Loss: {TOTAL_LOSSES} ✠ ({win_rate}%)")
                print("---------------------------------\n")
                log_info(f"{asset}: No conversion on later mtgs → sending previously saved S_WIN (mtg+{s_step}).")
                return
            else:
                # no safety found anywhere → LOSS
                res_msg = format_result(asset, signal_time, direction, "LOSS", TAG_USER_ID, mtg_step=step)
                await send_telegram_with_optional_chart(res_msg, asset, attempt_time.astimezone(datetime.timezone.utc))
                TRADE_HISTORY.append({"time": attempt_time.astimezone(TIMEZONE).strftime("%H:%M"), "asset": asset, "dir": direction, "outcome": "LOSS", "mtg_step": step})
                # terminal summary print
                total_trades = TOTAL_WINS + TOTAL_LOSSES
                win_rate = round((TOTAL_WINS / total_trades) * 100, 1) if total_trades > 0 else 0.0
                print("\n---------------------------------")
                print(f"🏆 Win: {TOTAL_WINS} | Loss: {TOTAL_LOSSES} ✠ ({win_rate}%)")
                print("---------------------------------\n")
                log_info(f"{asset}: Final LOSS at mtg+{step}.")
                return

# =============== Per-asset processing (unchanged main flow) ===============
async def process_asset(asset: str, signal_time: datetime.datetime) -> Optional[Dict]:
    trade_time_hm = (signal_time + timedelta(minutes=1)).astimezone(TIMEZONE).strftime("%H:%M")
    needed = max(max([int(x) for x in (get_param("EMA","periods", Config.EMA_PERIODS) if get_param("EMA","periods", Config.EMA_PERIODS) else Config.EMA_PERIODS)]), 220)
    try:
        live_candles = await asyncio.to_thread(fetch_otc_candles, asset, needed)
    except Exception:
        live_candles = []
    if not live_candles:
        return {"asset": asset, "status": "NO_DATA", "wins": 0, "losses": 0, "confidence": 0.0, "agreed_strategies": []}
    live_candle_cache[asset] = live_candles

    strat_tasks = []
    for s in SELECTED_STRATS:
        fn = STRATEGY_FUNC_MAP.get(s)
        if not fn: continue
        strat_tasks.append(asyncio.to_thread(fn, asset, live_candles))
    strat_results = await asyncio.gather(*strat_tasks) if strat_tasks else []
    strat_signals = [r for r in strat_results if r]
    if not strat_signals:
        return {"asset": asset, "status": "REJECT", "wins": 0, "losses": 0, "confidence": 0.0, "agreed_strategies": []}

    pair_for_api = normalize_pair_for_api(asset)
    start_offset = 2 if SKIP_RECENT else 1
    fetch_days = BACKTEST_DAYS + start_offset
    if pair_for_api in hist_candle_cache:
        parsed_hist = hist_candle_cache[pair_for_api]
    else:
        try:
            parsed_hist = await asyncio.to_thread(fetch_candles_for_pair, pair_for_api, fetch_days, Config.DEBUG)
        except Exception:
            parsed_hist = []
        hist_candle_cache[pair_for_api] = parsed_hist
    if not parsed_hist:
        return {"asset": asset, "status": "NO_DATA", "wins": 0, "losses": 0, "confidence": 0.0, "agreed_strategies": []}
    cmap = build_candle_map(parsed_hist)

    backtest_tasks = []
    for sig in strat_signals:
        # use MARTINGALE_STEPS_BACKTEST for filtering/backtest
        backtest_tasks.append(asyncio.to_thread(backtest_with_cmap, cmap, trade_time_hm, sig['direction'], BACKTEST_DAYS, MARTINGALE_STEPS_BACKTEST, SKIP_RECENT, Config.DEBUG))
    backtest_results = await asyncio.gather(*backtest_tasks) if backtest_tasks else []

    passed = []
    wins_total = 0; losses_total = 0
    for sig, res in zip(strat_signals, backtest_results):
        okflag, details = res
        w = sum(1 for d in details if 'WIN' in d.get('status',''))
        l = sum(1 for d in details if 'LOSS' in d.get('status',''))
        wins_total += w; losses_total += l
        if okflag:
            sig_copy = dict(sig)
            sig_copy['backtest'] = True
            sig_copy['backtest_details'] = details
            passed.append(sig_copy)

    if not passed:
        return {"asset": asset, "status": "REJECT", "wins": wins_total, "losses": losses_total, "confidence": 0.0, "agreed_strategies": []}

    by_dir = {}
    for p in passed:
        d = p['direction']
        by_dir.setdefault(d, []).append(p)
    chosen_direction = None
    chosen_signal = None
    best_conf = -1.0
    for d, lst in by_dir.items():
        local_best = max(lst, key=lambda x: x.get('confidence',0))
        if local_best.get('confidence',0) > best_conf:
            best_conf = local_best.get('confidence',0)
            chosen_direction = d
            chosen_signal = local_best
    chosen_signal['agreed_strategies'] = [p['strategy'] for p in by_dir.get(chosen_direction, [])]
    chosen_signal['asset'] = asset
    return {"asset": asset, "status": "PASS", "wins": wins_total, "losses": losses_total, "confidence": chosen_signal.get('confidence',0.0), "agreed_strategies": chosen_signal.get('agreed_strategies',[]), "direction": chosen_direction, "chosen_signal": chosen_signal}

# =============== Orchestrator ===============
async def process_all_assets_once():
    log_info("Scanning assets with selected strategies...")
    now_utc = datetime.datetime.now(datetime.timezone.utc)
    signal_time = now_utc

    live_candle_cache.clear()
    hist_candle_cache.clear()

    tasks = [process_asset(asset, signal_time) for asset in ASSETS]
    results = await _run_with_spinner(asyncio.gather(*tasks), "Processing assets")
    candidate_signals = [r for r in results if r]

    if not candidate_signals:
        log_warn("No assets processed this cycle.")
        return

    candidate_signals_sorted = sorted(candidate_signals, key=lambda x: x.get('confidence', 0.0), reverse=True)

    lines = []
    for res in candidate_signals_sorted:
        asset = res.get('asset', 'UNKNOWN')
        status = res.get('status', '—')
        direction = res.get('direction', '—')
        confidence = res.get('confidence', 0.0)
        wins = res.get('wins', 0)
        losses = res.get('losses', 0)
        strategies = ",".join(res.get('agreed_strategies', [])) if res.get('agreed_strategies') else ""
        lines.append(f"{asset} | {status} | {direction} | {confidence}% | BY ST:{wins} | {strategies}")

    print_box("Assets Summary (sorted by confidence)", lines)

    pass_items = [x for x in candidate_signals_sorted if x.get('status') == 'PASS']
    if not pass_items:
        log_warn("No PASS signals this cycle — nothing to pick.")
        return

    best_signal = pass_items[0]
    picked_line = f"{best_signal.get('asset')} -> {best_signal.get('direction')} | {best_signal.get('confidence')}% | W:{best_signal.get('wins',0)} L:{best_signal.get('losses',0)}"
    print_box("Picked Signal", [picked_line], title_color=BRIGHT_GREEN)

    sig_msg = format_signal(best_signal['asset'], datetime.datetime.now(datetime.timezone.utc), best_signal['direction'], TAG_USER_ID)
    print('\nSignal to send (telegram):\n')
    print(sig_msg)
    await send_telegram_with_optional_chart(sig_msg, best_signal.get('asset'), datetime.datetime.now(datetime.timezone.utc))
    await process_trade_for_signal(best_signal, datetime.datetime.now(datetime.timezone.utc))


def manual_partial_edit():
    """Run interactively when user types 'off' — allow editing specific signal entries (mark win/lose) and optionally send partials."""
    if not TRADE_HISTORY:
        print("No trade history to edit.")
        return False
    print_box("Current Trade History", [f"{i}: {t['time']} | {t['asset']} | {t['dir']} | {t['outcome']}" for i,t in enumerate(TRADE_HISTORY)])
    while True:
        choice = input("Do you want to edit any entry? (y/N): ").strip().lower()
        if choice != 'y':
            break
        idx_s = input("Enter index number of entry to edit (e.g., 0) or time (HH:MM): ").strip()
        idx = None
        if idx_s.isdigit():
            idx = int(idx_s)
            if idx < 0 or idx >= len(TRADE_HISTORY):
                print("Invalid index")
                continue
        else:
            # try find by time
            found = [i for i,t in enumerate(TRADE_HISTORY) if t.get('time') == idx_s]
            if not found:
                print("No entries found with that time")
                continue
            idx = found[0]
        new_out = input("Mark as WIN or LOSS? (WIN/LOSS): ").strip().upper()
        if new_out not in ('WIN','LOSS'):
            print("Invalid outcome")
            continue
        TRADE_HISTORY[idx]['outcome'] = new_out
        TRADE_HISTORY[idx]['manual'] = True
        print("Entry updated.")
    # After edits, ask to send partials
    sendp = input("Send edited partials now? (y/N): ").strip().lower() == 'y'
    if sendp:
        # send only edited entries
        for t in TRADE_HISTORY:
            if t.get('manual'):
                # build a small message
                msg = format_result(t['asset'], datetime.datetime.now(datetime.timezone.utc), t['dir'], 'WIN' if t['outcome']=='WIN' else 'LOSS', TAG_USER_ID, mtg_step=0)
                try:
                    # synchronous send in this thread
                    requests.post(f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage", data={"chat_id": TELEGRAM_CHAT_ID, "text": msg})
                except Exception as e:
                    print(f"Failed to send edited partial: {e}")
        return True
    return False
    
# =============== Main loop ===============
async def main_loop_best_signal():
    async def wait_until(next_dt: datetime.datetime):
        """Wait until next_dt (timezone-aware in TIMEZONE) but poll stdin every 1s for 'off'."""
        while True:
            now = datetime.datetime.now(TIMEZONE)
            remaining = (next_dt - now).total_seconds()
            if remaining <= 0:
                return True  # time reached
            # quick stdin check (non-blocking)
            try:
                if sys.stdin in select.select([sys.stdin], [], [], 0)[0]:
                    cmd = sys.stdin.readline().strip().lower()
                    if cmd == "off":
                        return False  # signal to stop
            except Exception:
                pass
            # sleep a short chunk so we stay responsive
            await asyncio.sleep(min(1.0, remaining))

    while True:
        # first: immediate stdin check before scheduling
        try:
            if sys.stdin in select.select([sys.stdin], [], [], 0)[0]:
                cmd = sys.stdin.readline().strip().lower()
                if cmd == "off":
                    # allow interactive manual edits before final summary/send
                    edited = await asyncio.to_thread(manual_partial_edit)
                    summary = format_summary()
                    print_box("Final Summary", summary.splitlines())
                    await send_telegram_with_optional_chart(summary, None, datetime.datetime.now(datetime.timezone.utc))
                    log_info("Shutting down by user command.")
                    break
        except Exception:
            pass

        # compute next exact minute boundary in local TIMEZONE (ceil to next minute unless exactly on :00)
        now_local = datetime.datetime.now(TIMEZONE)
        if now_local.second == 0 and now_local.microsecond == 0:
            # already aligned -> run immediately
            log_debug(f"Aligned now ({now_local.strftime('%H:%M:%S')}) — running scan now.")
            cont = True
        else:
            # ceil to next minute at :00
            next_min_local = (now_local + timedelta(minutes=1)).replace(second=0, microsecond=0)
            log_info(f"Waiting until next minute boundary {next_min_local.strftime('%Y-%m-%d %H:%M:%S')} to run scan...")
            cont = await wait_until(next_min_local)
        if not cont:
            # user asked 'off' while waiting
            edited = await asyncio.to_thread(manual_partial_edit)
            summary = format_summary()
            print_box("Final Summary", summary.splitlines())
            await send_telegram_with_optional_chart(summary, None, datetime.datetime.now(datetime.timezone.utc))
            log_info("Shutting down by user command.")
            break

        # At this point we are exactly at HH:MM:00 (local) — run the scan
        try:
            now = datetime.datetime.now(TIMEZONE)
            log_debug(f"Starting scan at {now.strftime('%Y-%m-%d %H:%M:%S')} (aligned).")
            await process_all_assets_once()
        except Exception as e:
            log_err(f"Exception during process_all_assets_once: {e}")

        # After processing, wait until the next exact minute boundary to start next cycle.
        # This ensures each cycle begins at HH:MM:00.
        now_after = datetime.datetime.now(TIMEZONE)
        next_min_after = (now_after + timedelta(minutes=1)).replace(second=0, microsecond=0)
        log_info(f"Cycle finished — sleeping until next run at {next_min_after.strftime('%Y-%m-%d %H:%M:%S')} ...")
        cont = await wait_until(next_min_after)
        if not cont:
            edited = await asyncio.to_thread(manual_partial_edit)
            summary = format_summary()
            print_box("Final Summary", summary.splitlines())
            await send_telegram_with_optional_chart(summary, None, datetime.datetime.now(datetime.timezone.utc))
            log_info("Shutting down by user command.")
            break

# =============== Interactive customization helpers ===============
def read_int(prompt: str, default: Optional[int] = None, allowed: Optional[List[int]] = None) -> int:
    while True:
        try:
            s = input(prompt).strip()
            if s == "" and default is not None:
                return default
            v = int(s)
            if allowed is not None and v not in allowed:
                print(f"Allowed values: {allowed}.")
                continue
            return v
        except Exception:
            print("Invalid integer, try again.")

def read_float(prompt: str, default: Optional[float] = None) -> float:
    while True:
        try:
            s = input(prompt).strip()
            if s == "" and default is not None:
                return default
            return float(s)
        except Exception:
            print("Invalid number, try again.")

def read_comma_ints(prompt: str, expected_count: Optional[int] = None, min_val: Optional[int]=None) -> List[int]:
    while True:
        s = input(prompt).strip()
        if not s:
            return []
        parts = [p.strip() for p in s.split(",") if p.strip()]
        try:
            vals = [int(p) for p in parts]
        except:
            print("Invalid integers, try again.")
            continue
        if expected_count is not None and len(vals) != expected_count:
            print(f"Expected exactly {expected_count} values (comma separated).")
            continue
        if min_val is not None and any(v < min_val for v in vals):
            print(f"Values must be >= {min_val}.")
            continue
        if sorted(vals) != vals:
            print("Values must be in ascending order (small -> large).")
            continue
        return vals

def customize_strategies_interactive(selected: List[str]):
    print("\n--- Strategy customization ---")
    print("If you skip (press Enter) default values will be used.")
    for s in selected:
        s_upper = s.upper()
        yn = input(f"Customize strategy '{s_upper}'? (y/N): ").strip().lower()
        if yn != 'y':
            continue

        # handle each strategy
        if s_upper == "EMA":
            mode = input("EMA mode: single or multiple? (single/multi) [single]: ").strip().lower() or "single"
            if mode == "single":
                v = read_int("Enter single EMA period (e.g., 20): ", default=20)
                STRAT_PARAMS["EMA"] = {"periods":[v]}
            else:
                vals = read_comma_ints("Enter exactly 4 EMA periods, comma separated (ascending) e.g. 10,20,50,100: ", expected_count=4, min_val=1)
                if not vals:
                    print("No valid EMA list provided; using default.")
                else:
                    STRAT_PARAMS["EMA"] = {"periods": vals}

        elif s_upper == "RSI":
            p = read_int(f"RSI period [{Config.RSI_PERIOD}]: ", default=Config.RSI_PERIOD)
            ob = read_int(f"RSI overbought [{Config.RSI_OVERBOUGHT}]: ", default=Config.RSI_OVERBOUGHT)
            os_ = read_int(f"RSI oversold [{Config.RSI_OVERSOLD}]: ", default=Config.RSI_OVERSOLD)
            STRAT_PARAMS["RSI"] = {"period": p, "overbought": ob, "oversold": os_}

        elif s_upper == "SR":
            lb = read_int(f"SR lookback [{Config.LOOKBACK}]: ", default=Config.LOOKBACK)
            wr = read_float(f"SR wick ratio [{Config.WICK_RATIO}]: ", default=Config.WICK_RATIO)
            STRAT_PARAMS["SR"] = {"lookback": lb, "wick_ratio": wr}

        elif s_upper == "MA":
            short = read_int("MA short period [9]: ", default=9)
            longp = read_int("MA long period [21]: ", default=21)
            STRAT_PARAMS["MA"] = {"short": short, "long": longp}

        elif s_upper == "FIB":
            print("FIB uses default 0.618 level — no extra params.")
            STRAT_PARAMS["FIB"] = {}

        elif s_upper == "TREND":
            length = read_int("TREND length (number of consecutive closed candles) [4]: ", default=4)
            STRAT_PARAMS["TREND"] = {"length": length}

        elif s_upper == "TCM":
            body_thresh = read_float("TCM body threshold multiplier [1.3]: ", default=1.3)
            wick_thresh = read_float("TCM last-wick threshold [0.6]: ", default=0.6)
            recent = read_int("TCM recent bodies count [10]: ", default=10)
            STRAT_PARAMS["TCM"] = {"body_thresh": body_thresh, "wick_thresh": wick_thresh, "recent_bodies_count": recent}

        elif s_upper == "WTS":
            wick_factor = read_float("WTS wick factor [1.5]: ", default=1.5)
            lookback = read_int("WTS lookback (candles) [10]: ", default=Config.LOOKBACK)
            STRAT_PARAMS["WTS"] = {"wick_factor": wick_factor, "lookback": lookback}

        elif s_upper == "PIS":
            lookback = read_int("PIS lookback (candles) [5]: ", default=5)
            STRAT_PARAMS["PIS"] = {"lookback": lookback}

        elif s_upper == "MACD":
            fast = read_int("MACD fast EMA period [12]: ", default=12)
            slow = read_int("MACD slow EMA period [26]: ", default=26)
            signal = read_int("MACD signal EMA period [9]: ", default=9)
            lookback = read_int("MACD lookback (candles) [8]: ", default=8)
            STRAT_PARAMS["MACD"] = {"fast": fast, "slow": slow, "signal": signal, "lookback": lookback}

        # ✅ EMAPULBACK customization (new, expanded)
        elif s_upper == "EMAPULBACK":
            print("\n--- Customize EMAPULBACK Strategy ---")
            current = STRAT_PARAMS.get("EMAPULBACK", {})

            # trend mode: single or multi
            mode = input(f"Trend EMA mode: single or multi? (single/multi) [{current.get('trend_mode','multi')}]: ").strip().lower() or current.get('trend_mode','multi')
            if mode not in ("single", "multi"):
                mode = "multi"

            if mode == "single":
                t = read_int(f"Single trend EMA period [{current.get('trend_ema_periods',[20])[0]}]: ", default=current.get('trend_ema_periods',[20])[0])
                trend_ema_periods = [t]
            else:
                vals = read_comma_ints(f"Enter 3-4 EMA periods for stacking, comma separated (ascending) [{','.join(map(str,current.get('trend_ema_periods',[20,50,100,200])))}]: ", expected_count=None, min_val=1)
                trend_ema_periods = vals or current.get('trend_ema_periods', [20,50,100,200])

            # thresholds and ATR
            touch_pct_threshold = read_float(f"Touch pct threshold (relative) [{current.get('touch_pct_threshold', 0.002)}]: ", default=current.get('touch_pct_threshold', 0.002))
            atr_lookback = read_int(f"ATR lookback [{current.get('atr_lookback', 10)}]: ", default=current.get('atr_lookback', 10))
            atr_multiplier = read_float(f"ATR multiplier [{current.get('atr_multiplier', 0.5)}]: ", default=current.get('atr_multiplier', 0.5))

            # slope and stacking
            min_slope_pct = read_float(f"Minimum EMA5 slope % [{current.get('min_slope_pct', 0.01)}]: ", default=current.get('min_slope_pct', 0.01))
            require_full_stack = input(f"Require full EMA stack? (y/N) [{'Y' if current.get('require_full_stack', True) else 'N'}]: ").strip().lower() == 'y'

            STRAT_PARAMS["EMAPULBACK"] = {
                "trend_mode": mode,
                "trend_ema_periods": trend_ema_periods,
                "touch_pct_threshold": touch_pct_threshold,
                "atr_lookback": atr_lookback,
                "atr_multiplier": atr_multiplier,
                "min_slope_pct": min_slope_pct,
                "require_full_stack": require_full_stack
            }
            print("EMAPULBACK settings updated!\n")

        # ✅ EMASLOPE customization (new strategy)
        elif s_upper in ("EMASLOPE", "EMA SLOP", "EMA_SLOP", "EMA SLOPE"):
            print("\n--- Customize EMASLOPE Strategy ---")
            current = STRAT_PARAMS.get("EMASLOPE", {})

            mode = input(f"Trend EMA mode for filter (single/multi) [{current.get('trend_mode','multi')}]: ").strip().lower() or current.get('trend_mode','multi')
            if mode not in ("single", "multi"):
                mode = "multi"

            if mode == "single":
                sp = read_int(f"Single trend EMA period [{current.get('trend_ema_periods',[20])[0]}]: ", default=current.get('trend_ema_periods',[20])[0])
                trend_ema_periods = [sp]
            else:
                vals = read_comma_ints(f"Enter 3-4 EMA periods for stacking, comma separated (ascending) [{','.join(map(str,current.get('trend_ema_periods',[20,50,100,200])))}]: ", expected_count=None, min_val=1)
                trend_ema_periods = vals or current.get('trend_ema_periods', [20,50,100,200])

            require_trend = input(f"Require detected trend? (y/N) [{'Y' if current.get('require_trend', True) else 'N'}]: ").strip().lower() == 'y'
            slope_min_pct = read_float(f"Minimum EMA5 slope % [{current.get('slope_min_pct', 0.01)}]: ", default=current.get('slope_min_pct', 0.01))
            engulfing_required = input(f"Require engulfing confirmation? (y/N) [{'Y' if current.get('engulfing_required', True) else 'N'}]: ").strip().lower() == 'y'
            engulfing_min_body_pct = read_float(f"Minimum engulfing body pct of price [{current.get('engulfing_min_body_pct', 0.15)}]: ", default=current.get('engulfing_min_body_pct', 0.15))

            STRAT_PARAMS["EMASLOPE"] = {
                "trend_mode": mode,
                "trend_ema_periods": trend_ema_periods,
                "require_trend": require_trend,
                "slope_min_pct": slope_min_pct,
                "engulfing_required": engulfing_required,
                "engulfing_min_body_pct": engulfing_min_body_pct
            }
            print("EMASLOPE settings updated!\n")
            
        # ✅ NEW: EMA200_SUPERTREND Strategy Customization
        elif s_upper in ("EMA200_SUPERTREND", "EMA200 SUPERTREND", "EMA200SUPER", "EMA200SUPERTREND"):
            print("\n--- Customize EMA200 + SuperTrend Strategy ---")
            current = STRAT_PARAMS.get("EMA200_SUPERTREND", {})

            ema200_period = read_int(
                f"EMA200 period [{current.get('ema200_period', 200)}]: ",
                default=current.get('ema200_period', 200)
            )

            supertrend_atr_period = read_int(
                f"SuperTrend ATR period [{current.get('supertrend_atr_period', 10)}]: ",
                default=current.get('supertrend_atr_period', 10)
            )

            supertrend_multiplier = read_float(
                f"SuperTrend ATR multiplier [{current.get('supertrend_multiplier', 3.0)}]: ",
                default=current.get('supertrend_multiplier', 3.0)
            )

            ema5_period = read_int(
                f"EMA slope (confirmation) EMA period [{current.get('ema5_period', 5)}]: ",
                default=current.get('ema5_period', 5)
            )

            ema5_slope_min = read_float(
                f"EMA5 minimum slope percent [{current.get('ema5_slope_min', 0.02)}]: ",
                default=current.get('ema5_slope_min', 0.02)
            )

            require_supertrend_trend = input(
                f"Require SuperTrend direction? (y/N) [{'Y' if current.get('require_supertrend_trend', True) else 'N'}]: "
            ).strip().lower() == 'y'

            require_slope_confirmation = input(
                f"Require EMA5 slope confirmation? (y/N) [{'Y' if current.get('require_slope_confirmation', True) else 'N'}]: "
            ).strip().lower() == 'y'

            STRAT_PARAMS["EMA200_SUPERTREND"] = {
                "ema200_period": ema200_period,
                "supertrend_atr_period": supertrend_atr_period,
                "supertrend_multiplier": supertrend_multiplier,
                "ema5_period": ema5_period,
                "ema5_slope_min": ema5_slope_min,
                "require_supertrend_trend": require_supertrend_trend,
                "require_slope_confirmation": require_slope_confirmation
            }

            print("EMA200_SUPERTREND settings updated!\n")    

        # ✅ ATR_SUPERT customization (added)
        elif s_upper == "ATR_SUPERT":
            print("\n--- Customize ATR_SUPERT Strategy ---")
            current = STRAT_PARAMS.get("ATR_SUPERT", {})

            atr_period = read_int(f"ATR period [{current.get('atr_period', 10)}]: ", default=current.get('atr_period', 10))
            atr_multiplier = read_float(f"ATR multiplier [{current.get('atr_multiplier', 3.0)}]: ", default=current.get('atr_multiplier', 3.0))
            require_reversal = input(f"Require reversal for signals? (y/N) [{'Y' if current.get('require_reversal', True) else 'N'}]: ").strip().lower() == 'y'
            htf_factor = read_int(f"HTF aggregation factor (1 = same TF, 2 = 2x) [{current.get('htf_factor', 1)}]: ", default=current.get('htf_factor', 1))
            use_htf_as_filter = input(f"Use HTF EMA200 as filter? (y/N) [{'Y' if current.get('use_htf_as_filter', False) else 'N'}]: ").strip().lower() == 'y'

            # EMA filter options
            print("\nEMA Filter (optional): choose how EMA filter should behave.")
            ema_filter_mode = input(f"EMA filter mode (none/single/multi) [{current.get('ema_filter_mode','none')}]: ").strip().lower() or current.get('ema_filter_mode','none')
            if ema_filter_mode not in ('none','single','multi'):
                ema_filter_mode = 'none'
            ema_periods = []
            if ema_filter_mode == 'single':
                p = read_int(f"Single EMA period for filter [{(current.get('ema_filter_periods',[20])[0])}]: ", default=(current.get('ema_filter_periods',[20])[0]))
                ema_periods = [p]
            elif ema_filter_mode == 'multi':
                vals = read_comma_ints(f"Enter EMA periods for multi-filter (ascending) e.g. 10,20,50 [{','.join(map(str,current.get('ema_filter_periods',[10,20,50])))}]: ", expected_count=None, min_val=1)
                ema_periods = vals or current.get('ema_filter_periods', [10,20,50])

            ema_filter_require = input(f"Enforce EMA filter (if set) as a hard requirement? (y/N) [{'Y' if current.get('ema_filter_require', False) else 'N'}]: ").strip().lower() == 'y'

            STRAT_PARAMS["ATR_SUPERT"] = {
                "atr_period": atr_period,
                "atr_multiplier": atr_multiplier,
                "require_reversal": require_reversal,
                "htf_factor": htf_factor,
                "use_htf_as_filter": use_htf_as_filter,
                "ema_filter_mode": ema_filter_mode,
                "ema_filter_periods": ema_periods,
                "ema_filter_require": ema_filter_require,
                "min_dist_atr_mult": current.get("min_dist_atr_mult", 0.15)
            }
            print("ATR_SUPERT settings updated!\n")

        else:
            print(f"No custom UI for {s_upper} — skipping.")



# =============== Entry & startup UI ===============
# ====================== ENTRY & STARTUP UI =======================
# ====================== ENTRY & STARTUP UI =======================
if __name__ == "__main__":
    try:
        import math
        import textwrap

        clear_screen()
        big_banner('SHADOW PRO')
        print(Style.BRIGHT + Fore.CYAN + "🛠️  SOFTWARE DEVELOPER TOWSIF_DEV 𖣘 AUTO SIGNAL GENERATOR (customizable)\n" + RESET)

        print(Style.BRIGHT + Fore.YELLOW + "============== TELEGRAM CONFIGURATION SETTING ==============" + RESET)

        TELEGRAM_BOT_TOKEN = input(Style.BRIGHT + Fore.WHITE + "\n🤖 ENTER YOUR TELEGRAM BOT TOKEN: " + RESET).strip()
        TELEGRAM_CHAT_ID = input(Style.BRIGHT + Fore.WHITE + "\n💬 ENTER YOUR TELEGRAM CHAT ID: " + RESET).strip()
        TAG_USER_ID = input(Style.BRIGHT + Fore.WHITE + "\n🏷️ ENTER TAG USER ID (@USERNAME OR NUMERIC ID): " + RESET).strip()

        print(Style.BRIGHT + Fore.GREEN + "\n======================= INPUT RECEIVED =======================\n" + RESET)

        # ----------------- New interactive strategy & pair selection UI -----------------
        ALL_STRATEGIES = ["EMA","RSI","SR","MA","FIB","TREND","TCM","WTS","PIS","SRF","MACD","EMAPULBACK","EMASLOPE","ATR_SUPERT","EMA200_SUPERTREND"]

        ALL_PAIRS_RAW = [
            "ADAUSD_otc","APTUSD_otc","ARBUSD_otc","ATOUSD_otc","AUDCAD","AUDCHF","AUDJPY","AUDNZD_otc","AUDUSD","AVAUSD_otc","AXJAUD",
            "AXP_otc","AXSUSD_otc","BA_otc","BCHUSD_otc","BEAUSD_otc","BNBUSD_otc","BONUSD_otc","BRLUSD_otc","BTCUSD_otc","CADCHF_otc",
            "CADJPY","CHFJPY","CHIA50","DASUSD_otc","DJIUSD","DOGUSD_otc","DOTUSD_otc","ETCUSD_otc","ETHUSD_otc","EURAUD","EURCAD",
            "EURCHF","EURGBP","EURJPY","EURNZD_otc","EURSGD_otc","EURUSD","F40EUR","FB_otc","FLOUSD_otc","FTSGBP","GALUSD_otc","GBPAUD",
            "GBPCAD","GBPCHF","GBPJPY","GBPNZD_otc","GBPUSD","HMSUSD_otc","HSIHKD","IBXEUR","INTC_otc","JNJ_otc","JPXJPY","LINUSD_otc",
            "LTCUSD_otc","MANUSD_otc","MCD_otc","MELUSD_otc","MSFT_otc","NDXUSD","NZDCAD_otc","NZDCHF_otc","NZDJPY_otc","PEPUSD_otc",
            "PFE_otc","SHIUSD_otc","SOLUSD_otc","STXEUR","TIAUSD_otc","TONUSD_otc","TRUUSD_otc","UKBrent_otc","USCrude_otc","USDARS_otc",
            "USDBDT_otc","USDCAD","USDCHF","USDCOP_otc","USDDZD_otc","USDEGP_otc","USDIDR_otc","USDINR_otc","USDJPY","USDMXN_otc",
            "USDNGN_otc","USDPHP_otc","USDPKR_otc","USDTRY_otc","USDZAR_otc","WIFUSD_otc","XAGUSD_otc","XAUUSD_otc","XRPUSD_otc","ZECUSD_otc"
        ]

        # Keep only OTC pairs (those containing '_otc'), per your request
        VALID_PAIRS = [p for p in ALL_PAIRS_RAW if "_otc" in p.lower()]

        # --- single big-box 7-column grid printer ---
        def print_single_box_grid(items, cols=7, col_width=20, title=None):
            if not items:
                print(Style.BRIGHT + Fore.YELLOW + "(no items to display)" + RESET)
                return
            total = len(items)
            rows = math.ceil(total / cols)
            # build 2D grid
            grid = [["" for _ in range(cols)] for _ in range(rows)]
            for idx, item in enumerate(items, start=1):
                r = (idx - 1) // cols
                c = (idx - 1) % cols
                grid[r][c] = f"{idx}. {item}"

            inner_w = cols * col_width + (cols - 1) * 1  # 1 space gap
            top_border = "┌" + "─" * inner_w + "┐"
            bottom_border = "└" + "─" * inner_w + "┘"

            if title:
                # print title above box centered
                print(Style.BRIGHT + Fore.YELLOW + f" {title} " + RESET)

            print(Style.BRIGHT + Fore.CYAN + top_border + RESET)
            for r in range(rows):
                line = "│"
                for c in range(cols):
                    cell = grid[r][c]
                    if len(cell) > col_width:
                        cell = cell[:col_width-3] + "..."
                    cell = cell.center(col_width)
                    line += cell
                    if c < cols - 1:
                        line += " "
                line += "│"
                print(Style.BRIGHT + Fore.WHITE + line + RESET)
            print(Style.BRIGHT + Fore.CYAN + bottom_border + RESET)
            print()

        # --- selection parsing (numbers, ranges, names) ---
        def parse_selection_input(selection, items):
            selection = (selection or "").strip()
            if not selection:
                return []
            parts = [p.strip() for p in selection.split(",") if p.strip()]
            picked = []
            for part in parts:
                if part.isdigit():
                    idx = int(part) - 1
                    if 0 <= idx < len(items):
                        val = items[idx]
                        if val not in picked:
                            picked.append(val)
                elif "-" in part:
                    try:
                        a, b = part.split("-", 1)
                        a_i = int(a.strip()) - 1
                        b_i = int(b.strip()) - 1
                        if a_i > b_i:
                            a_i, b_i = b_i, a_i
                        for idx in range(a_i, b_i+1):
                            if 0 <= idx < len(items):
                                val = items[idx]
                                if val not in picked:
                                    picked.append(val)
                    except Exception:
                        pass
                else:
                    found = None
                    for it in items:
                        if it.lower() == part.lower():
                            found = it
                            break
                    if not found:
                        for it in items:
                            if part.lower() in it.lower():
                                found = it
                                break
                    if found and found not in picked:
                        picked.append(found)
            return picked

        # --- strategies selection (keeps boxed 2-column list for clarity) ---
        def print_boxed_grid(items, cols=2, box_width=28, title=None):
            if title:
                print(Style.BRIGHT + Fore.YELLOW + title + RESET)
            n = len(items)
            rows = math.ceil(n / cols)
            boxes = []
            for idx, item in enumerate(items, start=1):
                header = f"{idx}. {item}"
                wrapped = textwrap.fill(header, box_width - 4)
                lines = wrapped.splitlines()
                boxes.append(lines)
            for r in range(rows):
                row_boxes = boxes[r*cols:(r+1)*cols]
                if not row_boxes:
                    continue
                max_lines = max((len(b) for b in row_boxes), default=1)
                border_line = ""
                for _ in row_boxes:
                    border_line += "┌" + "─"*(box_width-2) + "┐" + "  "
                print(Style.BRIGHT + Fore.CYAN + border_line + RESET)
                for i in range(max_lines):
                    line = ""
                    for b in row_boxes:
                        text = b[i] if i < len(b) else ""
                        text = text.center(box_width-2)
                        line += "│" + text + "│" + "  "
                    print(Style.BRIGHT + Fore.WHITE + line + RESET)
                bottom_line = ""
                for _ in row_boxes:
                    bottom_line += "└" + "─"*(box_width-2) + "┘" + "  "
                print(Style.BRIGHT + Fore.CYAN + bottom_line + RESET)
            print()

        def select_strategies_interactive(default_list=ALL_STRATEGIES):
            print()
            print(Style.BRIGHT + Fore.GREEN + "👉 Strategy selection: enter numbers (e.g. 1,3), ranges (2-5) or names (EMA,RSI). Press Enter to select ALL." + RESET)
            print_boxed_grid(default_list, cols=2, box_width=28, title="STRATEGIES")
            sel = input(Style.BRIGHT + Fore.WHITE + "Select strategies (numbers or names, ENTER=ALL): " + RESET).strip()
            if not sel:
                return default_list.copy()
            chosen = parse_selection_input(sel, default_list)
            if not chosen:
                print(Style.BRIGHT + Fore.RED + "No valid strategy selected — defaulting to ALL." + RESET)
                return default_list.copy()
            print(Style.BRIGHT + Fore.GREEN + f"Selected strategies: {', '.join(chosen)}" + RESET)
            return chosen

        # --- pairs selection using single big box (7 columns) ---
        def select_pairs_interactive(valid_pairs=VALID_PAIRS):
            print()
            print(Style.BRIGHT + Fore.YELLOW + "Pair input options:" + RESET)
            print(Style.BRIGHT + Fore.WHITE + " - Press 'L' to LIST known OTC pairs inside ONE big box (7 columns) and pick by number/name." + RESET)
            print(Style.BRIGHT + Fore.WHITE + " - Or press 'M' to manually type comma-separated pair names (we'll validate them)." + RESET)
            choice = input(Style.BRIGHT + Fore.WHITE + "Choose [L/M] (default M): " + RESET).strip().lower() or "m"

            if choice == 'l':
                print_single_box_grid(valid_pairs, cols=7, col_width=20, title="VALID OTC PAIRS")
                sel = input(Style.BRIGHT + Fore.WHITE + "Select pairs by numbers/names (e.g. 1,3-5,BTCUSD_otc) or ENTER to cancel: " + RESET).strip()
                if not sel:
                    print(Style.BRIGHT + Fore.YELLOW + "No selection made. Returning empty list." + RESET)
                    return []
                chosen = parse_selection_input(sel, valid_pairs)
                if not chosen:
                    print(Style.BRIGHT + Fore.RED + "No valid pair selected." + RESET)
                    return []
                print(Style.BRIGHT + Fore.GREEN + f"Selected pairs: {', '.join(chosen)}" + RESET)
                return chosen

            else:
                manual = input(Style.BRIGHT + Fore.WHITE + "Enter comma-separated pairs (e.g. ADAUSD_otc, BTCUSD_otc): " + RESET).strip()
                if not manual:
                    print(Style.BRIGHT + Fore.YELLOW + "No pairs entered. Returning empty list." + RESET)
                    return []
                entered = [p.strip() for p in manual.split(",") if p.strip()]
                matched = []
                for ent in entered:
                    found = None
                    for vp in valid_pairs:
                        if ent.lower() == vp.lower():
                            found = vp
                            break
                    if not found:
                        candidates = [vp for vp in valid_pairs if ent.lower() in vp.lower()]
                        if len(candidates) == 1:
                            found = candidates[0]
                        elif len(candidates) > 1:
                            for c in candidates:
                                if c.lower().startswith(ent.lower()):
                                    found = c
                                    break
                    if found and found not in matched:
                        matched.append(found)
                if not matched:
                    print(Style.BRIGHT + Fore.RED + "None of the entered pairs matched known OTC pairs. Nothing selected." + RESET)
                    return []
                print(Style.BRIGHT + Fore.CYAN + "Matched OTC pairs:" + RESET)
                for p in matched:
                    print("  " + Style.BRIGHT + Fore.GREEN + "✔️ " + p + RESET)
                ok = input(Style.BRIGHT + Fore.WHITE + "Confirm using these pairs? (Y/N): " + RESET).strip().lower() == 'y'
                if ok:
                    return matched
                else:
                    print(Style.BRIGHT + Fore.YELLOW + "Selection canceled. Returning empty list." + RESET)
                    return []

        # ---------- Use the new interactive selection (replaces previous pairs_input & simple strategy_input) ----------
        print(Style.BRIGHT + Fore.YELLOW + "============== SIGNAL CONFIGURATION SETTING ==============\n" + RESET)

        # Strategy: allow interactive numbered/grid selection, or fallback to manual comma input
        strategy_choice_mode = input(Style.BRIGHT + Fore.WHITE + "\nDo you want to SELECT strategies from a boxed list? (Y/N) [Y]: " + RESET).strip().lower() or 'y'
        if strategy_choice_mode == 'y':
            SELECTED_STRATS = select_strategies_interactive(ALL_STRATEGIES)
        else:
            strategy_input = input(
                Style.BRIGHT + Fore.WHITE +
                "\n🧭 CHOOSE STRATEGIES (COMMA SEPARATED) [EMA,RSI,SR,MA,FIB,TREND,TCM,WTS,PIS,SRF,MACD,EMAPULBACK,EMASLOPE,ATR_SUPERT] (DEFAULT ALL): "
                + RESET
            ).strip()
            if not strategy_input:
                SELECTED_STRATS = ALL_STRATEGIES.copy()
            else:
                SELECTED_STRATS = [s.strip().upper() for s in strategy_input.split(",") if s.strip()]

        customize_choice = input(
            Style.BRIGHT + Fore.WHITE + "\n⚙️ DO YOU WANT TO CUSTOMIZE SELECTED STRATEGIES? (Y/N): " + RESET
        ).strip().lower()

        if customize_choice == 'y':
            print(Style.BRIGHT + Fore.CYAN + "\n--- STRATEGY CUSTOMIZATION ---\nIF YOU SKIP (PRESS ENTER) DEFAULT VALUES WILL BE USED.\n" + RESET)
            try:
                customize_strategies_interactive(SELECTED_STRATS)
            except Exception:
                print(Style.BRIGHT + Fore.RED + "Customization function not available or failed — skipping customization." + RESET)
        else:
            print(Style.BRIGHT + Fore.GREEN + "✅ USING DEFAULT STRATEGY SETTINGS.\n" + RESET)

        # Pair selection: interactive (single big box 7-column display)
        print(Style.BRIGHT + Fore.YELLOW + "\n============== PAIR CONFIGURATION ==============\n" + RESET)
        ASSETS = select_pairs_interactive(VALID_PAIRS)

        # fallback behavior if none selected
        if not ASSETS:
            print(Style.BRIGHT + Fore.YELLOW + "No valid OTC pairs selected — giving one more chance to input manually." + RESET)
            ASSETS = select_pairs_interactive(VALID_PAIRS)
            if not ASSETS:
                print(Style.BRIGHT + Fore.RED + "Still no pairs selected — defaulting to first 10 valid OTC pairs." + RESET)
                ASSETS = VALID_PAIRS[:10]

        # ----------------- rest of original config inputs (unchanged) -----------------
        try:
            BACKTEST_DAYS = int(input(Style.BRIGHT + Fore.WHITE + "🔢 ENTER DAY FILTERS (E.G., 3): " + RESET).strip() or "3")
        except Exception:
            BACKTEST_DAYS = 3

        try:
            ms_signal = int(input(Style.BRIGHT + Fore.WHITE + "🔢 MARTINGALE FOR SIGNALS (0=EXACT ONLY, 1=EXACT+1, 2=EXACT+1+2) [1]: " + RESET).strip() or "1")
            if ms_signal not in (0,1,2):
                print(Style.BRIGHT + Fore.RED + "ONLY 0,1,2 ALLOWED FOR SIGNAL MTG. DEFAULTING TO 1." + RESET)
                ms_signal = 1
        except Exception:
            ms_signal = 1
        MARTINGALE_STEPS_SIGNAL = ms_signal

        try:
            ms_backtest = int(input(Style.BRIGHT + Fore.WHITE + "🔢 MARTINGALE FOR FILTER (0=EXACT ONLY, 1=EXACT+1, 2=EXACT+1+2) [1]: " + RESET).strip() or "1")
            if ms_backtest not in (0,1,2):
                print(Style.BRIGHT + Fore.RED + "ONLY 0,1,2 ALLOWED FOR BACKTEST MTG. DEFAULTING TO 1." + RESET)
                ms_backtest = 1
        except Exception:
            ms_backtest = 1
        MARTINGALE_STEPS_BACKTEST = ms_backtest

        SKIP_RECENT = input(Style.BRIGHT + Fore.WHITE + "🔁 SKIP THE MOST RECENT PRIOR (Y/N)? " + RESET).strip().lower() == 'y'
        print(Style.BRIGHT + Fore.GREEN + "\n======================= INPUT RECEIVED =======================\n" + RESET)
        send_charts_input = input(Style.BRIGHT + Fore.YELLOW + "📷 ATTACH CHART SCREENSHOTS TO TELEGRAM MESSAGES? (Y/N): " + RESET).strip().lower()
        SEND_CHARTS = (send_charts_input == 'y')

        safety_choice = input(Style.BRIGHT + Fore.YELLOW + "🔒 ENABLE SAFETY-MARGIN FOR LIVE SIGNALS? (Y/N): " + RESET).strip().lower()
        USE_SAFETY_MARGIN = (safety_choice == 'y')
        if USE_SAFETY_MARGIN:
            try:
                wv = float(input(Style.BRIGHT + Fore.WHITE + "Safety wick threshold (0.0-1.0) [0.80]: " + RESET).strip() or "0.80")
                bv = float(input(Style.BRIGHT + Fore.WHITE + "Safety body max ratio (0.0-1.0) [0.20]: " + RESET).strip() or "0.20")
                SAFETY_WICK_THRESHOLD = max(0.0, min(1.0, wv))
                SAFETY_BODY_THRESHOLD = max(0.0, min(1.0, bv))
            except Exception:
                SAFETY_WICK_THRESHOLD = 0.80
                SAFETY_BODY_THRESHOLD = 0.20

        SAVE_CSV = False
        DEBUG = True

        clear_screen()
        big_banner('SHADOW PRO')
        print(Style.BRIGHT + Fore.CYAN + "🛠️  SOFTWARE DEVELOPER TOWSIF_DEV 𖣘 AUTO SIGNAL GENERATOR\n" + RESET)
        
        show_loading_spinner(1.0)

        def build_input_summary():
            token_mask = TELEGRAM_BOT_TOKEN
            if token_mask and len(token_mask) > 8:
                token_mask = token_mask[:6] + "..." + token_mask[-4:]

            return [
                f"Bot Token        : {token_mask}",
                f"Chat ID          : {TELEGRAM_CHAT_ID}",
                f"Tag User         : {TAG_USER_ID}",
                f"Strategies       : {', '.join(SELECTED_STRATS)}",
                f"Backtest Days    : {BACKTEST_DAYS}",
                f"MTG (Signal)     : {MARTINGALE_STEPS_SIGNAL}",
                f"MTG (Backtest)   : {MARTINGALE_STEPS_BACKTEST}",
                f"Skip Recent Day  : {'Yes' if SKIP_RECENT else 'No'}",
                f"Send Charts      : {'Yes' if SEND_CHARTS else 'No'}",
                f"Use SafetyMargin : {'Yes' if USE_SAFETY_MARGIN else 'No'}",
            ]

        print(Style.BRIGHT + Fore.YELLOW + "📋 CONFIGURATION SUMMARY" + RESET)
        print(Style.BRIGHT + Fore.GREEN + "━" * 35 + RESET)

        for line in build_input_summary():
            typing_print("  " + line)
        print(Style.BRIGHT + Fore.GREEN + "━" * 35 + RESET)
        
        print(Fore.GREEN + "✅ Configuration locked. Starting engine...\n" + RESET)

        print(Style.BRIGHT + Fore.CYAN + "┏============== AI AUTO BOT SIGNAL TERMINAL ==============┓" + RESET)

        asyncio.run(main_loop_best_signal())

    except KeyboardInterrupt:
        print('\n')
        print(Style.BRIGHT + Fore.YELLOW + "KEYBOARD INTERRUPT RECEIVED — ATTEMPTING GRACEFUL SHUTDOWN..." + RESET)
        try:
            loop = asyncio.new_event_loop()
            asyncio.set_event_loop(loop)
            summary = format_summary()
            print_box("Final Summary", summary.splitlines())
            loop.run_until_complete(send_telegram_with_optional_chart(summary, None, datetime.datetime.now(datetime.timezone.utc)))
        except Exception as e:
            print(Style.BRIGHT + Fore.RED + f"COULD NOT SEND SUMMARY ON EXIT: {e}" + RESET)
        finally:
            sys.exit(0)
