
### Late Fees Report Analysis of PostgreSQL Database for DVD Rental Company

This report answers the question of how much of revenue from payments of rentals are due to late fees and compare that with how much maximal revenue lost due to having films returned late based on the data collected of movies rented from May 24, 2005 to February 14, 2006. The dataset shows that 7,269 of 16,044, a whopping 45.3%, of the rentals placed during this timeframe were returned late (not including the 1 rental that was not returned at all -- Academy Dinosaur). In this report we calculate the number of days each late return was, and multiply that by the per-day-rental cost of that film as a baseline for maximal revenue lost. The report concludes that late fees represent a figure from 25-33% of total revenue from rentals and the figure seems to be steadily increasing over time. It may be worth looking into whether or not this is a sustainable business practice as that is a significant proportion of revenue that could theoretically drop if consumers start returning rentals on time. Furthermore it seems tha the late fees charged represent more than the rentals could have made had they been rented out to other consumers if they were returned on time. It may be of interest to stakeholders to determine what competitors late fee rates are and to ensure that this is not a liability for losing consumers. For future analyses, payment date may be of great consideration to include in the late fee as well due to 6,623 of the 7,269 late returns having payments over half a year after the returns were made. Also this report chooses to aggregate a monthly basis, averaging out weekly fluctuations so that stakeholders may observe overarching trends and plan their response.


Detailed Table: **late_fees_detailed**
| Field Name | Field Type | Definition | Details |
|------------ | ------------| ----------- | - |
| rental_id | INTEGER  | The rental_id used to reference each rental. | PRIMARY KEY and FOREIGN KEY References rental(rental_id)|
| yearmonth_return_received | VARCHAR(15) | Year and month the rental was due to be returned.| 
| days_returned_late | INTEGER | How many days after the calculated return date was the rental actually returned. |
| film_title | VARCHAR(255) | The title of the film rented. |
| daily_rate | DECIMAL(4, 2) | The calculated daily rate of the rental: rental_rate / rental_duration. |
| maximum_revenue_lost | DECIMAL(10, 2) | The calculated daily_rate multiplied by the number of days late. |
| actual_late_fee  | DECIMAL(5,2) | The difference between the total payments on the rental and the rental rate. |


Summary Table: **late_fees_summary**
| Field Name | Field Type | Definition | Details |
|------------|------------|----------|--------|
| yearmonth | VARCHAR(15) | Year and month combine for identifying time period which info was aggregated on. | PRIMARY KEY |
| late_returns_count | INTEGER | Total number of late returns during this time period. |
| total_potential_maximum_revenue_lost | DECIMAL (12, 2) | Total value of potential revenue lost accumulated.|
| total_late_fee |  DECIMAL(12,2) | Total accumulated late fees on rentals returns due on the time period.
| total_revenue |  DECIMAL(12,2) | Total revenue on rentals due to be returned on the time period both late and not late, used for comparison. |


The report includes the types: **DECIMAL**,  **INT**, **VARCHAR**.
The specifics of each field for each type are listed in the table above under the previous section A1. 
The data from the DVD Database used to compute the new fields in the report are of the types: **DATETIME**, **DECIMAL**, **INT**, **VARCHAR**.


The detailed table section uses the following fields in the format table.field_name: 
film.title, film.id, film.rental_rate, film.rental_duration, 
inventory.inventory_id, inventory.film_id, 
payment.rental_id, payment.amount
rental.rental_id, rental.inventory_id, rental.return_date.

The summary table section aggregates the fields from the detailed table, so it ends up using the same fields as the detailed table just indirectly.


The field yearmonth_rental_received will require custom transformation with a user-defined function. They will transform **DATETIME** fields of rental.return_date into **VARCHAR** fields that are more quickly readable for stakeholders.


The field days_returned_late required a calculation involving computing a due date from a rental date and a rental duration as well as the number of days between the due date and date returned. This was required to calculate the maximum amount of money lost from the rate being late assuming the film was rented out during the time it was late.


