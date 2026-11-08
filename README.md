# Dubai Real Estate Analysis: Transactions, Projects and Combined Power BI Dashboards

An end-to-end data analytics project using open data from the **Dubai Land Department (DLD)**. It covers data cleaning in Excel and Power Query, data modelling, DAX measures, and three Power BI dashboards that look at property sales, new developments, and what happens when the two are combined.

**Author:** Tahoora Pathan, BBA Data Analytics, Amity University Sharjah
**Period covered:** January to August 2026
**Tools:** Excel, Power Query (M), Power BI Desktop, DAX

---

## 1. Project Goal

To answer questions like:

- How many properties are sold each month, and what kind (off-plan or ready, freehold or not, residential or commercial)?
- Which areas, metro stations, landmarks and property types see the most deals?
- Who are the biggest developers, and how large and how long are their projects?
- For projects that appear in **both** datasets, how do deals, sales value and project value compare?

---

## 2. Data Sources

Both datasets come from the DLD open data portal.

| Dataset | Raw size | After cleaning | Content |
|---|---|---|---|
| **Transactions** | ~150,500 rows | **148,807 rows** | Sales, Mortgage and Gifts deals: date, value, area, property type and subtype, parking, nearest metro, mall and landmark, project name |
| **Projects** | 324 rows | **316 rows** | Developer, start, end and adoption dates, project value, status, area, planned land, villa and unit counts |

---

## 3. Data Cleaning

### Transactions

| Issue found | What I did |
|---|---|
| Broken CSV rows: some records were split into fragments (34 odd rows with text in the wrong columns) | Re-imported in Power Query with `QuoteStyle.Csv` so quoted text is respected |
| Excel silently turned 62 transaction numbers into dates (e.g. `811-2-2026` became a date) | Imported with type detection off, so transaction numbers stayed as text |
| Duplicate rows in Sales (20 pairs) and Gifts (1 pair) | Removed the second row of each pair |
| Repeated Mortgage rows | **Kept on purpose.** One mortgage loan covers several units, and the same loan value is copied across them. Checked: all repeats were flagged as copies, giving 26,522 unique mortgage deals across 30,683 rows |
| September 2026 data covered only 3 days | Removed, so the analysis is Jan to Aug only |
| 53 Sales rows under 10,000 AED (e.g. a Dubai Marina flat at 163 AED) | Removed as unrealistic |
| Junk or empty columns (`PROCEDURE_EN`, `TOTAL_BUYER`, `TOTAL_SELLER`) | Removed |
| Text with stray spaces | Trimmed area, project and master project names |
| `INSTANCE_DATE` carried a time of day, which broke the date relationship | Converted to Date type |

### Projects

- Set start, end, adoption and inspection dates to Date type.
- Trimmed text columns; set project and developer numbers to Text (they are IDs, not amounts).
- Confirmed 324 unique project numbers, and sanity-checked values, completion % and unit counts.
- Removed **8 projects with a start date of 1 September 2026**, to match the Jan to Aug window of the Transactions data (324 to 316 rows).
- `master project` is 99% empty, so it was **not** used to link the datasets.

---

## 4. Data Model

```
                    DateTable
                        |
Transactions ------- Areas ------- Projects
      \                                /
       ---------- ProjectList ---------
```

- **DateTable** links to `Transactions[INSTANCE_DATE]`.
- **Areas** (273 distinct area names) and **ProjectList** (3,888 distinct project names) are small lookup tables. Each links one-to-many to both `Transactions` and `Projects`.
- Linking the big tables directly would create many-to-many relationships, which can double-count and filter unpredictably. The lookup tables keep the model clean.
- **`ProjectList[Matched]`** flags the **212 projects** whose names appear in both datasets. The combined dashboards filter to `Matched = Yes`, so every number covers the same set of projects.
- 103 of the 315 named Projects rows have no match in Transactions. I checked a sample, and these were genuine non-matches (mostly projects with no sales yet), not spelling errors.
- Columns the combined charts need from `Projects` (developer, status, end year, description) are copied onto `ProjectList`, so filters can reach the deals.

---

## 5. Key Measures (DAX)

Measures are kept in separate tables: `_Measures` (Transactions), `_ProjectMeasures` (Projects) and `_CombinedMeasures`.

```dax
Deals = DISTINCTCOUNT(Transactions[TRANSACTION_NUMBER])

Average Build Time (Months) =
AVERAGEX(Projects, DIVIDE(Projects[end date] - Projects[start date], 30.44))

Deals per Project = DIVIDE([Deals], [Distinct Projects])

Sell-through % = DIVIDE([Units Sold], [Total Planned Units])

Sales vs Project Value = DIVIDE([Total Sales Value], [Total Project Value])

Area Difference =
CALCULATE(
    SUM(Transactions[PROCEDURE_AREA]) - SUM(Transactions[ACTUAL_AREA]),
    Transactions[GROUP_EN] = "Sales"
)

-- Calculated column on ProjectList, so a Projects field can filter Transactions
Developer =
MINX(FILTER(Projects, Projects[PROJECT_EN] = ProjectList[PROJECT_EN]),
     Projects[DEVELOPER_EN])
```

