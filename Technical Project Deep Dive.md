# Logistics Analytics Project – Technical Deep Dive

## 1. Overview

This document provides a technical walkthrough of the end-to-end analytics pipeline implemented in this project, consisting in three main steps:

- Synthetic data generation in Python;
- Relational modelling and transformations in SQL Server;
- Analytical modelling, DAX measures, and visualisation logic in Power BI.

The goal was to simulate a realistic logistics environment and demonstrate best practices in data engineering, analytics modelling, and insight generation.

2. Data Generation (Python)
   2.1 Objective

With a view to making a unique, creative project, all datasets were synthetically generated while preserving realistic logistics behaviour, particularly when it comes to:

* Supplier lead times and reliability;
* Purchase volume variability;
* Inventory inflows and outflows;
* Delivery delays.

2.2 Tools and Libraries

* pandas – data structures and export
* numpy – random distributions
* faker – realistic entity names
* random – controlled randomness

A fixed random seed (42) was used to ensure reproducibility.



\# Import libraries

import pandas as pd

import numpy as np

from faker import Faker

import random



fake = Faker()



\# Set random seed for reproducibility

np.random.seed(42)

random.seed(42)



\# Generate synthetic data tables



\# 1. SUPPLIERS



num\_suppliers = 20

suppliers = \[]



for i in range(1, num\_suppliers + 1):

&nbsp;   suppliers.append({

&nbsp;       "supplier\_id": i,

&nbsp;       "supplier\_name": fake.company(),

&nbsp;       "country": fake.country(),

&nbsp;       "lead\_time\_days": np.random.randint(5, 30),

&nbsp;       "on\_time\_rate": round(np.random.uniform(0.7, 0.99), 2),

&nbsp;       "freight\_cost": np.random.randint(200, 3000)

&nbsp;   })



df\_suppliers = pd.DataFrame(suppliers)

df\_suppliers.to\_csv("suppliers.csv", index=False)





\# 2. PRODUCTS



num\_products = 80

products = \[]



for i in range(1, num\_products + 1):

&nbsp;   products.append({

&nbsp;       "product\_id": i,

&nbsp;       "product\_name": fake.word().capitalize() + " " + fake.word().capitalize(),

&nbsp;       "category": random.choice(\["Electronics", "Furniture", "Office Supplies", "Food", "Hardware"]),

&nbsp;       "unit\_cost": round(np.random.uniform(2, 250), 2),

&nbsp;       "supplier\_id": np.random.randint(1, num\_suppliers + 1)

&nbsp;   })



df\_products = pd.DataFrame(products)

df\_products.to\_csv("products.csv", index=False)





\# 3. PURCHASE ORDERS



num\_orders = 500

purchase\_orders = \[]



for i in range(1, num\_orders + 1):

&nbsp;   order\_date = fake.date\_between(start\_date="-1y", end\_date="today")

&nbsp;   qty = np.random.randint(10, 300)

&nbsp;   

&nbsp;   purchase\_orders.append({

&nbsp;       "order\_id": i,

&nbsp;       "product\_id": np.random.randint(1, num\_products + 1),

&nbsp;       "order\_date": order\_date,

&nbsp;       "quantity\_ordered": qty,

&nbsp;       "total\_cost": round(qty \* np.random.uniform(5, 150), 2)

&nbsp;   })



df\_po = pd.DataFrame(purchase\_orders)

df\_po.to\_csv("purchase\_orders.csv", index=False)





\# 4. INVENTORY MOVEMENTS



num\_products = 80

num\_movements\_per\_product = 18

opening\_stock = 500



inventory\_movements = \[]

movement\_id = 1



for product\_id in range(1, num\_products + 1):



&nbsp;   # OPENING BALANCE

&nbsp;   start\_date = fake.date\_between(start\_date="-1y", end\_date="-11m")



&nbsp;   inventory\_movements.append({

&nbsp;       "movement\_id": movement\_id,

&nbsp;       "product\_id": product\_id,

&nbsp;       "movement\_type": "OPENING",

&nbsp;       "quantity": opening\_stock,

&nbsp;       "movement\_date": start\_date

&nbsp;   })