The field actual_late_fee required a subquery on the payments table because there was at least one rental with multiple payments in order to aggregate all payments on the rental into a singular value. It made sense to create a function for this for code readability.


The detailed table late_fees_detailed shows the information on a per rental basis of which film was returned late, when it was returned, when the payment was received, and how much revenue could've been made had that film been rented out on the days while it was late. This could be analyzed for trends on which films tend to be late, looking for patterns in the number of days that films tend to be late for, and also could be used as a starting point for a discussion on implementing fees for late payments as well.


The summary table late_fees_summary includes summarized information that can show trends from month to month of number of total late returns and total potential revenue lost. This information can help determine whether there should be a flat rate for late returns or daily rate based on the film or some combination.


This report should be refreshed on a monthly basis for multiple reasons: firstly to determine how much additional revenue the company could be making if a late fee were to be instantiated, and secondly the script could then be adjusted on a monthly basis to determine how much late fee revenue is actually being generated to determine if the late fee would need adjustments. Given the high frequency of rentals being returned late this could have a potential to increase revenue significiantly or consumers might adjust their behavior which would allow the films to be rented more frequently, which in turn would also increase revenue.


### Function to extract year and month from a timestamp for the Late Fees Report
```
CREATE OR REPLACE FUNCTION extract_yearmonth(return_date TIMESTAMP)
RETURNS VARCHAR(15)
LANGUAGE plpgsql
AS $$
DECLARE extracted_month VARCHAR(9);
DECLARE extracted_year VARCHAR(4);
BEGIN
	extracted_month := to_char(return_date, 'Month');
	extracted_year := to_char(return_date, 'YYYY');
	RETURN CONCAT(extracted_year, ' ', extracted_month);
END;
$$;
```

### Creates detailed and summary tables for Late Fees Report
```
CREATE TABLE late_fees_detail(
rental_id INT PRIMARY KEY,
yearmonth_return_received VARCHAR(15),
days_returned_late INT,
film_title VARCHAR(255),
days_paid_late INT,
daily_rate DECIMAL(4,2),
maximum_revenue_lost DECIMAL(10,2),
FOREIGN KEY(rental_id) REFERENCES rental(rental_id)
);

CREATE TABLE late_fees_summary(
yearmonth VARCHAR(15),
late_returns_count INT,
total_potential_maximum_revenue_lost DECIMAL(10,2),
PRIMARY KEY (yearmonth)
);
```

### Calculate and record data for actual late fees and maximum revenue lost
```
INSERT INTO late_fees_detail
SELECT
	r.rental_id,
	extract_yearmonth(r.return_date) AS yearmonth_return_received,
	calc_date_diff(r.return_date, calc_due_date(r.rental_date, f.rental_duration)) AS days_returned_late,
	f.title AS film_title,
	CAST(f.rental_rate/f.rental_duration AS DECIMAL(10,2)) AS daily_rate,
	CAST(calc_date_diff(r.return_date, calc_due_date(r.rental_date, f.rental_duration)) * f.rental_rate/f.rental_duration AS DECIMAL(10,2)) AS maximum_revenue_lost,
	total_payment(r.rental_id) - f.rental_rate AS actual_late_fee
FROM 
	rental AS r
INNER JOIN
	inventory AS i ON r.inventory_id = i.inventory_id
INNER JOIN
	film as f on i.film_id = f.film_id
WHERE
	calc_due_date(r.rental_date, f.rental_duration) < r.return_date
ORDER BY
	extract_yearmonth(r.return_date),
	actual_late_fee DESC;
```