Other measures include Units Sold, Off-Plan and Ready, Freehold and Non-Freehold, Residential and Commercial, Total Sales Value, Avg Price per Sqm, Mortgage Deals and Value, Gifts Share %, Total Project Value, and a field parameter that switches a chart between total and average units per project.

---

## 6. Dashboards

### Transactions Dashboard
- Cards: Units Sold, Avg Sale Value, Mortgage Deals, Avg Mortgage Value, Gifts Share
- Pie and donut charts: Residential vs Commercial, Off-Plan vs Ready, Freehold vs Non-Freehold
- Deals by area, nearest metro and landmark; deals by property category and subtype
- Monthly trends for units sold and mortgages
- A Sales / Mortgage slicer switches all visuals between the two views

### Projects Dashboard
- Cards: Projects, Developers, Total Project Value, Average Build Time, Total Planned Units
- Projects per DLD zone, project value by status, top developers, projects launched per month
- Project scale and value by developer, with a Sum / Average switch
- Tables: next projects to finish, gap between adoption and start date

### Combined Dashboard (two pages)
- **Overview:** the 212 matched projects. Top projects and developers by deals, sell-through, monthly deals, completion-year pipeline, project value vs deals scatter chart, and the largest registered-vs-actual area gaps.
- **Area Insights:** sales vs project value by area, deals and deals per project by area, property subtypes, off-plan vs ready, average price per sqm, with metro, mall, landmark and area slicers.

> <img width="577" height="321" alt="image" src="https://github.com/user-attachments/assets/868ec5a7-20f0-4f69-ad39-ad0050dcac3e" />
<img width="580" height="323" alt="image" src="https://github.com/user-attachments/assets/55bce2e1-a8f7-44dc-8caf-0dd519bebba8" />
<img width="574" height="323" alt="image" src="https://github.com/user-attachments/assets/67234af6-555c-44c4-ae77-efaa7b1c9525" />
<img width="577" height="322" alt="image" src="https://github.com/user-attachments/assets/0516e09d-e69f-490b-914e-c0975876c37f" />


---

## 7. Key Findings

**Transactions**
- About **112K** units were sold between January and August 2026. About 69% were off-plan.
- Sales peaked in January (about 17.4K) and were lowest in May (about 10.3K).
- The median mortgage (about 1.36M AED) is well below the average (about 4.7M AED), because a few very large deals pull the average up.

**Projects**
- 316 projects from 200 developers, worth about 91bn AED and about 91K planned units.
- The average project takes about 35 months (almost 3 years) to build.

**Combined (212 matched projects)**
- These projects account for about **26K deals**. About **99.8% are Sales** and **92.5% are off-plan**.
- Deals peaked in April (about 4.2K), then dropped about 42% in May and stayed lower through August.
- Total sales value (about 95.4bn AED) is about **1.48 times** total project value (about 65bn AED). Sale prices include margin and land, and resales are counted too.
- **Maybach Six** is the busiest project (about 1,031 deals, roughly 55% sold). **Hado By Beyond** and **Creek Bay** are near sold out, while **Sobha Central** has sold only about a quarter.
- **Greens by Danube** has a registered-vs-actual area gap of about 241K sqm on its sales, roughly 40 times larger than the next project.

---

## 8. Limitations

- Combined results cover only the **212 projects found in both files**. Developer and status information exists only for these.
- **Sell-through %** divides units sold (which include resales) by planned units, so treat it as a rough guide, not an exact figure.
- Sales value above project value does not mean a project is profitable. Project value is a declared figure, and resales are included.
- Some deals for the matched projects are recorded under area names that do not appear in the Projects file, so area-level and project-level views can differ slightly.
- The data covers January to August 2026 only, and the two datasets come from the same source but were recorded for different purposes.

---

## 9. What I Learned

- Cleaning decisions (what to keep, what to remove, and why) matter as much as the charts.
- Why many-to-many relationships are risky, and how a lookup table gives a clean one-to-many model.
- How filters flow through a model, and why a chart can show one repeated number when its axis field sits on the wrong table.
- Why measures such as median and distinct count can tell a different story from a simple sum or average.

---

## 10. Repository Contents

```
/data        cleaned Excel workbook / source CSV notes
/dashboard   Power BI file (.pbix)
/images      dashboard screenshots
README.md
```

*Data source: Dubai Land Department open data.*
