# Expedia Data Warehouse Design & Business Analytics

## 📌 Overview

This project presents a comprehensive data warehouse design for an Expedia-style travel booking platform. The model captures the customer journey from searching for travel options and viewing listings to booking hotels, flights, and rental cars, followed by payments, cancellations, refunds, and revenue analysis.

The project focuses on dimensional data modeling, fact and dimension table relationships, Slowly Changing Dimension Type 2 (SCD Type 2), and SQL-based business analytics.

The data warehouse is designed to support reporting and analysis of booking performance, customer behavior, marketing channels, payment activity, and commercial revenue.

> **Disclaimer:** This is an independent educational portfolio project inspired by an Expedia-style travel platform. It is not Expedia's official data model and does not represent its internal architecture.

## 🎯 Project Objectives

- Design a relational data warehouse model for travel booking analytics.
- Identify fact tables, dimension tables, primary keys, and foreign key relationships.
- Model the customer journey from search and listing interactions to bookings and post-booking events.
- Maintain historical customer and property attributes using SCD Type 2 dimensions.
- Support hotel, flight, and car rental booking analysis.
- Develop SQL queries for business metrics and financial reporting.
- Analyze booking conversion, click-through rates, gross booking value, net revenue, cancellations, refunds, and customer retention.
- Support marketing channel performance and analytical reporting.

## 📁 Repository Structure

```text
Expedia-DWH-Design/
│
├── README.md
│
├── 01_Docs/
│   └── DWH_Data_Modelling_Class_2.pdf
│
└── 02_Architecture/
    └── ExpediaDWH.png
```

### 📄 File Description

- **README.md** — Provides the project overview, objectives, data architecture, fact and dimension tables, and business metrics.
- **01_Docs/DWH_Data_Modelling_Class_2.pdf** — Contains the reference documentation for the Expedia data warehouse design, table definitions, relationships, and SQL queries for business metrics.
- **02_Architecture/ExpediaDWH.png** — Contains the final Expedia data warehouse ER diagram, illustrating tables, attributes, and relationships between entities.

## 🏗️ Data Architecture (Star Schema)

The data warehouse uses a dimensional modeling approach with fact tables and dimension tables connected through key relationships.

Dimension tables provide descriptive context, while fact tables record business events and measurable metrics. The model includes shared dimensions for dates, time, geography, customers, devices, and marketing channels, alongside travel-product-specific dimensions and transactional facts.

### Dimension Tables (Context & Attributes)

Dimension tables store descriptive information about business entities and events.

| Dimension table | Purpose |
|---|---|
| `dim_date` | Calendar attributes such as year, quarter, month, week, day, and calendar flags. One row per calendar day. |
| `dim_time` | Time attributes including hour, minute, second, AM/PM, and day part. Designed for minute- or second-level granularity. |
| `dim_geo` | Country, region, state, city, market, timezone, latitude, and longitude. |
| `dim_device` | Device type, operating system, app or web platform, browser, and versions. |
| `dim_channel` | Marketing source, medium, campaign, ad group, advertisement, and UTM attributes. |
| `dim_customer` | Customer details, hashed contact information, loyalty tier, preferences, and marketing opt-in. Uses SCD Type 2 history tracking. |
| `dim_property` | Hotel/property name, brand, chain status, star rating, geography, room capacity, and check-in/check-out times. Uses SCD Type 2. |
| `dim_room` | Room type, bed type, occupancy capacity, smoking policy, and view type. |
| `dim_flight` | Airline carrier, route, cabin, refundability, and baggage policy. |
| `dim_car` | Rental car supplier, class, transmission, fuel type, and seating capacity. |
| `dim_promo` | Promotion code, discount type and value, eligibility rules, and validity dates. |
| `dim_payment_method` | Payment method, card network, issuer country, and 3-D Secure support. |
| `dim_currency_rate` | Daily exchange rates, currency code, conversion factors, source, and timestamp. |

### Fact Tables (Events & Metrics)

Fact tables capture measurable events throughout the travel booking lifecycle.

