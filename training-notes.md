Using this project to learn PowerBI and SQL queries. Start with CSV, move to PostGres, to learn in smaller steps

Pages:
page_id (number)
site_section (Text)
page_type (Text)
last_reviewed_date (Date)
compliance_status (Text)

Issues:
issue_id (number)
page_id (number) - match to pages data
issue_type (Text)
severity (Text)
identified_date (Date)
resolved_date (Date or blank)

Reviews:
review_id (number)
page_id (number)  - match to pages data
review_type (Text)
reviewer_role (Text)
review_date (Date)
outcome (Text)


========================


Before we start: DAX =/= SQL. DAX is PowerBI's way of applying functions/filters to data. SQL queries could still be used on datasets before those sets are pulled into PowerBI. What we're learning first with the CSVs is just the PowerBI/DAX side of things, then we'll move on to SQL later.

There are generally two ways we can approach how to work with data and PowerBI:

Workflow A – SQL-heavy teams
- Write SQL queries or create database views.
- Use Power BI mostly for visuals and light DAX.
- Complex logic lives in SQL.

Workflow B – BI-heavy teams
- Pull fairly raw tables into Power BI.
- Build most logic in DAX. Logic lives in DAX.
- Use SQL only for basic extraction.

And that decision usually depends on:
- Scale
- Performance
- Governance
- Skillset of the team
- Reusability

But probably not capability. Both are capable of similar kinds of logic and logical complexity.

For example, teams might prefer to have logic live in SQL when:
- The logic applies to many reports
- It should be centralized and governed
- Performance matters
- The dataset is large
- Data engineering owns transformation

So, generally, I think I'm in the camp of "use SQL for the logic", even just as a means of future-proofing against complexity and scaling. However, it's worth noting that if the logic isn't actually reusable between datasets/dashboards/tools, then it doesn't really matter whether it's in SQL or DAX/tool-specific. The key is reusability.

If our data/logic is or needs to be:
- interactive
- context-sensitive
- driven by slicers
- dependent on report-level filters
…it probably belongs in DAX (or whatever the specific dashboard/report tool uses).

Also worth noting that keeping the logic in DAX forced me to create a whole table just to configure date logic, and we still ended up with a mid-final-month tail at 12/22 instead of 12/31 (causing december to calculate 14 backlogged issues instead of 15 since the month was still "in progress"). Neither of those was an issue when doing the same logic in SQL/backend. With more time spent tweaking, there may be ways of resolving those filtering issues through DAX, but just doing it in SQL instead didn't require any additional filter negotiations. Once again confirming, at a general level, that shaping data *before* it hits the front end is just a very nice workflow.


========================

Learning notes:

A measure is not a stored value. It’s a calculation that runs dynamically based on the current filter context.

Filter context = The subset of data currently visible because of slicers, page filters, visual grouping, or relationships.

Without CALCULATE:
COUNTROWS(issues)
Just counts what’s currently visible.

With CALCULATE:
CALCULATE(
    COUNTROWS(issues),
    issues[resolved_date] = BLANK()
)
It says:
Temporarily add this filter condition, then count.

Effectively, CALCULATE is a way to combine or manipulate filters. So, measures act kind of like functions; reusable sets of filters to pull into visual dashboards.

Unlike a JS function, there are no explicit parameters, but every time it runs, the current filter context is *implicitly* passed in. You don’t call it with arguments. The visual provides the “arguments” automatically.

---

Avg Resolution Days becomes a little more complex. Big picture goal: ror all resolved issues, calculate how many days each one took to resolve, then average those numbers.

CALCULATE modifies filter context and performs an aggregation.

Instead, we're using the AVERAGEX(table, expression) function.
The X functions (SUMX, AVERAGEX, etc.) are special.
They:
- Take a table
- Iterate row by row
- Evaluate an expression per row
- Aggregate the results
Note: this is reminding me a lot of using functions in Excel/Google Sheets.

In this function, first FILTER returns a new temporary table: all issues where resolved_date is not blank.
In SQL terms, which we'll learn soon, it'd be like:
  SELECT *
  FROM issues
  WHERE resolved_date IS NOT NULL;
So now we have only resolved issues.

Then, DATEDIFF, per row, calculates resolved_date − identified_date (in days).

Then the top layer, AVERAGEX, averages all the results.

------

Active vs inactive relationships

Power BI allows only one active relationship between two tables at a time if they would create ambiguity.

If both were active:

When you put Date on the axis, Power BI wouldn’t know whether it should filter issues by identified_date, or by resolved_date.

So the rule is:
One active relationship (default filter path)
Other relationships inactive (opt-in via USERELATIONSHIP)

We made identified_date active because it’s usually the primary “event date” in issue tracking. Then when we want resolution timing, we temporarily activate the other one using:
USERELATIONSHIP(...)


==================

For the SQL version, we'll just use "import" instead of DirectQuery.

Import:

Power BI:
- Pulls data into its own internal model
- Stores it inside the .pbix
- Performs calculations locally

Pros:
- Faster
- More flexible DAX
- No live dependency
- Ideal for small/medium datasets

Cons:
- Needs manual refresh to update data

DirectQuery:
- Power BI:
- Does not store the data
- Sends SQL queries to Postgres live
- Every visual interaction triggers a database query

Pros:
- Real-time data
- No data duplication
- Good for large or production databases

Cons:
- Slower
- DAX limitations
- Requires stable server
- Performance depends on DB

