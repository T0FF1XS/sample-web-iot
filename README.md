# IoT Climate Monitor

An ESP32 + DHT11 sensor sends temperature and humidity to a **PHP API**, which stores it in **MySQL**. A **web dashboard** shows the data and gives **actionable insights** (comfort score, mold risk, trends, offline alerts).

```
ESP32 + DHT11 --POST JSON + X-API-Key--> api/api.php --> MySQL (iot_db)
Browser dashboard <-- api.php (history) + insights.php (analysis) <--+
```

---

## WHAT TO DO (step by step)

### Step 0 - What you need
- A PC with **XAMPP** (Apache + PHP + MySQL) - or PHP 8+ and MySQL/MariaDB installed separately
- *Optional hardware:* ESP32 board, DHT11 sensor, 3 jumper wires, USB cable
- *Optional software:* Arduino IDE (for the ESP32). Python 3 (for the simulator)
- No hardware? Skip Step 5 and use the simulator in Step 4.

### Step 1 - Put the project on your server
1. Copy the whole `climate-monitor` folder into `C:\xampp\htdocs\` (Windows) or `/opt/lampp/htdocs/` (Linux/macOS).
2. Start **Apache** and **MySQL** in the XAMPP control panel.
   *(Without XAMPP: open a terminal in the project folder and run `php -S 0.0.0.0:8000`.)*

### Step 2 - Create the database
Open a terminal in the project folder and run:
```
mysql -u root -p < database/schema.sql
```
(or open phpMyAdmin -> Import -> choose `database/schema.sql`). This creates database `iot_db` with tables `devices` and `sensor_data`.

### Step 3 - Configure the API
Edit `api/config.php`:
- `db` -> your MySQL user and password (XAMPP default: user `root`, empty password).
- `api_key` -> **change it** to your own secret text. Remember it - the ESP32 needs the same value.

### Step 4 - Test with fake data (no hardware needed)
```
python3 simulator/simulate.py seed --hours 24 | mysql -u root iot_db
python3 simulator/simulate.py live --url http://localhost/climate-monitor/api/api.php --key YOUR_API_KEY
```
- First command loads 24 hours of history. Second command sends a new reading every 5 seconds (stop with Ctrl+C).
- Using `php -S ...:8000`? Use `http://localhost:8000/api/api.php` as the URL.

### Step 5 - Connect the real ESP32
1. **Wire the DHT11:** DATA -> GPIO 4, VCC -> 3V3, GND -> GND (add a 10k resistor between DATA and VCC if your module has no built-in one).
2. In Arduino IDE: install **ESP32 board support**, and the library **"DHT sensor library" by Adafruit** (accept the Adafruit Unified Sensor dependency).
3. Open `esp32/esp32.ino` and edit the "EDIT THESE" block:
   - `WIFI_SSID`, `WIFI_PASSWORD` - your 2.4 GHz WiFi (ESP32 does not support 5 GHz)
   - `SERVER_URL` - `http://<YOUR-PC-IP>/climate-monitor/api/api.php`. Find your IP with `ipconfig` (Windows) or `ip a` (Linux). **Do not use `localhost`**; the ESP32 is a different device.
   - `API_KEY` - same as in `api/config.php`
   - `DEVICE_ID` - a unique name, e.g. `esp32-lab`
4. Select your board, upload, then open **Serial Monitor at 115200 baud**. You should see `-> 201` after every reading.
5. If it fails: check the PC firewall allows port 80, the PC and ESP32 are on the same WiFi, and the API key matches (a `401` means wrong key).

### Step 6 - Open the dashboard
Go to `http://localhost/climate-monitor/dashboard/` (or `http://localhost:8000/dashboard/`).
- Pick a device and a time range, watch it refresh every 5 s.
- Read the **Actionable insights** panel and follow the "Do this" advice.
- Click **Export CSV** to download the shown data for your report.