| Fact table | Grain and purpose |
|---|---|
| `fact_search_session` | One row per search session, recording customer, device, geography, channel, destination, travel dates, passenger counts, and search results. |
| `fact_impression` | One row per listing shown during a search session, including product, position, displayed price, and impressions. |
| `fact_click` | One row per listing click, recording the search session, product, channel, displayed price, and clicks. |
| `fact_booking_item` | One row per booked item, such as a room-night bundle, flight-leg bundle, or car rental item. Captures GBV, discounts, taxes, fees, ancillaries, and net revenue attributes. |
| `fact_cancellation` | One row per item-level cancellation event, including cancellation reason, fees, recoverable amount, and refundability. |
| `fact_payment` | One row per payment transaction, including authorization, capture, refund amounts, transaction type, and payment timestamp. |
| `fact_refund` | One row per refund instruction or settlement, recording refund amount, reason, and chargeback flag. |

### Key Modeling Concepts

**1. Primary and foreign keys**

Primary keys identify records within tables. Foreign keys connect fact tables to their related dimensions and connect post-booking events to booking items.

**2. Slowly Changing Dimension Type 2**

The customer and property dimensions include historical tracking attributes such as `effective_from`, `effective_to`, and `is_current`. This allows historical versions of customer or property records to be retained.

**3. Role-playing date dimension**

The date dimension is referenced for different business events, including booking, search travel dates, cancellation, refund, and property or travel start and end dates.

**4. Travel product coverage**

The booking fact supports hotel, flight, and rental car items using product-specific dimension keys and the `product_type` attribute.

## 📊 SQL Implementations for Business Metrics

The following metrics are defined in the project documentation. Each query uses `:start_date` and `:end_date` as reporting-period parameters where applicable.

### 1. Search-to-Booking Conversion Rate

**Definition:** The percentage of unique search sessions compared with distinct bookings during the reporting period.

**Formula:** Distinct bookings ÷ distinct search sessions × 100.

```sql
WITH searches AS (
    SELECT COUNT(DISTINCT fs.search_session_id_nk) AS search_sessions
    FROM fact_search_session fs
    JOIN dim_date d ON fs.date_key = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
),
bookings AS (
    SELECT COUNT(DISTINCT bi.booking_id_nk) AS bookings
    FROM fact_booking_item bi
    JOIN dim_date d ON bi.date_key_booked = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
)
SELECT
    s.search_sessions,
    b.bookings,
    ROUND(
        b.bookings * 100.0 / NULLIF(s.search_sessions, 0), 2
    ) AS conversion_rate_percent
FROM searches s
CROSS JOIN bookings b;
```

### 2. Click-Through Rate (Listings)

**Definition:** The percentage of listing impressions that received a click.

**Formula:** Total clicks ÷ total impressions × 100.

```sql
WITH imps AS (
    SELECT COUNT(*) AS impressions
    FROM fact_impression fi
    JOIN dim_date d ON fi.date_key = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
),
clks AS (
    SELECT COUNT(*) AS clicks
    FROM fact_click fc
    JOIN dim_date d ON fc.date_key = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
)
SELECT
    i.impressions,
    c.clicks,
    ROUND(
        c.clicks * 100.0 / NULLIF(i.impressions, 0), 2
    ) AS click_through_rate_percent
FROM imps i
CROSS JOIN clks c;
```

### 3. Gross Booking Value (GBV)

**Definition:** The total pre-discount, pre-refund value of booked items in the reporting period.

**Formula:** Sum of `gbv_amount`.

```sql
SELECT
    SUM(bi.gbv_amount) AS gross_booking_value
FROM fact_booking_item bi
JOIN dim_date d ON bi.date_key_booked = d.date_key
WHERE d.cal_date BETWEEN :start_date AND :end_date;
```

### 4. Net Revenue

**Definition:** Commercial revenue from bookings, including take rate, fees, and ancillaries, less refunds.

**Formula:** Sum of take-rate revenue, fees, and ancillary revenue, minus refunds.

```sql
SELECT
    SUM(
        bi.gbv_amount * bi.take_rate_pct
        + bi.fee_amount
        + bi.ancillary_amount
    ) - COALESCE(SUM(r.refund_amount), 0) AS net_revenue
FROM fact_booking_item bi
LEFT JOIN fact_refund r
    ON r.booking_item_id_nk = bi.booking_item_id_nk
JOIN dim_date d
    ON bi.date_key_booked = d.date_key
WHERE d.cal_date BETWEEN :start_date AND :end_date;
```

