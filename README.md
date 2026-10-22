# NYC Mobility Data Pipeline

End-to-end Python pipeline integrating **New York Citi Bike trips**, **historical weather data** and the **New York Knicks NBA schedule**, to find out what drives bike-sharing demand.

> Course project – *Data Management*, Master's Degree in Data Science, Università degli Studi di Milano-Bicocca
> Team: Beatrice Bigoni, Clelia Meloni, Luca Rossini

📄 **Full report:** [Project_Report_NYC_Mobility_Data_Pipeline.pdf](Project_Report_NYC_Mobility_Data_Pipeline.pdf)
💻 **Code:** [NYC_Mobility_Data_Pipeline.ipynb](NYC_Mobility_Data_Pipeline.ipynb)

---

## Key results

Analysis period: **1 November 2024 – 31 October 2025**, one row per hour (**8,759 hours**).

| Finding | Value |
|---|---|
| Average hourly trips during **Knicks home games** | **6,969** (+32% vs. hours with no game) |
| Average hourly trips during away games | 6,107 |
| Average hourly trips with **no game** | 5,253 |
| Correlation of **temperature** with hourly trips | **0.42** (main driver) |
| Correlation of **humidity** with hourly trips | −0.25 |
| Effect of **rain** | every rain class is below clear weather (clear: 5,445 trips/h, rain: 3,654 trips/h) |

## Research questions

1. **Weather** – To what extent do meteorological variables (temperature, precipitation, humidity, wind) influence the hourly volume of trips and the distribution of demand classes?
2. **Events** – Do major sporting events (Knicks home games at Madison Square Garden) generate a measurable change in demand compared to usual patterns?

## Data sources

| Source | Type | How it is acquired |
|---|---|---|
| [Citi Bike trip data](https://citibikenyc.com/system-data) | Structured – 53 monthly CSV files | Automatic file detection (`*-citibike-tripdata_*.csv`) and file-by-file processing to keep memory usage low |
| [Open-Meteo](https://open-meteo.com/) historical weather API | Hourly time series (temperature, apparent temperature, humidity, precipitation, weather code, wind) | `openmeteo-requests` client with persistent caching (`requests-cache`) and automatic retries (`retry-requests`) |
| [Basketball Reference](https://www.basketball-reference.com/) – Knicks schedule, 2024-25 and 2025-26 seasons | Semi-structured – HTML tables | Dynamic web scraping with Selenium (cookie banner handling, explicit waits) and pandas `read_html` |

## Pipeline

```
 Citi Bike CSVs ─┐
 Weather API  ───┼─► Cleaning ─► Integration ─► Target ─► SQLite ─► SQL validation & EDA
 Knicks scraping ┘
```

1. **Acquisition** – three heterogeneous sources (local files, REST API, web scraping).
2. **Cleaning & feature engineering**
   - *Citi Bike*: only the needed columns are loaded, trip duration is computed, trips shorter than 1 minute or longer than 3 hours are removed, member/casual counts are created and everything is aggregated by hour.
   - *Weather*: values rounded to one decimal, timestamps converted to New York local time.
   - *Knicks*: repeated header rows removed, home/away flag derived from the `@` marker, tip-off times parsed with regex and rounded to the nearest hour.
3. **Integration**
   - common hourly key `pk_dt` (`YYYYMMDDHH`) for all three sources;
   - consistency check: full weather coverage for every bike hour;
   - **event time window**: each game marks 4 hours (1 hour before tip-off, tip-off, and the 2 following hours);
   - left joins with Citi Bike as master table; hours without a game are labelled `No Game`.
4. **Target variable** – `demand_class` (Low / Medium / High) from the quartiles of hourly trips, ready for supervised machine learning.
5. **Storage** – incremental save into a SQLite database (`nyc_mobility.db`, table `abt_classification`) with deduplication on `pk_dt`.
6. **Validation & EDA** – SQL queries on events and weather, correlation analysis and charts (seaborn / matplotlib).

## Tech stack

Python · pandas · NumPy · Open-Meteo API (openmeteo-requests, requests-cache, retry-requests) · Selenium · BeautifulSoup · Docker · SQLite · SQL · seaborn · matplotlib · Jupyter Notebook

## How to run

1. Download the Citi Bike monthly trip files for November 2024 – October 2025 from the [Citi Bike System Data](https://citibikenyc.com/system-data) page and put the CSV files in the notebook's working folder (set in the *Set Work directory* cell).
2. Install the packages:
   ```bash
   pip install pandas numpy lxml requests requests-cache retry-requests openmeteo-requests beautifulsoup4 selenium seaborn matplotlib
   ```
3. Start a Selenium Chrome container, which the notebook uses for web scraping:
   ```bash
   docker run -d -p 4444:4444 --shm-size=2g --name selenium selenium/standalone-chrome
   ```
   Inside Docker Compose the notebook reaches it at `http://selenium:4444/wd/hub`; if you run the notebook locally, change the address to `http://localhost:4444/wd/hub`.
4. Open `NYC_Mobility_Data_Pipeline.ipynb` and run the cells in order.

> The raw Citi Bike data and the SQLite database are not included in this repository because of their size.