&nbsp;   movement\_id += 1



&nbsp;   current\_stock = opening\_stock



&nbsp;   # FUTURE MOVEMENTS

&nbsp;   movement\_dates = sorted(

&nbsp;       fake.date\_between(start\_date=start\_date, end\_date="today")

&nbsp;       for \_ in range(num\_movements\_per\_product)

&nbsp;   )



&nbsp;   for movement\_date in movement\_dates:



&nbsp;       movement\_type = random.choice(\["IN", "OUT"])



&nbsp;       if movement\_type == "IN":

&nbsp;           qty = np.random.randint(20, 150)

&nbsp;           current\_stock += qty



&nbsp;       else:  # OUT

&nbsp;           max\_out = min(120, current\_stock)

&nbsp;           if max\_out <= 0:

&nbsp;               continue



&nbsp;           qty = np.random.randint(1, max\_out + 1)

&nbsp;           current\_stock -= qty



&nbsp;       inventory\_movements.append({

&nbsp;           "movement\_id": movement\_id,

&nbsp;           "product\_id": product\_id,

&nbsp;           "movement\_type": movement\_type,

&nbsp;           "quantity": qty,

&nbsp;           "movement\_date": movement\_date

&nbsp;       })

&nbsp;       movement\_id += 1



dfinv = pd.DataFrame(inventory\_movements)

dfinv.to\_csv("inventory\_movements.csv", index=False)





\# 5. DELIVERY TIMES



delivery\_times = \[]



df\_po\_merged = df\_po.merge(df\_products\[\["product\_id", "supplier\_id"]], on="product\_id", how="left")



for idx, row in df\_po\_merged.iterrows():

&nbsp;   supplier\_id = row\["supplier\_id"]

&nbsp;   

&nbsp;   expected\_lt = int(df\_suppliers.loc\[df\_suppliers\["supplier\_id"] == supplier\_id, "lead\_time\_days"].values\[0])

&nbsp;   

&nbsp;   actual\_lt = max(1, int(np.random.normal(expected\_lt, 3)))

&nbsp;   

&nbsp;   delivery\_date = pd.to\_datetime(row\["order\_date"]) + pd.Timedelta(days=actual\_lt)

&nbsp;   

&nbsp;   delivery\_times.append({

&nbsp;       "order\_id": row\["order\_id"],

&nbsp;       "product\_id": row\["product\_id"],

&nbsp;       "supplier\_id": supplier\_id,

&nbsp;       "order\_date": row\["order\_date"],

&nbsp;       "expected\_lead\_time\_days": expected\_lt,

&nbsp;       "actual\_lead\_time\_days": actual\_lt,

&nbsp;       "delay\_days": actual\_lt - expected\_lt,

&nbsp;       "expected\_delivery\_date": pd.to\_datetime(row\["order\_date"]) + pd.Timedelta(days=expected\_lt),

&nbsp;       "actual\_delivery\_date": delivery\_date

&nbsp;   })



df\_deliveries = pd.DataFrame(delivery\_times)

df\_deliveries.to\_csv("delivery\_times.csv", index=False)



print("Synthetic supply-chain dataset created successfully!")



2.3 Core Tables Generated
Suppliers

Each supplier includes:

Lead time (days)

On-time delivery rate

Freight cost

Country of origin

These attributes later enable correlation analysis between reliability, cost, and lead time.

Products

Products were assigned:

A supplier

A product category

A unit cost

This structure supports supplier–product–category rollups in Power BI.

Purchase Orders

Purchase orders were generated with:

Random order dates over a 1-year window

Variable quantities

Calculated total cost

Each order represents a procurement event, not a shipment.

Inventory Movements

Inventory was modeled using signed quantities:

+quantity → stock inflow (IN)

-quantity → stock outflow (OUT)

Key design decisions:

An opening stock level per product was initialized

OUT movements were constrained to available stock

This prevents unrealistic negative inventory at the product level

