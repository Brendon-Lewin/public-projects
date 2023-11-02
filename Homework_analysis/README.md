# Analysis of homework hand-in rates and evaluation of different calculation methods

This analysis calculates the different homework hand in percentages using a variety of different methods using data from satchel one (a homework tracking system which is currently underused) and the homework detention data from the student information management system (H0s).

```python
%load_ext sql
%sql duckdb://
%config SqlMagic.displaylimit = None
```



<span style="None">displaylimit: Value None will be treated as 0 (no limit)</span>



```sql
%%sql 
CREATE TABLE H0s AS SELECT * FROM read_csv_auto('~/miniconda3/Notebooks/H0_analysis/H0_all.csv');
CREATE TABLE satchel_records AS SELECT * FROM read_csv_auto('~/miniconda3/Notebooks/H0_analysis/student_submissions_report.csv');
CREATE TABLE homework_subject AS SELECT * FROM read_csv_auto('~/miniconda3/Notebooks/H0_analysis/count_homework_set_by_subject.csv');
CREATE TABLE homework_staff AS SELECT * FROM read_csv_auto('~/miniconda3/Notebooks/H0_analysis/count_homework_set_by_staff.csv');
```


<span style="None">Running query in &#x27;duckdb://&#x27;</span>





<table>
    <thead>
        <tr>
            <th>Count</th>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>



# Measures for calculating % homework not completed for last academic year

## H0s vs not submitted
This analysis compares the number of H0s set to the number of homework records marked as not submitted to assertain which is a better measure of non submission.


```sql
%%sql

WITH staff_h0_count AS (
    SELECT CONCAT(SPLIT_PART(recorded_by,' ',1) ,' ', LEFT(SPLIT_PART(recorded_by,' ', 2),1), '. ', SPLIT_PART(recorded_by,' ', 3)) AS teacher,
        COUNT(*) AS h0_count
    FROM H0s
    WHERE date < '2023-09-01'
    GROUP BY teacher
    ),

staff_n_sub AS (
    SELECT teacher, COUNT(*) AS count_n_sub
    FROM satchel_records
    WHERE status = 'Not submitted'
    AND due < '2023-09-01'
    GROUP BY teacher)

SELECT ROUND(AVG(h0_count),1) AS avg_h0_count, ROUND(AVG(count_n_sub),1) AS avg_not_sub_count
FROM staff_h0_count
INNER JOIN staff_n_sub
USING(teacher)
```


<span style="None">Running query in &#x27;duckdb://&#x27;</span>





<table>
    <thead>
        <tr>
            <th>avg_h0_count</th>
            <th>avg_not_sub_count</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>64.7</td>
            <td>89.6</td>
        </tr>
    </tbody>
</table>



### Conclusion
This shows that the average number of H0s set is lower than the number of homework records on satchel one that have been set as not submitted. For all further analysis, the number of not submitted records on satchel will be used as the indicator of homework non submission as this is still a lower bound, which is higher than the number of H0s. This means that the number of H0s will always give an indication of the number of homeworks not submitted which is too low.

## Satchel one records, excluding no status
This approach takes all of the records for every homework from satchel one that have a status set (submitted, submitted late, resubmission, absent, not submitted). It finds the percentage of those that have been marked as not submitted, assuming the rest have been submitted. 
This effectively assumes that this is a representative sample of all of the homework records that have no status set.


```sql
%%sql

WITH count_n_sub AS (
    SELECT COUNT(*) AS count
    FROM satchel_records
    WHERE status = 'Not submitted'
    AND due < '2023-09-01'),
    
count_w_status AS (
    SELECT COUNT(*) AS count
    FROM satchel_records
    WHERE status IN ('Submitted', 'Not submitted', 'Absent', 'Resubmission', 'Submitted late')
    AND due < '2023-09-01')

SELECT count_n_sub.count AS count_not_sub,
    count_w_status.count AS count_with_status,
    ROUND(count_n_sub.count * 100 / count_w_status.count,2) AS perc_not_sub
FROM count_n_sub, count_w_status;
```