### 5. Average Selling Price (ASP)

**Definition:** The average pre-discount value per booked item.

**Formula:** Gross booking value ÷ number of booked items.

```sql
SELECT
    COUNT(*) AS booking_items,
    SUM(bi.gbv_amount) AS gbv,
    ROUND(
        SUM(bi.gbv_amount) / NULLIF(COUNT(*), 0), 2
    ) AS average_selling_price
FROM fact_booking_item bi
JOIN dim_date d ON bi.date_key_booked = d.date_key
WHERE d.cal_date BETWEEN :start_date AND :end_date;
```

### 6. Cancellation Rate

**Definition:** The percentage of booked items that were cancelled during the reporting period.

**Formula:** Cancelled items ÷ booked items × 100.

```sql
WITH booked AS (
    SELECT COUNT(*) AS booked_items
    FROM fact_booking_item bi
    JOIN dim_date d ON bi.date_key_booked = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
),
cancelled AS (
    SELECT COUNT(DISTINCT fc.booking_item_id_nk) AS cancelled_items
    FROM fact_cancellation fc
    JOIN dim_date d ON fc.date_key_cancel = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
)
SELECT
    b.booked_items,
    c.cancelled_items,
    ROUND(
        c.cancelled_items * 100.0 / NULLIF(b.booked_items, 0), 2
    ) AS cancellation_rate_percent
FROM booked b
CROSS JOIN cancelled c;
```

### 7. Refund Rate

**Definition:** The percentage of booked items with at least one refund recorded during the reporting period.

**Formula:** Refunded items ÷ booked items × 100.

```sql
WITH booked AS (
    SELECT COUNT(DISTINCT bi.booking_item_id_nk) AS booked_items
    FROM fact_booking_item bi
    JOIN dim_date d ON bi.date_key_booked = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
),
ref_items AS (
    SELECT COUNT(DISTINCT r.booking_item_id_nk) AS refunded_items
    FROM fact_refund r
    JOIN dim_date d ON r.date_key_refund = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
)
SELECT
    b.booked_items,
    r.refunded_items,
    ROUND(
        r.refunded_items * 100.0 / NULLIF(b.booked_items, 0), 2
    ) AS refund_rate_percent
FROM booked b
CROSS JOIN ref_items r;
```

### 8. Ancillary Attachment Rate

**Definition:** The percentage of booked items that include an ancillary amount, such as baggage or breakfast.

**Formula:** Items with ancillary amount greater than zero ÷ total booked items × 100.

```sql
SELECT
    COUNT(*) AS booking_items,
    SUM(
        CASE WHEN bi.ancillary_amount > 0 THEN 1 ELSE 0 END
    ) AS items_with_ancillary,
    ROUND(
        SUM(CASE WHEN bi.ancillary_amount > 0 THEN 1 ELSE 0 END)
        * 100.0 / NULLIF(COUNT(*), 0), 2
    ) AS ancillary_attachment_rate_percent,
    SUM(bi.ancillary_amount) AS ancillary_revenue
FROM fact_booking_item bi
JOIN dim_date d ON bi.date_key_booked = d.date_key
WHERE d.cal_date BETWEEN :start_date AND :end_date;
```

### 9. Take Rate Percentage

**Definition:** Net revenue expressed as a percentage of gross booking value.

**Formula:** Net revenue ÷ GBV × 100.

```sql
WITH base AS (
    SELECT
        SUM(bi.gbv_amount) AS gbv,
        SUM(
            bi.gbv_amount * bi.take_rate_pct
            + bi.fee_amount
            + bi.ancillary_amount
        ) - COALESCE(SUM(r.refund_amount), 0) AS net_revenue
    FROM fact_booking_item bi
    LEFT JOIN fact_refund r
        ON r.booking_item_id_nk = bi.booking_item_id_nk
    JOIN dim_date d ON bi.date_key_booked = d.date_key
    WHERE d.cal_date BETWEEN :start_date AND :end_date
)
SELECT
    gbv,
    net_revenue,
    ROUND(
        net_revenue * 100.0 / NULLIF(gbv, 0), 2
    ) AS take_rate_percent
FROM base;
```