### Trigger to Update the Summary Table After Inserting into the Detailed Table
```
CREATE OR REPLACE FUNCTION update_summary_trigger()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
TRUNCATE late_fees_summary;

INSERT INTO late_fees_summary
SELECT
	yearmonth_return_received AS yearmonth,
	COUNT(*) AS late_returns_count,
	COALESCE(SUM(maximum_revenue_lost),0) AS total_potential_maximum_revenue_lost,
	COALESCE(SUM(actual_late_fee), 0) AS total_late_fee,
	COALESCE(calc_total_revenue(yearmonth_return_received), 0) AS total_revenue
FROM
	late_fees_detail
GROUP BY
	yearmonth_return_received
ORDER BY
	TO_DATE(SUBSTRING(yearmonth_return_received, 0, 4), 'YYYY') DESC,
	TO_DATE(SUBSTRING(yearmonth_return_received, 5, 14), 'Month') DESC;
RETURN NEW;
$$;

CREATE TRIGGER update_summary
AFTER INSERT
ON late_fees_detail
FOR EACH STATEMENT
EXECUTE PROCEDURE update_summary_trigger();
```

### Stored Procedure for Refreshing the Report Tables
```
CREATE OR REPLACE PROCEDURE refresh_summary_and_detailed()
LANGUAGE plpgsql
AS $$
BEGIN
TRUNCATE late_fees_detail;

INSERT INTO late_fees_detail
SELECT
	r.rental_id,
	extract_yearmonth(r.return_date) AS yearmonth_return_received,
	calc_date_diff(r.return_date, calc_due_date(r.rental_date, f.rental_duration)) AS days_returned_late,
	f.title AS film_title,
	CAST(f.rental_rate/f.rental_duration AS DECIMAL(10,2)) AS daily_rate,
	CAST(calc_date_diff(r.return_date, calc_due_date(r.rental_date, f.rental_duration)) * f.rental_rate/f.rental_duration AS DECIMAL(10,2)) AS maximum_revenue_lost,
	total_payment(r.rental_id) - f.rental_rate AS actual_late_fee
FROM 
	rental AS r
INNER JOIN
	inventory AS i ON r.inventory_id = i.inventory_id
INNER JOIN
	film as f on i.film_id = f.film_id
WHERE
	calc_due_date(r.rental_date, f.rental_duration) < r.return_date
ORDER BY
	extract_yearmonth(r.return_date),
	actual_late_fee DESC;
RETURN;
END;
$$;
```

#### Job Scheduling Tool Recommendation

For this use case of a monthly update on the dvd rental data set the tool pgAgent would be recommended as it has a user-friendly graphical user interface which can be accessed through PgAdmin 4, which was the tool used to test the code in this report. There are other tools that could also work but are more geared towards those who have linux experience such as IT professionals or software engineers, which one would expect in a different setting such as a data centers.

Following Hugo Dias's guide (which also includes installation steps which are not listed here), we would create a new scheduling job to run the stored procedure refresh_summary_and_detailed to run once a month. We would need to ensure that the agent is running in the background by manually setting the host, database name, user, and port parameters for pgAgent. [1] I personally would have it run on at midnight on the first Monday of the month to ensure that the data tables are refreshed for analysts to view. Values may change as late payments may come in after the month of which the rental was due which should be of note to the analysts. 

### Sources Used

1. Dias, H. (2020, February 3). An Overview of Job Scheduling Tools for PostgreSQL. Severalnines. https://severalnines.com/blog/overview-job-scheduling-tools-postgresql/

2. 9.8. Data Type Formatting Functions. (2023, February 9). PostgreSQL Documentation. https://www.postgresql.org/docs/current/functions-formatting.html

3. adura826adura826, et al. (2020, July 28). DATE ADD function in PostgreSQL. Stack Overflow. https://www.stackoverflow.com/a/63142013

4. 43.10. Trigger Functions. (2023, May 11). PostgreSQL Documentation. https://www.postgresql.org/docs/current/plpgsql-trigger.html

5. Neon. (2024, March 22). PostgreSQL TO_DATE() Function. Neon. https://neon.tech/postgresql/postgresql-date-functions/postgresql-to_date
