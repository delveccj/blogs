# Collecting ADS-B Aircraft Position Data from OpenSky Network

|||info
**Assignment Overview**

In this project, you will learn how to collect real-time aircraft position data from the OpenSky Network using their free REST API. ADS-B (Automatic Dependent Surveillance-Broadcast) is a technology where aircraft broadcast their GPS position, altitude, speed, and heading. The OpenSky Network is a crowd-sourced research project that collects this data from volunteer receivers around the world.
|||

---

## Learning Objectives

By the end of this assignment, you will be able to:

1. Understand what ADS-B data is and how it is collected
2. Register for an OpenSky Network account
3. Make authenticated API requests to retrieve live aircraft positions
4. Parse and interpret the JSON response data
5. Store collected data in a structured format for analysis

---

## Prerequisites

Before starting, ensure you have the following installed on your system:

- **Python 3.10+** (required by the OpenSky Python library)
- **pip** (Python package manager)
- A text editor or IDE (VS Code, PyCharm, etc.)
- Basic familiarity with Python and JSON

You can check your Python version by running:

```bash
python3 --version
```

---

## Background: What is ADS-B?

ADS-B stands for **Automatic Dependent Surveillance-Broadcast**. It is a surveillance technology in which an aircraft determines its position via satellite navigation (GPS) and periodically broadcasts it. This enables it to be tracked by air traffic control and other aircraft.

Key facts:

- ADS-B is **mandatory** for most aircraft in controlled airspace (as of 2020 in the United States)
- Aircraft transmit their position roughly **every second**
- Signals can be received by anyone with a simple ADS-B receiver — it is **unencrypted public data**
- The OpenSky Network aggregates data from thousands of volunteer receivers worldwide

|||info
**Legal Note**

ADS-B data is publicly broadcast over radio frequencies. Collecting and using this data for research and educational purposes is legally permitted. The OpenSky Network's terms of use require that you cite their work if you publish results. See the Citation section at the end of this guide.
|||

---

## Part 1: Registering for an OpenSky Network Account

While you can make anonymous API calls, registering for a free account gives you significantly more daily credits (4,000 vs. 400) and better time resolution (5 seconds vs. 10 seconds).

### Steps