### Step 7 - Prepare your demo / report (for the group project)
1. Take screenshots of: wiring, Serial Monitor, phpMyAdmin table with rows, the dashboard.
2. Trigger insights live: breathe on the sensor (humidity up), hold it in your hand (temperature up), unplug the ESP32 for 1 minute (the dashboard shows **Sensor offline**).
3. Explain the data flow diagram above and the insight rules (table below).

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Dashboard says "API ERROR" | Apache/MySQL not running, or wrong DB password in `api/config.php` |
| ESP32 prints `-1` / connection refused | Wrong IP in `SERVER_URL`, firewall, or different network |
| ESP32 prints `401` | `API_KEY` differs between firmware and `config.php` |
| ESP32 prints `DHT11 read failed` | Check wiring/pin 4; DHT11 needs 3.3 V and a pull-up |
| Charts empty but no error | No data in the last hour: pick a longer range or run the simulator |
| Times look 8 hours off in MySQL | Data is stored in UTC; the dashboard converts to your local time |

## Project layout

| Path | Purpose |
|---|---|
| `esp32/esp32.ino` | Firmware: reads DHT11, uploads every 5 s, WiFi reconnect, retries |
| `api/api.php` | `POST` store reading (API-key protected) / `GET` history, device list |
| `api/insights.php` | Analysis engine: stats, derived metrics, trend, recommendations |
| `api/lib.php`, `api/config.php` | DB connection, helpers, thresholds, secrets |
| `database/schema.sql` | `devices` and `sensor_data` tables |
| `dashboard/index.html` | Dashboard (Chart.js, light/dark theme, CSV export) |
| `simulator/simulate.py` | Fake sensor for demos without hardware |

## API reference

`POST api/api.php` with header `X-API-Key: <key>`
```json
{ "device_id": "esp32-01", "temperature": 24.5, "humidity": 55.0, "rssi": -61 }
```
Returns `201` on success; `400` invalid data, `401` wrong key, `500` database error.

`GET api/api.php?hours=1&device=esp32-01` - history, oldest first (ranges over 1 h are averaged with min/max per bucket)
`GET api/api.php?devices=1` - known devices and last seen time
`GET api/insights.php?hours=24&device=esp32-01` - analysis and recommendations

## Insights

| Insight | How it is computed | Suggested action |
|---|---|---|
| Comfort score (0-100) | Distance from 20.5-24.5 C and 40-60 % RH | Overall status |
| Too hot / too cold | Thresholds in `config.php` | Ventilate, cool, heat |
| Mold & condensation risk | RH >= 65 / 75 %, dew point within 2.5 C of air | Dehumidify, ventilate |
| Heat index | NOAA Rothfusz regression | Prioritise dehumidifying |
| Fast temperature change | Least-squares slope, last 30 min | Look for heat source / open window |
| Recurring problems | >= 25 % of selected range beyond limits | Permanent fix |
| Unusual reading | z-score >= 3 vs window average | Check sensor placement |
| Offline / data gaps / weak WiFi | Age of last reading, completeness %, RSSI | Check power / WiFi |

## Security notes
- Writes need the shared API key; all SQL uses prepared statements; the dashboard renders text with `textContent`.
- For internet access use HTTPS (or a tunnel like ngrok) and a dedicated MySQL user (see the comment block in `schema.sql`).

## Suggested team roles (3-4 members)

| Role | Owns | Deliverables |
|---|---|---|
| 1. Hardware / firmware | `esp32/` | Wiring diagram, firmware, WiFi-drop and unplugged-sensor tests |
| 2. Backend / API | `api/api.php`, `lib.php` | Endpoints, validation, curl/Postman tests |
| 3. Database & insights | `database/`, `api/insights.php` | Schema, thresholds, justification of rules |
| 4. Frontend & documentation | `dashboard/`, README | Dashboard, screenshots, demo video |

With 3 members, merge role 4 into the others.
