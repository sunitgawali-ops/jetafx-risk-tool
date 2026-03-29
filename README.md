import io
import re
import pandas as pd
import streamlit as st

st.set_page_config(page_title="Toxic Flow Detector", layout="wide")
st.title("Forex Toxic Flow Detector")
st.write("Upload MT5 trade history. The app will automatically read only the Positions table.")

def make_unique_columns(columns):
    counts = {}
    new_cols = []
    for col in columns:
        col_name = "" if pd.isna(col) else str(col).strip()
        if col_name == "":
            col_name = "Blank"

        if col_name in counts:
            counts[col_name] += 1
            new_cols.append(f"{col_name}_{counts[col_name]}")
        else:
            counts[col_name] = 1
            new_cols.append(col_name)
    return new_cols

def extract_account_number(raw_df):
    for i in range(len(raw_df)):
        row_vals = [str(x).strip() for x in raw_df.iloc[i].tolist() if pd.notna(x)]
        joined = " | ".join(row_vals)
        if "Account:" in joined:
            # Example: 208444 (USD, 1:100, JetaFX...)
            match = re.search(r"Account:\s*.*?(\d{4,})", joined)
            if match:
                return match.group(1)
            # fallback: find first 4+ digit number anywhere in the row
            match = re.search(r"(\d{4,})", joined)
            if match:
                return match.group(1)
    return "Unknown"

def extract_positions_table(raw_df):
    positions_row = None

    for i in range(len(raw_df)):
        row_values = raw_df.iloc[i].astype(str).str.strip().str.lower().tolist()
        if any(v == "positions" for v in row_values):
            positions_row = i
            break

    if positions_row is None:
        raise ValueError("Could not find 'Positions' section in the uploaded file.")

    header_row = positions_row + 1
    raw_headers = raw_df.iloc[header_row].tolist()
    headers = make_unique_columns(raw_headers)

    data_start = header_row + 1
    df_pos = raw_df.iloc[data_start:].copy()
    df_pos.columns = headers

    stop_idx = None
    for i in range(len(df_pos)):
        row_vals = df_pos.iloc[i].astype(str).str.strip().str.lower().tolist()
        if any(v in ["orders", "deals"] for v in row_vals):
            stop_idx = i
            break

    if stop_idx is not None:
        df_pos = df_pos.iloc[:stop_idx]

    df_pos = df_pos.dropna(how="all").reset_index(drop=True)

    keep_cols = []
    for c in df_pos.columns:
        col_data = df_pos[c]
        if col_data.notna().sum() == 0:
            continue
        if str(c).lower().startswith("blank") and col_data.astype(str).str.strip().replace("None", "").eq("").all():
            continue
        keep_cols.append(c)

    df_pos = df_pos[keep_cols].copy()
    return df_pos

def to_excel_bytes(df_dict):
    output = io.BytesIO()
    with pd.ExcelWriter(output, engine="openpyxl") as writer:
        for sheet_name, sheet_df in df_dict.items():
            sheet_df.to_excel(writer, index=False, sheet_name=sheet_name[:31])
    output.seek(0)
    return output.getvalue()

uploaded_file = st.file_uploader("Upload Excel or CSV file", type=["xlsx", "xls", "csv"])