Ergo: DirectQuery would be what we want for a live database that's continually seeing updates, like a real-world production DB. This project uses non-real dummy data on a local server. Import works just fine.

-----

To "Save" an SQL query, you'll want to run 
CREATE VIEW name_of_view AS
followed by your query. This view acts as a sort of filtered version of the DB table(s) - the data doesn't get duplicated, and when the DB updates, so does the view. Then the view can be imported into PowerBI.

Much like DAX above, learning SQL query syntax is just something that'll take time and practice. But here's the query that recreates the monthly issues we did in DAX:

SELECT
    DATE_TRUNC('month', identified_date)::date AS month,
    COUNT(*) AS issues_identified
FROM issues
GROUP BY month;

DATE_TRUNC is a function used to truncate a date or timestamp to a specified level of precision, such as year, month, or day. In our case, we're talling it to truncate to the monthly precision.

So regardless of whether you're using SQL, or DAX or any other tool's internal filtering language, the key idea is that you're using the syntax to filter the data so it's prepped for use in PowerBI or whatever dashboard-builder you're using. 

Google filtering techniques or syntax documentation to figure out what filtering/selection logic is available, and don't forget AI is a good tool for discovering *general* syntax/logic patterns.

Of course, "filtering" might be underselling how powerful SQL is compared to something like DAX. DAX struggled with the dates because it is just filtering and is tool-specific. SQL is more like shaping/deriving the data before any tool filters even touch it. In the end, it feels like we're using one or the other to filter, but it's happening at different layers - hence the issues with dates and date ranges in the CSV/DAX version of the project.

---

SQL queries used in the SQL version of the project:

-

monthly issues identified

SELECT
    DATE_TRUNC('month', identified_date)::date AS month,
    COUNT(*) AS issues_identified
FROM issues
GROUP BY month
ORDER BY month;

-

Monthly Issues Resolved

SELECT
    DATE_TRUNC('month', resolved_date)::date AS month,
    COUNT(*) AS issues_resolved
FROM issues
WHERE resolved_date IS NOT NULL
GROUP BY month
ORDER BY month;

-

average resolution time in days

CREATE VIEW resolved_durations AS
SELECT
    issue_id,
    identified_date,
    resolved_date,
    (resolved_date - identified_date) AS resolution_days
FROM issues
WHERE resolved_date IS NOT NULL;

-

Backlog

WITH months AS (
    SELECT
        (DATE_TRUNC('month', d) + INTERVAL '1 month - 1 day')::date AS month_end
    FROM GENERATE_SERIES(
        (SELECT MIN(identified_date) FROM issues),
        (SELECT MAX(identified_date) FROM issues),
        INTERVAL '1 month'
    ) d
)

SELECT
    month_end,
    COUNT(i.issue_id) AS open_backlog
FROM months m
LEFT JOIN issues i
    ON i.identified_date <= m.month_end
   AND (i.resolved_date IS NULL OR i.resolved_date > m.month_end)
GROUP BY month_end
ORDER BY month_end;

Let's dive a little into this one since it's more complex than the other two and I like trying to understand things.

WITH months AS (...) creates a temporary table of month-end dates

DATE_TRUNC('month', d) snaps each generated date to the first day of its month, i.e. 2025-07-01

INTERVAL '1 month - 1 day' shifts that “first of month” to “last of month”, i.e. 2025-07-31

::date Casts it back to a plain DATE (no timestamp).

Then, we join every month_end to every issue that is open at that time.
LEFT JOIN issues i
ON i.identified_date <= m.month_end
AND (i.resolved_date IS NULL OR i.resolved_date > m.month_end)

Read this as:
Include issue i in month m if:
  - It was created on or before the month end
  - And it was not resolved yet by that month end
That second condition has two cases:
  - resolved_date IS NULL → still open forever
  - resolved_date > month_end → resolved later, so still open at month end

Why LEFT JOIN? So you still get months even if backlog = 0. (If you used INNER JOIN and a month had no open issues, that month might disappear.)

Then, count how many open issues each month has by the end of the month
Note: COUNT(*) vs COUNT(i.issue_id) - COUNT(i.issue_id) is safer with LEFT JOIN because it won’t count NULLs.

---

More on JOINs

A JOIN combines rows from two tables based on a condition.

In your backlog query:
Left table: months
Right table: issues

For each row in months, SQL checks which rows in issues satisfy:
i.identified_date <= month_end
AND
(i.resolved_date IS NULL OR i.resolved_date > month_end)


If a row matches, SQL combines them into one output row. If multiple rows match, you get multiple combined rows.

What Does “0..N” Mean?
For each month:
- There might be 0 matching issues (no backlog)
- There might be 1
- There might be 5
- There might be 15

There is no fixed number. So we say 0 to N matches.

INNER JOIN = Only return rows where there is a match in both tables.

LEFT JOIN = Keep all rows from the LEFT table, and if the other table doesn't match, fill its column with null. In essence, you're treating one table as the "spine".

RIGHT JOIN = Keep all rows from the RIGHT table, and if the other table doesn't match, fill its column with null. In essence, you're treating one table as the "spine". People usually just swap table order and use LEFT JOIN instead - that way, left is always authoritative/the spine, but it's good to know that queries might be written "backwards". Just know: "left" or "right" before the JOIN is telling you which table the spine is.

FULL OUTER JOIN = Keep everything from both sides. If something doesn’t match on either side, fill with NULL.