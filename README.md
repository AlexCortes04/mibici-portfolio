# MiBici Guadalajara 2025: Trip Analysis & Station Relocation Recommendations

A data analysis portfolio project built from 4.53M real trip records from Guadalajara's public bike-share system (MiBici), investigating whether underused stations are genuinely isolated from the network or simply unpopular, and whether that distinction can inform relocation decisions.

<img width="1148" height="650" alt="Dashboard walkthrough showing navigation across Overview, Who Rides, Station Performance, and Relocation Analysis pages" src="https://github.com/user-attachments/assets/574c130f-bc84-4553-82ec-92b12f1d092a" />

**Full interactive dashboard:** download [`powerbi/mibici-portfolio.pbix`](powerbi/mibici-portfolio.pbix) and open it in [Power BI Desktop](https://www.microsoft.com/en-us/power-platform/products/power-bi/downloads) (free), all data is embedded in the file, no database connection needed to explore it.

## The question

MiBici has 484 stations. Some are heavily used, some barely used at all. Before recommending any station be relocated, it matters which kind of "unused" a station actually is: a station nobody wants, or a station nobody can easily reach. Only the second case is something trip data can speak to, and only the second case is something a relocation (rather than a different fix) would help.

This project treats that question as something answerable from existing trip and station data alone, without needing MiBici's internal operational records, approaching the dataset the way an external consultant would: working only with what's publicly available.

## Data

- **Trip data:** 12 monthly CSVs (January-December 2025), ~350,000-420,000 rows each, 4,532,032 trips total. Fields: trip ID, user ID, gender, birth year, start/end timestamps, origin/destination station IDs.
- **Station data:** a single metadata file, 484 stations, with name, latitude/longitude, and dock capacity. No timestamp of its own (see Limitations).
- **External context:** Jalisco's *Encuesta Anual de Movilidad* (EAM) 2023, used to ground specific thresholds (see Methodology) in real user behavior rather than arbitrary cutoffs. Per the EAM 2023, 55% of users want "ampliación" (expansion) and 18% want "mayor disponibilidad" (better availability) when asked how MiBici could improve; most riders report walking roughly 5 minutes to an alternate station before switching to transit or rideshare if one is empty.

## Methodology

**Stack:** Python (pandas, in Jupyter notebooks) for cleaning and transformation, PostgreSQL for staged, auditable storage, Power BI Desktop for the interactive dashboard. Chosen to mirror a real analytics workflow rather than doing everything in one tool.

**Pipeline, in four stages, one notebook each:**

1. **Ingestion** (`01_ingest_staging.ipynb`): raw CSVs loaded untouched into a `staging` schema. Source files mix encodings (4 of 12 monthly files are Latin-1, the rest UTF-8), handled with a try-UTF-8/fall-back-to-Latin-1 loader.
2. **Cleaning** (`02_cleaning.ipynb`): rows are flagged, never deleted, so the full dataset stays auditable. Validity thresholds are grounded in MiBici's actual rental contract rather than guessed: trip duration is flagged invalid outside 1-480 minutes (the lower bound confirmed by checking that short trips are typically followed by a same-user retry within minutes; the upper bound is MiBici's own 8-hour continuous-use policy), and birth year is flagged invalid outside 1930-2009 (the contract's 18+ or 16+-with-guardian subscription age). Zero referential integrity issues were found between trips and stations.
3. **Modeling** (`03_modeling.ipynb`): a star schema (`Fact_Trips`, `Dim_Usuario`, `Dim_Station`, `Dim_Date`, plus a station-distance bridge table using the Haversine formula) built into an `analytics` schema.
4. **Derived metrics** (`04_derived_metrics.ipynb`): age-at-trip, trip distance, and the two metrics behind the relocation analysis (below).

**Why not simulate absolute bike availability:** the original plan was to simulate minute-by-minute bike counts per station to directly answer "when does a station run below 2 bikes." This was deliberately dropped: doing so honestly would require assuming a starting inventory and an overnight rebalancing schedule that nothing in this dataset can verify. Instead, **net flow imbalance** (departures minus arrivals, per station, per hour) is used as a proxy, it identifies stations under real operational strain without fabricating an assumption the data can't support.

**Station density, not nearest-neighbor distance:** isolation was first measured as distance to the single nearest station, but this showed almost no variation across the network (mean 231m, max 586m), the deployed network is geographically dense within a defined corridor and simply absent outside it. **Station density** (count of other stations within 400m, roughly a 5-minute walk per the EAM 2023 figure above) captures this pattern far better and is the metric actually used.

**A mid-project correction worth stating directly:** an initial relocation-candidate flag (bottom 25% of all 484 stations by both trip volume and density) was found to be contaminated by 113 stations showing exactly 0 departures in 2025, a hard gap in the distribution with no stations at all between 1 and 100 departures confirmed these are stations added to the network *after* 2025, not underused 2025 stations. The station metadata file turns out to reflect a roughly current network snapshot, not a fixed 2025-dated one. All density and candidacy calculations were recomputed using only the 371 stations with genuine 2025 activity, both for the population considered and for the percentile thresholds themselves.

## Key findings

- **MiBici served 4,532,032 trips in 2025**, with clear commute-pattern peaks at 7-9am and 5-7pm, and noticeably lower weekend volume.
- **Riders are 72% male, 28% female**, and average trip duration (9-11 minutes) is nearly flat across every age bracket, suggesting ride length is driven by trip purpose, not rider demographics.
- **Station popularity and peak-hour imbalance measure different problems.** Station 049 is the single busiest station overall by total volume. Stations 51 and 194 show extreme one-directional imbalance during specific commute hours (8,600+ and 7,200+ more departures than arrivals at their peak hour) despite being well-connected (5-7 nearby stations), pointing to an operational rebalancing need, not a network-placement problem.
- **35 stations qualify as relocation sources**: bottom 25% of established 2025 stations on both trip volume and density. Station 180 (GDL-113) is the clearest case, minimal 2025 usage, only 2 nearby stations.
- **18 stations qualify as destination candidates**: top 25% by volume, bottom 25% by density, proven demand with little nearby backup. Four of these cluster along a single corridor, Calzada Federalismo (GDL-090, GDL-067, GDL-115, GDL-098), each handling 20,000+ annual departures with only 1-2 nearby alternatives. Station GDL-097 is the most extreme single case: 20,535 departures with **zero** other stations within 400m.
- **Recommendation:** relocating capacity from isolated, low-use edge stations (like GDL-113) toward the Federalismo corridor and similarly under-served high-demand stations would relieve documented strain using MiBici's existing resources, without new infrastructure spend.
- **Cross-validated against MiBici's own public dashboard**: top-3 station rankings matched within 0-2 trips (under 0.003% variance), and station GDL-113's exact departure count (1,183) matched precisely.


<img width="575" height="325" alt="Relocation Analysis map showing 35 relocation source stations and 18 destination candidates" src="https://github.com/user-attachments/assets/c3edfc9b-7389-412b-b59e-8112a8d6ed35" />

## Dashboard

Four pages: **Overview** (trip volume and timing patterns), **Who Rides** (age/gender demographics), **Station Performance** (popularity and imbalance), and **Relocation Analysis** (the map and recommendation above). Built in Power BI Desktop, connected directly to the `analytics` schema in PostgreSQL. Data embedded in the dashboard as of October 7, 2026.

<img width="575" height="324" alt="Overview page showing total trips, hourly and weekday trip patterns" src="https://github.com/user-attachments/assets/91dd0e85-8094-4af3-8632-ef3d4e532794" />

<img width="575" height="324" alt="Who Rides page showing age and gender demographics" src="https://github.com/user-attachments/assets/3704d3ef-ddd9-4f5e-8c32-7f7d47191780" />

<img width="575" height="323" alt="Station Performance page showing most/least popular stations and peak-hour imbalance" src="https://github.com/user-attachments/assets/b86fa7e8-6576-4520-a676-895b90edfbea" />

## Repository structure

```
mibici-portfolio/
├── data/
│   ├── raw/          (gitignored, source CSVs too large for GitHub)
│   └── processed/    (gitignored)
├── notebooks/
│   ├── 01_ingest_staging.ipynb
│   ├── 02_cleaning.ipynb
│   ├── 03_modeling.ipynb
│   └── 04_derived_metrics.ipynb
├── powerbi/
│   └── mibici-portfolio.pbix
├── docs/
│   └── roadmap.md
├── .env.example
├── requirements.txt
└── README.md
```

## Reproducing this project

1. Clone the repo, copy `.env.example` to `.env` and fill in your own PostgreSQL credentials.
2. `pip install -r requirements.txt`
3. Place the 12 monthly trip CSVs and the station CSV in `data/raw/`.
4. Run the four notebooks in order (01 through 04).
5. Open `powerbi/mibici-portfolio.pbix` in Power BI Desktop and connect it to your local database.

## Known limitations

- **Age is approximate.** Only birth year is available, not full birthdate, so age-at-trip can be off by up to a year.
- **No rebalancing/truck data exists**, which is why net flow imbalance (directional) is used instead of simulated absolute bike counts.
- **The station metadata file has no snapshot date.** This surfaced directly during this analysis (see Methodology) and was corrected for, but the underlying ambiguity about exactly when stations were added or decommissioned remains.
- **Trip data can only reveal demand at stations that already exist.** This analysis can recommend relocating capacity between existing stations; it cannot identify genuinely new, currently unserved locations with unmet demand.
- **Column names are a deliberate mix of Spanish and English**: original source fields retain their Spanish names (`Usuario_Id`, `Año_de_nacimiento`), while derived columns use English (`is_valid_duration`, `distance_km`).

## Future work

- **Bike lane proximity.** Querying OpenStreetMap's cycleway data (via the Overpass API) to compute each station's distance to the nearest bike lane could clarify whether low-density relocation candidates are also underserved by cycling infrastructure, compounding the isolation problem, or whether infrastructure is adequate and it's purely a network-coverage gap.

## Sources

- MiBici trip and station data: [MiBici Datos Abiertos](https://mibici.net/es/datos-abiertos/)
- MiBici rental contract (used to ground duration and age thresholds)
- Encuesta Anual de Movilidad (EAM) 2023, Jalisco
- Official MiBici public dashboard (used for cross-validation)
