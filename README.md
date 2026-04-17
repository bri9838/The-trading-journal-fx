import streamlit as st
import pandas as pd
import numpy as np
import os
from datetime import datetime
import streamlit.components.v1 as components

# --- STEP 1: SMART IMPORT ---
try:
    import MetaTrader5 as mt5
    MT5_AVAILABLE = True
except ImportError:
    MT5_AVAILABLE = False

# --- STEP 2: UI & BRANDING (High-Visibility Neon Theme) ---
st.set_page_config(page_title="The Global Trading Book", layout="wide", initial_sidebar_state="collapsed")

st.markdown("""
    <style>
    /* Global Background & Text */
    .stApp { background-color: #0e1117; color: #e0e0e0; }
    
    /* Neon Glow Metrics */
    div[data-testid="stMetricValue"] { 
        font-size: 32px !important; 
        color: #00ffd0 !important; 
        text-shadow: 0 0 15px rgba(0, 255, 208, 0.4);
        font-weight: 800;
    }
    
    /* Input Fields Visibility */
    .stTextInput>div>div>input, .stNumberInput>div>div>input {
        background-color: #1a1c24 !important;
        color: #ffffff !important;
        border: 1.5px solid #4facfe !important;
        border-radius: 10px;
    }

    /* Tabs Styling */
    .stTabs [aria-selected="true"] { 
        background: linear-gradient(90deg, #4facfe, #00f2fe) !important; 
        color: black !important;
        font-weight: bold;
        border-radius: 10px;
    }

    /* Place Order Button (Gradient) */
    .stButton > button { 
        width: 100%; 
        border-radius: 12px; 
        font-weight: bold; 
        height: 3.5em; 
        background: linear-gradient(45deg, #00dbde, #fc00ff) !important; 
        color: white !important;
        border: none;
        box-shadow: 0 4px 15px rgba(0, 0, 0, 0.3);
    }
    
    h1 { 
        text-align: center; 
        color: #ffffff;
        text-shadow: 0 0 20px rgba(79, 172, 254, 0.6);
    }
    </style>
    """, unsafe_allow_html=True)

# --- STEP 3: EXECUTION LOGIC ---
def place_tgtb_order(symbol, lot, side, order_type, price=None):
    if not MT5_AVAILABLE or not mt5.initialize():
        return None
        
    order_dict = {
        "Market": mt5.ORDER_TYPE_BUY if side == "BUY" else mt5.ORDER_TYPE_SELL,
        "Limit": mt5.ORDER_TYPE_BUY_LIMIT if side == "BUY" else mt5.ORDER_TYPE_SELL_LIMIT,
        "Stop": mt5.ORDER_TYPE_BUY_STOP if side == "BUY" else mt5.ORDER_TYPE_SELL_STOP
    }
    
    tick = mt5.symbol_info_tick(symbol)
    if tick is None: return None
        
    p = price if order_type != "Market" else (tick.ask if side == "BUY" else tick.bid)
    
    request = {
        "action": mt5.TRADE_ACTION_DEAL if order_type == "Market" else mt5.TRADE_ACTION_PENDING,
        "symbol": symbol, "volume": lot, "type": order_dict[order_type], "price": p,
        "magic": 786, "comment": "TGTB Pro", "type_time": mt5.ORDER_TIME_GTC, "type_filling": mt5.ORDER_FILLING_IOC
    }
    return mt5.order_send(request)

# --- STEP 4: DATABASE ---
DATA_FILE = "global_trading_book_data.csv"
if not os.path.exists(DATA_FILE):
    pd.DataFrame(columns=["Date", "Symbol", "Side", "Type", "Price", "Qty", "PnL", "Mood", "Notes"]).to_csv(DATA_FILE, index=False)

df = pd.read_csv(DATA_FILE)

# --- STEP 5: MAIN APP ---
st.markdown("<h1>📔 THE GLOBAL TRADING BOOK</h1>", unsafe_allow_html=True)

