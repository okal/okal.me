---
title: "Concurrent Database Access Approaches in Python"
date: 2020-09-08T05:23:42+03:00
draft: true
---

## Background

*(NOTE: You can skip the boring part, and head right to the benchmarks and associated source code on GitHub.[^0])*

One of the services my team at JUMO works on is a reporting platform for our partners.
In addition to pre-generated reports, users have access to an interactive dashboard.
A key requirement for this was the ability for the Business Intelligence (BI) team to define the dashboards
with minimal developer involvement. We designed a domain specific language (DSL) that enabled them to declaratively
define dashboard layout, run arbitrarily complex SQL queries and map the results to pre-built,
but highly configurable visualisations.

The approach had a lot going for it. Since the DSL is constrained, it makes it possible to
validate dashboards at the authoring stage, providing meaningful feedback in case of errors.
The approach may have proven perhaps *too* successful. Dashboards became more and more complex over time.
This was not a problem in and of itself, but the approach
we took to executing the queries started to buckle under the weight of larger dashboard
definitions. Queries were executed serially. This led to a degraded user experience, since some
dashboards took well over 30 seconds to load. Our attention as a team had shifted to a new service,
so we had limited time to work on optimising the reporting platform. But it bugged me for years,
so I decided to finally do something about it. I needed to figure out the best way to run the queries in
concurrently.

## Enter the GIL

The service is built in Python, which has what I consider an undeserved reputation for poor concurrency support. This is due to
the (in)famous Global Interpreter Lock (the GIL). The GIL means that, typically speaking, Python
code cannot readily take advantage of multiple processor cores. There have been attempts at building alternative
Python runtimes that get rid of the GIL altogether, but the fact that they haven't taken off
is perhaps testament to the fact that it isn't the great handicap it's widely held to be. In practice,
for IO-bound workflows, I found it to not present much of an obstacle to optimisation. For CPU-bound
workflows, the `multiprocessing`[^1] module might be worth investigating.
Data has to be serialized and deserialized to be passed between multiple Python processes. This added
coordination overhead might make it worthwhile to explore what other languages have to offer.

## Test setup

To emulate a real-world set of queries, I generated 26 queries based on the below template. It includes
joins, nested queries, and aggregations.

```sql
SELECT CONCAT(reports.first_name, ' ', reports.last_name) AS employee_name, average_salary, CONCAT(managers.first_name, ' ', managers.last_name) AS manager_name, departments.dept_name FROM employees reports
    INNER JOIN dept_emp ON dept_emp.emp_no = reports.emp_no
    INNER JOIN dept_manager ON dept_manager.dept_no = dept_emp.dept_no
    INNER JOIN employees managers ON managers.emp_no = dept_manager.emp_no
    INNER JOIN departments ON departments.dept_no = dept_emp.dept_no
    INNER JOIN (
        SELECT FLOOR(AVG(salaries.salary)) as average_salary, salaries.emp_no FROM salaries
        GROUP BY salaries.emp_no
    ) AS average_salaries ON average_salaries.emp_no = reports.emp_no
    WHERE reports.first_name LIKE '{}%'
    ORDER BY reports.first_name
    LIMIT 5000
```

I used the test_db[^2] sample employees database, and hyperfine[^3] to run the benchmarks. I used
SQLAlchemy[^4] in all cases except the asyncio scenario, which used aiomysql[^5].

### Constraints

Database connections aren't infinite resources, so whatever concurrency approach
I investigated had to take that into consideration, and not spawn as many concurrent executions as there
were queries.

## Results

#### Single-threaded.
```
Benchmark #1: python test_single.py
  Time (mean ± σ):     17.746 s ±  0.442 s    [User: 2.370 s, System: 0.205 s]
  Range (min … max):   17.359 s … 18.781 s    10 runs
```

#### Multi-threaded

```
Benchmark #1: python test_mt.py
  Time (mean ± σ):      8.054 s ±  0.640 s    [User: 4.481 s, System: 1.782 s]
  Range (min … max):    7.148 s …  9.526 s    10 runs
```
#### Multi-threaded with BoundedSemaphore.

```
Benchmark #1: python test_mt_bounded.py
  Time (mean ± σ):      8.275 s ±  1.053 s    [User: 4.298 s, System: 1.269 s]
  Range (min … max):    6.798 s … 10.541 s    10 runs
```
#### `confurrent.futures` with ThreadPool

```
Benchmark #1: python test_cft.py
  Time (mean ± σ):      7.806 s ±  1.284 s    [User: 4.155 s, System: 1.232 s]
  Range (min … max):    6.637 s … 10.521 s    10 runs
```

#### `asyncio`

```
Benchmark #1: python test_asyncio.py
  Time (mean ± σ):      7.642 s ±  0.947 s    [User: 3.571 s, System: 0.134 s]
  Range (min … max):    6.380 s …  9.654 s    10 runs
```

## Interpretation

The asyncio based approach was the clear winner. It had the lowest mean cumulative execution time
coupled with the second lowest standard deviation, making it both efficient and fairly consistent.

It may make sense in a particular set up to go with a thread-based approach, which may work
better with existing libraries.

## References

[^0]: https://github.com/okal/python-concurrent-db-io-tests/blob/master/src/test_asyncio.py
[^1]: https://docs.python.org/3/library/multiprocessing.html
[^2]: https://github.com/datacharmer/test_db
[^3]: https://github.com/sharkdp/hyperfine
[^4]: https://www.sqlalchemy.org/
[^5]: https://aiomysql.readthedocs.io/en/latest/