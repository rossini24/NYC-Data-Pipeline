# NYC Mobility Data Pipeline

End-to-end Python pipeline integrating **New York Citi Bike trips**, **historical weather data** and the **New York Knicks NBA schedule**, to find out what drives bike-sharing demand.

> Course project – *Data Management*, Master's Degree in Data Science, Università degli Studi di Milano-Bicocca
> Team: Beatrice Bigoni, Clelia Meloni, Luca Rossini

📄 **Full report:** [Project_Report_NYC_Mobility_Data_Pipeline.pdf](Project_Report_NYC_Mobility_Data_Pipeline.pdf)
💻 **Code:** [NYC_Mobility_Data_Pipeline.ipynb](NYC_Mobility_Data_Pipeline.ipynb)

---

## Key results

| Finding | Value |
|---|---|
| Average hourly trips during **Knicks home games** | **6,969** (+32% vs. no-game hours) |
| Average hourly trips with **no game** | 5,253 |
| Correlation of **temperature** with hourly trips | **ρ ≈ 0.42** (main driver) |
| Effect of **rain** | any rainfall lowers demand (clear: 5,445 trips/h → moderate rain: 3,667) |

## Research questions

1. **Weather** – To what extent do meteorological variables (temperature, precipitation, wind…) influence the hourly volume of trips and the distribution of demand classes?
2. **Events** – Do major sporting events (Knicks home games at Madison Square Garden) generate a measurable change in demand compared to usual patterns?

## Data sources

| Source | Type | How it is acquired |
|---|---|---|
| Citi Bike trip data | Structured – monthly CSV files (millions of trips) | Batch loading of local files, processed one file at a time |
| Open-Meteo historical weather | Semi-structured – REST API (JSON) | API calls with persistent caching (`requests_cache`) and retry logic with exponential backoff |
| New York Knicks schedule (Basketball Reference) | Semi-structured – HTML tables | Dynamic web scraping with Selenium (explicit waits, cookie banner handling) |

## Pipeline

```
 Citi Bike CSVs ─┐
 Weather API  ───┼─► Cleaning ─► Integration ─► Data quality ─► SQLite ─► SQL validation & EDA
 Knicks scraping ┘
```

1. **Acquisition** – three heterogeneous sources, with caching and retries for robust, reproducible downloads.
2. **Cleaning & feature engineering**
   - Citi Bike: memory-efficient iterative loading, trip duration, outlier filter (trips between 1 and 180 minutes), hourly aggregation.
   - Weather: type enforcement and rounding, UTC → local time alignment.
   - Knicks: removal of repeated HTML headers, explicit home/away flag, regex time parsing, rounding of tip-off times to the hour.
3. **Integration** – common hourly granularity, surrogate primary key `pk_dt` (`YYYYMMDDHH`), event **time-window expansion** (from 1 hour before to 2 hours after tip-off), left joins with Citi Bike as master table.
4. **Data quality** – completeness (missing weather hours fixed at the source by extending the download window), accuracy (outlier filtering), consistency (timezones and granularity), uniqueness (deduplication on `pk_dt`).
5. **Storage** – incremental, deduplicated *Analytical Base Table* in a SQLite database.
6. **Validation** – SQL queries, correlation analysis and exploratory data analysis.

The final table also includes a **demand class target** (Low / Medium / High, based on quartiles), ready for supervised machine learning.

## Tech stack

Python · pandas · requests / requests_cache · Selenium · SQLite · SQL · Jupyter Notebook

## How to run

1. Download the Citi Bike monthly trip files from the official [Citi Bike System Data](https://citibikenyc.com/system-data) page and place the `*-citibike-tripdata_*.csv` files in the same folder as the notebook.
2. Install the Python packages imported at the top of the notebook (e.g. `pip install pandas requests requests-cache selenium`) and make sure Google Chrome is installed for Selenium.
3. Open `NYC_Mobility_Data_Pipeline.ipynb` and run the cells in order.

> The raw Citi Bike data is not included in this repository because of its size.