# Sidebar with Drawdown Tracker
with st.sidebar:
    st.header("⚙️ Risk Status")
    
    if not df.empty:
        cumulative_pnl = df['PnL'].cumsum()
        peak = cumulative_pnl.max()
        current_pnl = cumulative_pnl.iloc[-1]
        drawdown_val = peak - current_pnl if current_pnl < peak else 0.0
        drawdown_pct = (drawdown_val / abs(peak) * 100) if peak != 0 else 0.0
        st.metric("Current Drawdown", f"${drawdown_val:,.2f}", f"-{drawdown_pct:.2f}%", delta_color="inverse")

    if MT5_AVAILABLE:
        st.success("MT5 Ready 🟢")
        if st.button("Reconnect MT5"): mt5.initialize()
    
    daily_limit = st.number_input("Daily Loss Limit ($)", value=500.0)

tabs = st.tabs(["⚡ TRADE", "📈 STATS", "🧠 AI REVIEW", "🚨 EMERGENCY"])

with tabs[0]: # TRADE TAB
    col_chart, col_form = st.columns([3, 1])
    
    with col_chart:
        sym = st.text_input("Symbol", value="XAUUSD").upper()
        # Full Size Chart (650px)
        chart_html = f'<div style="height:650px;"><div id="tv"></div><script src="https://s3.tradingview.com/tv.js"></script><script>new TradingView.widget({{"autosize":true,"symbol":"{sym}","interval":"15","theme":"dark","style":"1","container_id":"tv","hide_side_toolbar":false,"allow_symbol_change":true}});</script></div>'
        components.html(chart_html, height=650)
    
    with col_form:
        st.subheader("Quick Entry")
        
        # Quick Lot Buttons Logic
        if 'trade_lots' not in st.session_state: st.session_state['trade_lots'] = 0.01
        
        q_cols = st.columns(3)
        if q_cols[0].button("0.01"): st.session_state['trade_lots'] = 0.01
        if q_cols[1].button("0.10"): st.session_state['trade_lots'] = 0.10
        if q_cols[2].button("1.00"): st.session_state['trade_lots'] = 1.00

        with st.form("trade_form", clear_on_submit=True):
            ot = st.selectbox("Order Type", ["Market", "Limit", "Stop"])
            sd = st.radio("Side", ["BUY", "SELL"], horizontal=True)
            lt = st.number_input("Lots", value=st.session_state['trade_lots'], step=0.01)
            pr = st.number_input("Price", format="%.5f") if ot != "Market" else 0.0
            md = st.selectbox("Psychology", ["Disciplined", "FOMO", "Impulsive", "Revenge"])
            nt = st.text_input("Notes")
            
            if st.form_submit_button("PLACE & LOG"):
                exec_price = pr
                if MT5_AVAILABLE:
                    res = place_tgtb_order(sym, lt, sd, ot, pr)
                    if res and res.retcode == mt5.TRADE_RETCODE_DONE:
                        st.success("Trade Executed!")
                        exec_price = res.price
                
                new_trade = [datetime.now().strftime("%Y-%m-%d %H:%M"), sym, sd, ot, exec_price, lt, 0, md, nt]
                pd.DataFrame([new_trade]).to_csv(DATA_FILE, mode='a', header=False, index=False)
                st.rerun()

with tabs[1]: # STATS TAB
    if not df.empty:
        st.metric("Total P&L", f"${df['PnL'].sum():,.2f}")
        st.line_chart(df['PnL'].cumsum())

with tabs[2]: # AI REVIEW TAB
    st.header("🧠 AI Trade Analysis")
    if df.empty:
        st.info("AI Analysis ke liye trades log karein.")
    else:
        revenge_loss = df[df['Mood'] == 'Revenge']['PnL'].sum()
        fomo_count = len(df[df['Mood'] == 'FOMO'])
        
        st.subheader("AI Coach Insights")
        if revenge_loss < 0:
            st.error(f"⚠️ Alert: Revenge trading se ${abs(revenge_loss)} ka loss hua. Stop and Breath!")
        if fomo_count > 0:
            st.warning(f"📉 Insight: Aapne {fomo_count} trades FOMO mein liye hain. Be patient.")
            
        st.subheader("Journal History")
        st.dataframe(df.style.applymap(lambda x: 'background-color: #1b4d3e' if x > 0 else 'background-color: #4d1b1b', subset=['PnL']), use_container_width=True)

with tabs[3]: # EMERGENCY TAB
    st.header("🛑 Kill Switch")
    if st.button("CLOSE ALL POSITIONS"):
        st.warning("Action triggered. (MT5 connection required)")