if uploaded_file is not None:
    try:
        if uploaded_file.name.endswith(".csv"):
            df_raw = pd.read_csv(uploaded_file, header=None)
            account_number = "Unknown"
        else:
            xls = pd.ExcelFile(uploaded_file)
            sheet_name = st.selectbox("Select Sheet", xls.sheet_names)
            df_raw = pd.read_excel(uploaded_file, sheet_name=sheet_name, header=None)
            account_number = extract_account_number(df_raw)

        st.subheader("Raw Sheet Preview")
        st.dataframe(df_raw.head(15), use_container_width=True)

        df = extract_positions_table(df_raw)

        st.subheader("Detected Positions Table")
        st.dataframe(df.head(20), use_container_width=True)

    except Exception as e:
        st.error(f"Error while reading file: {e}")
        st.stop()

    cols = list(df.columns)

    open_col_default = "Time" if "Time" in cols else cols[0]
    close_col_default = "Time_2" if "Time_2" in cols else cols[min(1, len(cols)-1)]
    profit_col_default = "Profit" if "Profit" in cols else cols[0]
    symbol_col_default = "Symbol" if "Symbol" in cols else cols[0]
    commission_col_default = "Commission" if "Commission" in cols else cols[0]
    swap_col_default = "Swap" if "Swap" in cols else cols[0]

    st.subheader("Confirm Columns")
    c1, c2, c3 = st.columns(3)
    with c1:
        open_col = st.selectbox("Open Time Column", cols, index=cols.index(open_col_default))
        close_col = st.selectbox("Close Time Column", cols, index=cols.index(close_col_default))
    with c2:
        profit_col = st.selectbox("Profit Column", cols, index=cols.index(profit_col_default))
        symbol_col = st.selectbox("Symbol Column", cols, index=cols.index(symbol_col_default))
    with c3:
        commission_col = st.selectbox("Commission Column", cols, index=cols.index(commission_col_default))
        swap_col = st.selectbox("Swap Column", cols, index=cols.index(swap_col_default))

    trade_limit = st.number_input("Show dates where client took more than this many trades in a day", min_value=1, value=80, step=1)

    if st.button("Run Analysis"):
        try:
            df[open_col] = pd.to_datetime(df[open_col], errors="coerce")
            df[close_col] = pd.to_datetime(df[close_col], errors="coerce")
            df[profit_col] = pd.to_numeric(df[profit_col], errors="coerce")
            df[commission_col] = pd.to_numeric(df[commission_col], errors="coerce")
            df[swap_col] = pd.to_numeric(df[swap_col], errors="coerce")

            df = df.dropna(subset=[open_col, close_col, profit_col]).copy()
            df[commission_col] = df[commission_col].fillna(0)
            df[swap_col] = df[swap_col].fillna(0)

            if df.empty:
                st.error("No valid position rows found after cleaning. Please check selected columns.")
                st.stop()

            df["holding_seconds"] = (df[close_col] - df[open_col]).dt.total_seconds()
            df = df[df["holding_seconds"] >= 0].copy()
            df["trade_date"] = df[open_col].dt.date
            df["net_pnl"] = df[profit_col] + df[commission_col] + df[swap_col]

            short_trades = df[df["holding_seconds"] <= 180].copy()

            total_trades = len(df)
            short_count = len(short_trades)
            short_pct = (short_count / total_trades * 100) if total_trades > 0 else 0

            positive_short_profit = short_trades.loc[short_trades[profit_col] > 0, profit_col].sum()
            negative_short_profit = short_trades.loc[short_trades[profit_col] < 0, profit_col].sum()

            total_profit = df[profit_col].sum()
            total_commission = df[commission_col].sum()
            total_swap = df[swap_col].sum()
            overall_net_pnl = df["net_pnl"].sum()
            overall_status = "Overall Profit" if overall_net_pnl > 0 else "Overall Loss" if overall_net_pnl < 0 else "Break-even"

            df_sorted = df.sort_values(by=open_col).copy()
            df_sorted["prev_close"] = df_sorted[close_col].shift(1)
            df_sorted["gap_seconds"] = (df_sorted[open_col] - df_sorted["prev_close"]).dt.total_seconds()
            quick_reentry = df_sorted[(df_sorted["gap_seconds"] >= 0) & (df_sorted["gap_seconds"] <= 5)].copy()

            daily_counts = df.groupby("trade_date").size().reset_index(name="trade_count")
            above_limit = daily_counts[daily_counts["trade_count"] > trade_limit].copy()

            st.markdown(f"## Trade Analysis Summary - Account {account_number}")

            summary_df = pd.DataFrame({
                "Metric": [
                    "Total Trades",
                    "Trades ≤ 180 sec",
                    "% Trades ≤ 180 sec",
                    "Positive Profit (≤180 sec)",
                    "Negative Profit (≤180 sec)",
                    "Total Profit",
                    "Total Commission",
                    "Total Swap",
                    "Overall Net P&L (Profit + Commission + Swap)",
                    "Overall Result"
                ],
                "Value": [
                    total_trades,
                    short_count,
                    f"{short_pct:.2f}%",
                    round(positive_short_profit, 2),
                    round(negative_short_profit, 2),
                    round(total_profit, 2),
                    round(total_commission, 2),
                    round(total_swap, 2),
                    round(overall_net_pnl, 2),
                    overall_status
                ]
            })

            st.dataframe(summary_df, use_container_width=True, hide_index=True)

            if not above_limit.empty:
                st.markdown(f"### Dates with More Than {trade_limit} Trades")
                st.dataframe(above_limit, use_container_width=True, hide_index=True)
            else:
                st.markdown(f"### Dates with More Than {trade_limit} Trades")
                st.info(f"No dates found where trades were more than {trade_limit}.")

            st.markdown("### Close → Next Open within 5 sec")
            st.write(f"Count: {len(quick_reentry)}")
            if not quick_reentry.empty:
                st.dataframe(quick_reentry, use_container_width=True)
            else:
                st.info("No such cases found.")

            st.markdown("### Scalping Trade Details (Trades ≤ 180 sec)")
            if not short_trades.empty:
                st.dataframe(short_trades, use_container_width=True)
            else:
                st.info("No trades closed within 180 seconds.")

            st.markdown("### Toxic Flow Verdict")
            if short_pct >= 60:
                st.error("High short-duration trading detected. Possible toxic flow.")
            elif short_pct >= 30:
                st.warning("Moderate short-duration trading detected. Needs review.")
            else:
                st.success("Trading flow looks relatively normal based on holding time.")

            # Download buttons
            scalping_download_df = short_trades.copy()
            summary_download = summary_df.copy()
            above_limit_download = above_limit.copy()
            quick_reentry_download = quick_reentry.copy()

            st.markdown("### Download Reports")
            d1, d2 = st.columns(2)

            with d1:
                st.download_button(
                    label="Download Scalping Trade Details (Excel)",
                    data=to_excel_bytes({"Scalping_Trades": scalping_download_df}),
                    file_name=f"scalping_trade_details_{account_number}.xlsx",
                    mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
                )

            with d2:
                st.download_button(
                    label="Download Full Analysis Report (Excel)",
                    data=to_excel_bytes({
                        "Summary": summary_download,
                        "Scalping_Trades": scalping_download_df,
                        "Close_to_Open_5sec": quick_reentry_download,
                        "Days_Above_Limit": above_limit_download
                    }),
                    file_name=f"trade_analysis_report_{account_number}.xlsx",
                    mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
                )

        except Exception as e:
            st.error(f"Error during analysis: {e}")