1. Navigate to [https://opensky-network.org](https://opensky-network.org)
2. Click **Sign up** in the upper-right corner
3. Fill in the registration form with your email address
4. Choose a username and password
5. Verify your email address via the confirmation link
6. Log in to your new account

|||important
**Keep your credentials secure.** Never commit your username and password directly into source code. Use environment variables or a secrets file that is excluded from version control (e.g., added to `.gitignore`).
|||

---

## Part 2: Understanding the OpenSky REST API

The base URL for the OpenSky REST API is:

```
https://opensky-network.org/api
```

### Available Endpoints (Free Tier)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/states/all` | GET | Retrieve state vectors (positions) for all visible aircraft |
| `/flights/all` | GET | Retrieve flights within a time interval |
| `/flights/aircraft` | GET | Retrieve flights for a specific aircraft (by ICAO24 address) |
| `/flights/arrival` | GET | Retrieve arrivals at a specific airport |
| `/flights/departure` | GET | Retrieve departures from a specific airport |
| `/tracks/all` | GET | Retrieve the trajectory (track) for a specific aircraft |

### What is a State Vector?

A **state vector** is the core data unit in the OpenSky API. It represents the current state of a single aircraft and includes:

| Field | Type | Description |
|-------|------|-------------|
| `icao24` | string | Unique ICAO 24-bit address of the transponder (hex) |
| `callsign` | string | Callsign broadcast by the aircraft (e.g., "UAL123") |
| `origin_country` | string | Country of registration |
| `longitude` | float | Longitude in degrees (WGS84) |
| `latitude` | float | Latitude in degrees (WGS84) |
| `baro_altitude` | float | Barometric altitude in meters |
| `geo_altitude` | float | Geometric altitude in meters |
| `velocity` | float | Speed over ground in meters/second |
| `true_track` | float | True track in degrees (0-360) |
| `vertical_rate` | float | Vertical rate in meters/second |
| `squawk` | string | Squawk code (4-digit transponder code) |
| `position_source` | int | Source of position (0=ADS-B, 1=ASTERIX, 2=MLAT, 3=Flarm) |

### Rate Limiting and Credits

API access is managed through a **credit system**. Each request consumes credits based on the geographic area queried.

| User Type | Daily Credits | Time Resolution |
|-----------|---------------|-----------------|
| Anonymous | 400 | 10 seconds |
| Authenticated (free) | 4,000 | 5 seconds |
| Contributing feeder | 8,000 | 5 seconds |

Credit costs for `/states/all`:

| Bounding Box Area | Credits |
|-------------------|---------|
| 0–25 sq degrees | 1 |
| 25–100 sq degrees | 2 |
| 100–400 sq degrees | 3 |
| Global or >400 sq degrees | 4 |

---

## Part 3: Authentication with OAuth2

Since March 2026, OpenSky uses **OAuth2 client credentials** for authentication (legacy basic auth is no longer supported).

### Step 1: Obtain Your Client Credentials

1. Log in to [https://opensky-network.org](https://opensky-network.org)
2. Navigate to your **Account Settings** or **API Access** section
3. Generate your **Client ID** and **Client Secret**

### Step 2: Request an Access Token

You must exchange your client credentials for a temporary access token. Tokens expire after **30 minutes**.

Using `curl`:

```bash
curl -s -X POST "https://auth.opensky-network.org/auth/realms/opensky-network/protocol/openid-connect/token" \
  -d "grant_type=client_credentials" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET" | python3 -m json.tool
```

You will receive a JSON response containing:

- `access_token` — the Bearer token to use in API requests
- `expires_in` — number of seconds until the token expires (typically 1800)
- `token_type` — always `Bearer`

### Step 3: Use the Token in API Requests

Pass the token in the `Authorization` header:

```bash
curl -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  "https://opensky-network.org/api/states/all?limit=5"
```

---

## Part 4: Your First Data Collection Script

Now let's write a Python script to collect aircraft position data.

### Step 1: Install Dependencies

```bash
pip install requests
```

### Step 2: Create the Script

Create a new file called `collect_opensky.py` and add the following code:

```python
import requests
import json
import time
from datetime import datetime

# --- Configuration ---
TOKEN_URL = "https://auth.opensky-network.org/auth/realms/opensky-network/protocol/openid-connect/token"
API_BASE = "https://opensky-network.org/api"

# Replace with your own credentials (use environment variables in production)
CLIENT_ID = "your_client_id_here"
CLIENT_SECRET = "your_client_secret_here"


def get_access_token():
    """Request an OAuth2 access token from OpenSky."""
    response = requests.post(TOKEN_URL, data={
        "grant_type": "client_credentials",
        "client_id": CLIENT_ID,
        "client_secret": CLIENT_SECRET,
    })
    response.raise_for_status()
    return response.json()["access_token"]


def get_all_states(token, bbox=None):
    """
    Retrieve state vectors for all visible aircraft.
    
    bbox: optional bounding box as (lamin, lomin, lamax, lomax)
    """
    headers = {"Authorization": f"Bearer {token}"}
    params = {}

    if bbox:
        params["lamin"] = bbox[0]
        params["lomin"] = bbox[1]
        params["lamax"] = bbox[2]
        params["lomax"] = bbox[3]

    response = requests.get(f"{API_BASE}/states/all", headers=headers, params=params)
    response.raise_for_status()
    return response.json()


def format_state_vector(sv):
    """Extract key fields from a raw state vector array."""
    return {
        "icao24": sv[0],
        "callsign": sv[1].strip() if sv[1] else None,
        "origin_country": sv[2],
        "longitude": sv[5],
        "latitude": sv[6],
        "baro_altitude_m": sv[7],
        "velocity_ms": sv[9],
        "true_track_deg": sv[10],
        "vertical_rate_ms": sv[11],
        "squawk": sv[14],
    }


def main():
    print("=== OpenSky Network Data Collection ===")
    print(f"Timestamp: {datetime.utcnow().isoformat()}Z\n")

    # Step 1: Authenticate
    print("Authenticating...")
    token = get_access_token()
    print("Token obtained successfully.\n")

    # Step 2: Define a bounding box (example: continental US)
    # Format: (lamin, lomin, lamax, lomax)
    us_bbox = (24.0, -125.0, 49.0, -66.0)

    # Step 3: Query the API
    print("Querying aircraft states for the continental US...")
    raw_data = get_all_states(token, bbox=us_bbox)

    # Step 4: Parse and display results
    timestamp = raw_data.get("time")
    states = raw_data.get("states", [])

    print(f"Total aircraft received: {len(states)}\n")

    # Display the first 10 aircraft
    print("--- Sample Aircraft (first 10) ---")
    for sv in states[:10]:
        ac = format_state_vector(sv)
        print(
            f"  {ac['icao24']} | {ac['callsign'] or 'N/A':>8s} | "
            f"Lat: {ac['latitude']:.2f}, Lon: {ac['longitude']:.2f} | "
            f"Alt: {ac['baro_altitude_m']}m | Country: {ac['origin_country']}"
        )

    # Step 5: Save full data to a JSON file
    output_file = f"opensky_data_{timestamp}.json"
    parsed_states = [format_state_vector(sv) for sv in states]
    with open(output_file, "w") as f:
        json.dump({
            "query_time": timestamp,
            "bbox": {"lamin": us_bbox[0], "lomin": us_bbox[1], "lamax": us_bbox[2], "lomax": us_bbox[3]},
            "aircraft_count": len(parsed_states),
            "aircraft": parsed_states,
        }, f, indent=2)

    print(f"\nFull data saved to: {output_file}")


if __name__ == "__main__":
    main()
```

### Step 3: Run the Script

```bash
python3 collect_opensky.py
```

|||warning
**Remember to replace** `your_client_id_here` and `your_client_secret_here` with your actual credentials. For security, use environment variables instead of hardcoding credentials.
|||

---

## Part 5: Using Environment Variables (Recommended)

To avoid hardcoding credentials, store them in environment variables.

### Step 1: Set Environment Variables

On macOS / Linux:

```bash
export OPENSKY_CLIENT_ID="your_client_id_here"
export OPENSKY_CLIENT_SECRET="your_client_secret_here"
```

On Windows (PowerShell):

```powershell
$env:OPENSKY_CLIENT_ID="your_client_id_here"
$env:OPENSKY_CLIENT_SECRET="your_client_secret_here"
```

### Step 2: Update the Script

Replace the credential lines with:

```python
import os

CLIENT_ID = os.environ.get("OPENSKY_CLIENT_ID")
CLIENT_SECRET = os.environ.get("OPENSKY_CLIENT_SECRET")
```

---

## Part 6: Collecting Data Over Time

For analysis, you will likely want to collect data at regular intervals. Here is a simple polling loop:

```python
import time
import os
import requests
import json
from datetime import datetime

# ... (include get_access_token, get_all_states, format_state_vector from above) ...

POLL_INTERVAL_SECONDS = 60  # Collect data every 60 seconds
DURATION_MINUTES = 10       # Run for 10 minutes


def collect_over_time():
    """Poll the OpenSky API at regular intervals and save results."""
    token = get_access_token()
    all_readings = []

    start_time = time.time()
    end_time = start_time + (DURATION_MINUTES * 60)

    reading_count = 0
    while time.time() < end_time:
        reading_count += 1
        print(f"\n--- Reading #{reading_count} at {datetime.utcnow().isoformat()}Z ---")

        try:
            raw_data = get_all_states(token)
            parsed = [format_state_vector(sv) for sv in raw_data.get("states", [])]
            all_readings.append({
                "timestamp": raw_data.get("time"),
                "count": len(parsed),
                "aircraft": parsed,
            })
            print(f"  Collected {len(parsed)} aircraft.")
        except requests.exceptions.HTTPError as e:
            print(f"  Error: {e}")
            if e.response.status_code == 429:
                print("  Rate limit hit. Waiting 60 seconds...")
                time.sleep(60)
                continue

        time.sleep(POLL_INTERVAL_SECONDS)

    # Save all readings
    output_file = f"opensky_session_{int(start_time)}.json"
    with open(output_file, "w") as f:
        json.dump(all_readings, f, indent=2)
    print(f"\nSession data saved to: {output_file}")


if __name__ == "__main__":
    collect_over_time()
```

---

## Part 7: What Can You Do With This Data?

Once you have collected position data, here are some ideas for analysis:

1. **Count aircraft by origin country** — which countries have the most flights in your area?
2. **Altitude distribution** — what altitudes are most common?
3. **Speed analysis** — what is the average ground speed of aircraft at different altitudes?
4. **Bearing / heading histogram** — what are the most popular flight directions?
5. **Airspace density over time** — how does traffic change throughout the day?

|||info
**Project Idea:** Combine OpenSky data with a mapping library like `folium` (Python) or `Leaflet` (JavaScript) to create a live aircraft visualization on a map.
|||

---

## Part 8: Troubleshooting

| Problem | Solution |
|---------|----------|
| `401 Unauthorized` | Your token has expired. Request a new one. |
| `429 Too Many Requests` | You have exceeded your credit quota. Wait for credits to refresh (check `X-Rate-Limit-Retry-After-Seconds` header). |
| `404 Not Found` | Check that your API endpoint and query parameters are correct. |
| Empty `states` array | The bounding box may not cover any area with air traffic. Try a wider area. |
| `ModuleNotFoundError: requests` | Run `pip install requests` |

---

## Citation

If you use OpenSky Network data in a publication, presentation, or web page, you must cite the original paper:

> Matthias Schäfer, Martin Strohmeier, Vincent Lenders, Ivan Martinovic and Matthias Wilhelm.
> "Bringing Up OpenSky: A Large-scale ADS-B Sensor Network for Research".
> In Proceedings of the 13th IEEE/ACM International Symposium on Information Processing in Sensor Networks (IPSN), pages 83-94, April 2014.

You may also include the URL: `https://opensky-network.org`

See the [OpenSky Terms of Use](https://opensky-network.org/about/terms-of-use) for full details on the data license.

---

## Resources

- [OpenSky Network](https://opensky-network.org)
- [REST API Documentation](https://openskynetwork.github.io/opensky-api/rest.html)
- [Python API Client](https://github.com/openskynetwork/opensky-api)
- [OpenSky Discord](https://discord.gg/RPh89jpVVz)