### 10. Repeat Customer Rate (12-Month Lookback)

**Definition:** The percentage of customers with at least two distinct bookings during the last 12 months.

**Formula:** Customers with at least two bookings ÷ customers with at least one booking × 100.

```sql
WITH cust_trips AS (
    SELECT
        bi.customer_sk,
        COUNT(DISTINCT bi.booking_id_nk) AS bookings_12m
    FROM fact_booking_item bi
    JOIN dim_date d ON bi.date_key_booked = d.date_key
    WHERE d.cal_date BETWEEN date_sub(current_date(), 365)
                         AND current_date()
    GROUP BY bi.customer_sk
)
SELECT
    COUNT(*) AS customers_with_bookings,
    SUM(
        CASE WHEN bookings_12m >= 2 THEN 1 ELSE 0 END
    ) AS repeat_customers,
    ROUND(
        SUM(CASE WHEN bookings_12m >= 2 THEN 1 ELSE 0 END)
        * 100.0 / NULLIF(COUNT(*), 0), 2
    ) AS repeat_customer_rate_percent
FROM cust_trips;
```

### 11. Channel Performance (Net Revenue by Marketing Channel)

**Definition:** Net revenue contribution by marketing source, medium, and campaign.

**Metrics:** Gross booking value, net revenue, and take rate percentage.

```sql
SELECT
    ch.source,
    ch.medium,
    ch.campaign,
    SUM(bi.gbv_amount) AS gross_booking_value,
    SUM(
        bi.gbv_amount * bi.take_rate_pct
        + bi.fee_amount
        + bi.ancillary_amount
    ) - COALESCE(SUM(r.refund_amount), 0) AS net_revenue,
    ROUND(
        (
            SUM(
                bi.gbv_amount * bi.take_rate_pct
                + bi.fee_amount
                + bi.ancillary_amount
            ) - COALESCE(SUM(r.refund_amount), 0)
        ) * 100.0 / NULLIF(SUM(bi.gbv_amount), 0), 2
    ) AS take_rate_percent
FROM fact_booking_item bi
JOIN dim_channel ch ON bi.channel_sk = ch.channel_sk
LEFT JOIN fact_refund r
    ON r.booking_item_id_nk = bi.booking_item_id_nk
JOIN dim_date d ON bi.date_key_booked = d.date_key
WHERE d.cal_date BETWEEN :start_date AND :end_date
GROUP BY ch.source, ch.medium, ch.campaign
ORDER BY net_revenue DESC;
```

## 🛠️ Tools & Technologies

- **Data Modeling:** Entity Relationship Diagrams, dimensional modeling, fact and dimension tables.
- **SQL:** Joins, aggregations, Common Table Expressions (CTEs), conditional expressions, date functions, and analytical calculations.
- **Data Warehousing Concepts:** Grain definition, surrogate keys, foreign key relationships, role-playing dimensions, and SCD Type 2.
- **Business Analytics:** Conversion rate, click-through rate, GBV, net revenue, cancellation rate, refund rate, and marketing channel analysis.
- **Version Control:** Git and GitHub for project documentation and source-code management.

*These technologies describe the design and SQL work documented in this project; they do not imply that a production warehouse has been deployed.*

## 🚀 How to Use This Repository

1. Review `ExpediaDWH.png` to understand the complete data model and table relationships.
2. Refer to the data architecture section to understand fact and dimension tables.
3. Open the SQL scripts to review the business metric definitions and queries.
4. Replace `:start_date` and `:end_date` with the appropriate reporting parameters for your SQL environment.
5. Execute the queries against a database containing the required tables and compatible column definitions.
6. Validate query results against sample data before using them for reporting.

## 🔮 Future Enhancements

- Add SQL DDL scripts to create the tables.
- Create sample datasets for the fact and dimension tables.
- Build a data dictionary documenting table grains, columns, and relationships.
- Add data quality and referential integrity validation queries.
- Review revenue calculations to avoid double-counting when a booking item has multiple refund records.
- Validate conversion metrics using an explicit relationship between search sessions and bookings.
- Develop dashboards to visualize booking and revenue performance.

## 👨‍💻 Author

**Shyam Madeshiya**

Aspiring Data Engineer interested in data warehousing, SQL, data modeling, and analytical data solutions.

