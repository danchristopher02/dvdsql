# DVD Rental Business Report Project

This project demonstrates how to extract raw data from a larger DVD rental dataset to generate a business report. This report identifies which DVD categories are most popular at each store, helping management optimize inventory and improve customer satisfaction.

## Business Problem

The goal of this project was to create a report that identifies the most popular DVD categories at each store. By increasing the stock of popular categories and reducing the stock of unpopular ones, management can optimize inventory to meet customer demand while saving storage costs.

## Report Overview

The report involves two primary tables that will be made from extracting the raw data from the larger DVD Rental dataset:

1. **Detailed Table**:
   - Provides records of all rentals made, including rental ID, store, and category name.
   - Allows for identification of which specific rentals were made at which store, and in which category.

2. **Summary Table**:
   - Aggregates data to show the total number of rentals per category at each store.
   - Helps identify trends in DVD category popularity at the store level.
  

## Youtube Video of me explaining my SQL code
[Video Explanation](https://www.youtube.com/watch?v=nR9F7SjP1s8&t=348s)

## Data and Fields

### Detailed Table Fields

- **store**: `VARCHAR(7)` – The store where the rental occurred (e.g., "Store 1").
- **category_name**: `VARCHAR(25)` – The name of the category for the rented DVD.
- **rental_id**: `INT` – A unique id for each rental.

### Summary Table Fields

- **store**: `VARCHAR(7)` – The store where the rental occurred.
- **category_name**: `VARCHAR(25)` – The name of the category for the rented DVD.
- **rentals_sold**: `INT` – The total number of rentals for a specific category at a specific store.

## Tables Used

The following tables from the larger DVD dataset were used:

- **Staff Table**: Provides `store_id` to identify where the rental occurred.
- **Category Table**: Provides `name` for the DVD categories.
- **Rental Table**: Provides `rental_id` to track individual rentals.

## SQL Code

### Store Name Transformation

To make the store names more user-friendly, a user-defined function `fn_store_name` was created to transform the `store_id` from the **Staff Table** into a `VARCHAR(7)` format. The function concatenates the string "Store " with the `store_id`, making it more readable.

```sql
CREATE OR REPLACE FUNCTION fn_store_name(store_num SMALLINT)
RETURNS VARCHAR(7)
LANGUAGE plpgsql
AS
$$
DECLARE store VARCHAR(7);
BEGIN
    SELECT CONCAT ('Store ', CAST (store_num AS VARCHAR(7))) INTO store;
    RETURN store;
END;
$$;
```

### SQL Code for Detailed and Summary Tables

To store the report data, the following tables were created:

```sql
CREATE TABLE detailed (
    Store            VARCHAR(7),
    Category_Name    VARCHAR(25),
    Rental_ID        INT
);

CREATE TABLE summary (
    Store            VARCHAR(7),
    Category_Name    VARCHAR(25),
    Rentals_Sold     INT
);
```

### Query to Extract Data for the Detailed Table

The raw data needed for the detailed table was extracted using the following SQL query, which joins multiple tables to retrieve the necessary information:

```sql
INSERT INTO detailed
SELECT fn_store_name(s.store_id), c.name, r.rental_id
FROM staff s
RIGHT JOIN rental r ON s.staff_id = r.staff_id
LEFT JOIN inventory i ON r.inventory_id = i.inventory_id
LEFT JOIN film f ON i.film_id = f.film_id
LEFT JOIN film_category fc ON f.film_id = fc.film_id
LEFT JOIN category c ON fc.category_id = c.category_id;
```

### Trigger to Update the Summary Table

A trigger was implemented to automatically update the summary table whenever new data is inserted into the detailed table:

```sql
CREATE OR REPLACE FUNCTION insert_trigger_function()
RETURNS TRIGGER
LANGUAGE plpgsql
AS
$$
BEGIN
    DELETE FROM summary;
    INSERT INTO summary
    SELECT store, category_name, COUNT(rental_id) AS rentals_sold
    FROM detailed
    GROUP BY store, category_name
    ORDER BY store, rentals_sold DESC;
    RETURN NEW;
END;
$$;

CREATE OR REPLACE TRIGGER new_detailed
AFTER INSERT
ON detailed
FOR EACH STATEMENT
EXECUTE PROCEDURE insert_trigger_function();
```

### Stored Procedure to Refresh Tables

A stored procedure was created to refresh the data in both the detailed table and the summary table. This procedure clears the contents of both tables and reloads the data:

```sql
CREATE OR REPLACE PROCEDURE refresh_tables()
LANGUAGE plpgsql
AS
$$
BEGIN
    DELETE FROM detailed;
    DELETE FROM summary;

    INSERT INTO detailed
    SELECT fn_store_name(s.store_id), c.name, r.rental_id
    FROM staff s
    RIGHT JOIN rental r ON s.staff_id = r.staff_id
    LEFT JOIN inventory i ON r.inventory_id = i.inventory_id
    LEFT JOIN film f ON i.film_id = f.film_id
    LEFT JOIN film_category fc ON f.film_id = fc.film_id
    LEFT JOIN category c ON fc.category_id = c.category_id;

    RETURN;
END;
$$;
```

### Refresh Schedule

To automate the refresh process, I recommend using pgAgent, a job scheduling tool for PostgreSQL. This tool can automatically run the refresh_tables() procedure on a schedule, such as once a day. It can be installed via StackBuilder on Windows, and once configured, a pgAgent job can be set up in pgAdmin to run the procedure at the desired interval.

The report is designed to be refreshed daily to ensure the data remains relevant. Given the high rental activity (average of 391 rentals per day), this will ensure that stakeholders have access to the most up-to-date information.

### Business Use Cases

Detailed Table:

- Used to track specific rental transactions, identifying where and in which category each rental was made.

![image](https://github.com/user-attachments/assets/27daac96-3c2b-4bf4-bc74-16e747385460)


Summary Table:

- Used to analyze which DVD categories are most popular at each store.
- Provides strategic insights into inventory management and sales trends.

![image](https://github.com/user-attachments/assets/1bee6323-60b1-4c73-b1ee-8b37e34ab15b)