This approach simplifies:

Running balance calculations

Inventory KPIs in DAX

Delivery Times

Delivery performance was simulated independently from purchase orders using:

Expected lead time from supplier master data

Actual lead time generated via a normal distribution

Calculated delay days

This separation reflects real systems where delivery tracking is operationally distinct from procurement.

3. SQL Server Modeling
   3.1 Database Design

All tables were loaded into a SQL Server database named Logistics.

A star schema was implemented:

Fact Tables

Fact\_Purchase\_Order

Fact\_Inventory

Dimension Tables

Dim\_Supplier

Dim\_Product

Dim\_Product\_Category

Dim\_Date

This design supports efficient analytical queries and Power BI modeling.

3.2 Views (Semantic Layer)

To simplify downstream consumption, enriched views were created.

Examples:

Product views combining supplier and category attributes

Purchase order views enriched with delivery performance

Inventory views joined to product and supplier dimensions

These views:

Encapsulate join logic

Reduce complexity in Power BI

Act as a semantic layer between raw data and reporting

3.3 Stored Procedures

Stored procedures were implemented to:

Retrieve supplier performance summaries

Support parameterized filtering (supplier, date)

This demonstrates:

Reusable SQL logic

Backend analytics capability beyond Power BI

3.4 SQL Techniques Used

Common Table Expressions (CTEs)

Window functions (running totals)

Conditional logic

Aggregations and grouping

Parameterized procedures

4. Power BI Data Model
   4.1 Model Structure

Power BI uses a clean star schema mirroring the SQL model.

Key relationships:

Fact tables connect to shared dimensions

Single-direction filtering

Date dimension drives all time intelligence

Redundant descriptive columns were removed from fact tables to avoid ambiguity and duplication.

4.2 Date Dimension

A dedicated date table was created to support:

Time intelligence

Monthly and yearly aggregation

Slicers and trend analysis

This avoids reliance on implicit date hierarchies.

5. DAX Measures \& KPIs
   5.1 Inventory Logic

Because inventory quantities are signed:

SUM(quantity) = net movement

Running balances are calculated via date-aware measures

Closing Stock Balance

Returns inventory as of the selected date context.

Average Daily OUT Movements

Calculates average daily consumption using:

Date-aware iteration

Only negative quantities (OUT)

Days of Inventory Coverage

Indicates how long current stock would last at current consumption rates.

5.2 Performance KPIs

Total Spend

Average Delay (days)

Average On-Time Rate

% Days Below Safety Stock

These KPIs dynamically respond to slicers and cross-filtering.

6. Visual Design \& Interactivity
   6.1 Dashboard Structure

The report is split into logical sections:

Main dashboard – operational overview

Supplier analysis – performance trade-offs

Correlation analysis – relationship exploration

6.2 Visual Techniques Used

Waterfall charts for spend decomposition

Line and area charts for trends

Scatter plots for correlation analysis

Conditional formatting for risk highlighting

Tooltips for contextual detail

Bookmarks for filter control

6.3 Design Decisions

Negative inventory visually highlighted

Safety stock thresholds explicitly marked

Drill-downs limited to avoid over-filtering confusion

Analytical visuals separated from KPI dashboards

7. Key Analytical Takeaways

From a technical perspective, the project demonstrates:

End-to-end ownership of the data lifecycle

Proper dimensional modeling

Correct handling of time-dependent metrics

Awareness of DAX evaluation context pitfalls

Thoughtful visual and interaction design

8. Limitations \& Assumptions

Data is synthetic and illustrative

Safety stock thresholds are assumed

No real operational constraints (MOQ, batching, contracts)

These limitations are explicitly acknowledged to avoid over-interpretation.

9. Potential Enhancements

Reorder point simulation

Forecast-based consumption modeling

Supplier risk scoring

ABC / XYZ inventory segmentation

10. Conclusion

This project demonstrates a production-style analytics workflow, from data generation through modeling to insight communication.

It emphasizes:

Analytical correctness

Modeling discipline

Business relevance

Clear storytelling

