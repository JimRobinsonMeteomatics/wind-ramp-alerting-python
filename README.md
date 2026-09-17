# wind-ramp-alerting-python
Automated Python alerting pipeline using the Meteomatics Weather API to monitor hub-height wind speeds and detect grid ramp risks.

import datetime as dt
import webbrowser
import meteomatics.api as api

# ==========================================
# 1. CREDENTIALS & ISO CONFIGURATION
# ==========================================
USERNAME = "YOUR_ENTERPRISE_USERNAME"
PASSWORD = "YOUR_ENTERPRISE_PASSWORD"

# Key wind corridor coordinates across major US power markets
ISO_NODES = {
    "ERCOT (West Texas)": (32.4700, -100.4000),
    "MISO (Iowa Corridor)": (42.0308, -93.6319),
    "PJM (Western Hub/PA-WV)": (40.0000, -79.5000),
}

START_TIME = dt.datetime.now(dt.timezone.utc)
END_TIME = START_TIME + dt.timedelta(hours=6)
INTERVAL = dt.timedelta(minutes=15)

# High-resolution parameters (100m hub height + gusts)
PARAMETERS = ["wind_speed_100m:ms", "wind_gusts_10m_1h:ms"]

# Threshold for operational wind drop alert (m/s drop over 1 hour)
RAMP_THRESHOLD_MS = -4.0


def display_multi_iso_dashboard(summary_cards_html: str):
    """Generates a multi-ISO operations dashboard and opens it in the browser."""
    html_content = f"""<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Multi-ISO Wind Ramp Operations Desk</title>
    <style>
        body {{
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            background-color: #0f172a;
            color: #f8fafc;
            padding: 30px;
            margin: 0;
            display: flex;
            flex-direction: column;
            align-items: center;
        }}
        .header {{
            max-width: 900px;
            width: 100%;
            margin-bottom: 24px;
        }}
        .header h1 {{
            font-size: 1.6rem;
            margin: 0 0 6px 0;
            color: #f1f5f9;
        }}
        .header .meta {{
            font-size: 0.85rem;
            color: #94a3b8;
        }}
        .container {{
            max-width: 900px;
            width: 100%;
            display: flex;
            flex-direction: column;
            gap: 20px;
        }}
        .card {{
            background-color: #1e293b;
            border: 1px solid #334155;
            border-radius: 8px;
            padding: 20px;
            box-shadow: 0 4px 16px rgba(0, 0, 0, 0.4);
        }}
        .card.alert {{
            border-left: 6px solid #dc2626;
        }}
        .card.ok {{
            border-left: 6px solid #16a34a;
        }}
        .badge {{
            display: inline-block;
            padding: 4px 10px;
            font-size: 11px;
            font-weight: 700;
            border-radius: 4px;
            letter-spacing: 0.5px;
            margin-bottom: 10px;
        }}
        .badge.alert {{
            background-color: #dc2626;
            color: #fff;
        }}
        .badge.ok {{
            background-color: #16a34a;
            color: #fff;
        }}
        .iso-title {{
            font-size: 1.2rem;
            font-weight: 600;
            margin: 0 0 10px 0;
        }}
        table {{
            width: 100%;
            border-collapse: collapse;
            margin-top: 12px;
            font-size: 0.88rem;
        }}
        th, td {{
            text-align: left;
            padding: 8px 12px;
            border-bottom: 1px solid #334155;
        }}
        th {{
            background-color: #0f172a;
            color: #94a3b8;
            text-transform: uppercase;
            font-size: 0.72rem;
            letter-spacing: 0.5px;
        }}
        td {{
            color: #f8fafc;
        }}
    </style>
</head>
<body>
    <div class="header">
        <h1>Real-Time Wind Ramp Monitor: ERCOT / MISO / PJM</h1>
        <div class="meta">Ingested via Meteomatics API | 100m Hub Height | {dt.datetime.now(dt.timezone.utc).strftime('%Y-%m-%d %H:%M:%S UTC')}</div>
    </div>
    <div class="container">
        {summary_cards_html}
    </div>
</body>
</html>
"""
    file_path = "iso_wind_ramp_dashboard.html"
    with open(file_path, "w", encoding="utf-8") as f:
        f.write(html_content)

    webbrowser.open(file_path)
    print(f"\nMulti-ISO Dashboard generated and opened: {file_path}")


def main():
    print("Ingesting live 100m hub-height data across ERCOT, MISO, and PJM...")

    # Build coordinate list for multi-point query
    coords = list(ISO_NODES.values())

    # Query all nodes simultaneously
    df = api.query_time_series(
        coords, START_TIME, END_TIME, INTERVAL, PARAMETERS, USERNAME, PASSWORD
    )

    cards_html = ""

    for iso_name, (lat, lon) in ISO_NODES.items():
        try:
            iso_df = df.xs((lat, lon), level=("lat", "lon")).copy()
        except KeyError:
            iso_df = df.copy()

        # Calculate 1-hour rolling change (4 intervals of 15 min)
        iso_df["delta_1h_ms"] = iso_df["wind_speed_100m:ms"].diff(periods=4).round(2)
        critical_ramps = iso_df[iso_df["delta_1h_ms"] <= RAMP_THRESHOLD_MS]

        current_speed = iso_df["wind_speed_100m:ms"].iloc[0]
        max_speed = iso_df["wind_speed_100m:ms"].max()
        min_speed = iso_df["wind_speed_100m:ms"].min()

        if not critical_ramps.empty:
            worst_drop = critical_ramps.sort_values(by="delta_1h_ms").iloc[0]
            display_df = critical_ramps[["wind_speed_100m:ms", "delta_1h_ms"]].copy()
            display_df.columns = ["100m Wind Speed (m/s)", "1-Hr Ramp Delta (m/s)"]
            table_html = display_df.to_html()

            cards_html += f"""
            <div class="card alert">
                <div class="badge alert">CRITICAL RAMP RISK</div>
                <div class="iso-title">{iso_name} [Node: {lat}, {lon}]</div>
                <div><strong>Max Drop:</strong> {worst_drop['delta_1h_ms']:.1f} m/s in 60 mins | <strong>Current Hub Speed:</strong> {current_speed:.1f} m/s (Range: {min_speed:.1f} - {max_speed:.1f} m/s)</div>
                {table_html}
            </div>
            """
        else:
            cards_html += f"""
            <div class="card ok">
                <div class="badge ok">ALL CLEAR</div>
                <div class="iso-title">{iso_name} [Node: {lat}, {lon}]</div>
                <div><strong>Status:</strong> Stable generation curve. No downward ramp exceeding {abs(RAMP_THRESHOLD_MS)} m/s/hr.</div>
                <div><strong>Current Hub Speed:</strong> {current_speed:.1f} m/s | <strong>6-Hour Forecast Range:</strong> {min_speed:.1f} m/s – {max_speed:.1f} m/s</div>
            </div>
            """

    display_multi_iso_dashboard(cards_html)


if __name__ == "__main__":
    main()