<span style="None">Running query in &#x27;duckdb://&#x27;</span>





<table>
    <thead>
        <tr>
            <th>count_not_sub</th>
            <th>count_with_status</th>
            <th>perc_not_sub</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>4796</td>
            <td>41680</td>
            <td>11.51</td>
        </tr>
    </tbody>
</table>



### Conclusion:
This shows that for all homework entries on satchel one with a status set, 11.51% were marked as not submitted.
This is based on 31.67% of the total records in the dataset as the other 68.33% have no status set

## Satchel one records, including no status
This analysis takes all of the records for every homework from satchel one that have any status, or no status. It assumes that all homeworks with no status have been submitted for a best case scenario.


```sql
%%sql

WITH count_n_sub AS (
    SELECT COUNT(*) AS count
    FROM satchel_records
    WHERE status = 'Not submitted'
    AND due < '2023-09-01'),
    
count_w_status AS (
    SELECT COUNT(*) AS count
    FROM satchel_records
    WHERE due < '2023-09-01')

SELECT count_n_sub.count AS count_not_sub,
    count_w_status.count AS count_all_records,
    ROUND(count_n_sub.count * 100 / count_w_status.count,2) AS perc_not_sub
FROM count_n_sub, count_w_status;
```


<span style="None">Running query in &#x27;duckdb://&#x27;</span>





<table>
    <thead>
        <tr>
            <th>count_not_sub</th>
            <th>count_all_records</th>
            <th>perc_not_sub</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>4796</td>
            <td>131605</td>
            <td>3.64</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
WITH count_no_status AS (
    SELECT COUNT(*) AS count
    FROM satchel_records
    WHERE status = 'No status set'
    AND due < '2023-09-01'),

count_w_status AS (
    SELECT COUNT(*) AS count
    FROM satchel_records
    WHERE due < '2023-09-01')

SELECT count_no_status.count AS count_no_status,
        count_w_status.count AS count_total_records,
        ROUND(count_no_status.count * 100 / count_w_status.count,2) AS perc_no_status
FROM count_no_status, count_w_status
```


<span style="None">Running query in &#x27;duckdb://&#x27;</span>





<table>
    <thead>
        <tr>
            <th>count_no_status</th>
            <th>count_total_records</th>
            <th>perc_no_status</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>89925</td>
            <td>131605</td>
            <td>68.33</td>
        </tr>
    </tbody>
</table>



### Conclusion:
This shows that for all homework entries on satchel one, 3.64% were marked as not submitted. This is an absolute (and unrealistic) lower bound. This is due to the fact that 68.33% of homework records on satchel have no status. 41,680 records have a status, compared to 89,925 without.

## Satchel records using staff with high percentages of recorded status's as a sample
This analysis uses the satchel one records of the staff that have the highest percentages a status's recorded as a representative sample of the whole staff body.


```sql
%%sql

WITH staff_no_status AS (
    SELECT teacher, COUNT(*) AS count_no_status
    FROM satchel_records
    WHERE status = 'No status set'
    AND due < '2023-09-01'
    GROUP BY teacher),

staff_all_records AS (
    SELECT teacher, COUNT(*) AS count_all_records
    FROM satchel_records
    WHERE due < '2023-09-01'
    GROUP BY teacher),

staff_n_sub AS (
    SELECT teacher, COUNT(*) AS count_n_sub
    FROM satchel_records
    WHERE status = 'Not submitted'
    AND due < '2023-09-01'
    GROUP BY teacher),
    
staff_w_status AS (
    SELECT teacher, COUNT(*) AS count_w_status
    FROM satchel_records
    WHERE status IN ('Submitted', 'Not submitted', 'Absent', 'Resubmission', 'Submitted late')
    AND due < '2023-09-01'
    GROUP BY teacher)


SELECT COUNT(teacher), AVG(ROUND((count_n_sub * 100 / count_w_status),2)) AS perc_not_sub
FROM staff_all_records
INNER JOIN staff_no_status
USING(teacher)
INNER JOIN staff_n_sub
USING(teacher)
INNER JOIN staff_w_status
USING(teacher)
WHERE (count_no_status / count_all_records) < 0.5
AND teacher <> 'Anonymous Teacher'

```


<span style="None">Running query in &#x27;duckdb://&#x27;</span>





<table>
    <thead>
        <tr>
            <th>count(teacher)</th>
            <th>perc_not_sub</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>13</td>
            <td>9.46</td>
        </tr>
    </tbody>
</table>




```sql
%%sql

WITH staff_no_status AS (
    SELECT teacher, COUNT(*) AS count_no_status
    FROM satchel_records
    WHERE status = 'No status set'
    AND due < '2023-09-01'
    GROUP BY teacher),

staff_all_records AS (
    SELECT teacher, COUNT(*) AS count_all_records
    FROM satchel_records
    WHERE due < '2023-09-01'
    GROUP BY teacher),

staff_n_sub AS (
    SELECT teacher, COUNT(*) AS count_n_sub
    FROM satchel_records
    WHERE status = 'Not submitted'
    AND due < '2023-09-01'
    GROUP BY teacher),
    
staff_w_status AS (
    SELECT teacher, COUNT(*) AS count_w_status
    FROM satchel_records
    WHERE status IN ('Submitted', 'Not submitted', 'Absent', 'Resubmission', 'Submitted late')
    AND due < '2023-09-01'
    GROUP BY teacher)


SELECT COUNT(teacher), AVG(ROUND((count_n_sub * 100 / count_all_records),2)) AS perc_not_sub
FROM staff_all_records
INNER JOIN staff_no_status
USING(teacher)
INNER JOIN staff_n_sub
USING(teacher)
INNER JOIN staff_w_status
USING(teacher)
WHERE (count_no_status / count_all_records) < 0.5
AND teacher <> 'Anonymous Teacher'

```


<span style="None">Running query in &#x27;duckdb://&#x27;</span>





<table>
    <thead>
        <tr>
            <th>count(teacher)</th>
            <th>perc_not_sub</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>13</td>
            <td>6.632307692307692</td>
        </tr>
    </tbody>
</table>



### Conclusion

Not including records with no status set:
| % with status | % not submitted | Count of staff |
|---------------|-----------------|----------------|
| 10            | 12.85           | 42             |
| 20            | 11.76           | 31             |
| 30            | 10.38           | 27             |
| 40            | 10.45           | 21             |
| 50            | 9.46            | 13             |
| 60            | 9.09            | 10             |
| 70            | 10.32           | 6              |
| 80            | 6.02            | 1              |
| 90            | 6.02            | 1              |

Including records with no status set (lower bound):
| % with status | % not submitted | Count of staff |
|---------------|-----------------|----------------|
| 10            | 4.75            | 42             |
| 20            | 5.56            | 31             |
| 30            | 5.60            | 27             |
| 40            | 6.17            | 21             |
| 50            | 6.63            | 13             |
| 60            | 6.74            | 10             |
| 70            | 8.09            | 6              |
| 80            | 5.59            | 1              |
| 90            | 5.59            | 1              |

This shows that the quality of the average number of homeworks not being submitted is different where staff do not regularly put status's into satchel one. This makes sense as the fewer status's put on, the less likely the ones that are there are to be reliable.
However, this also shows that once you cut out the staff with less that 30% of their status's put on, the average percentage not submitted doesn't vary much, holding around roughly 10% not submitted. The final two rows are not a reliable indicator as they include only one staff member, so that would not be a representative sample of the whole staff body.
When the same analysis is done, but this time including all records even if a status is not set, then the percentages predictably drop to give lower bounds for each row.
To get a measure for the actual non submission rate, the mean average of the results for staff with % status between 30 and 70 inclusive has been used. The absolute lower bound, or best case scenario where all records with no status are assumed to have been submitted, can be calculated in the same way using the second table.

Based on this analysis, the best estimate for the not submitted rate is 9.94%, with a lower bound of 6.65 %. 
Due to it being more likely that a staff member will not set a status on satchel if the homework is completed compared to when it is not, the actual result is most likely between the two values given, at approximately 8%.
