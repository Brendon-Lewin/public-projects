# A8 and P8 subjects 2023
The aim of this analysis is the produce a table showing which subjects went into each A8 / P8 bucket and how they contributed to the school results overall.


```python
%load_ext sql
%sql duckdb://
%config SqlMagic.displaylimit = None
%config SqlMagic.named_parameters=True
```


<span style="None">displaylimit: Value None will be treated as 0 (no limit)</span>



```sql
%%sql
    
CREATE TABLE attainment_8_scores AS SELECT * FROM read_csv_auto('~/miniconda3/Notebooks/GCSE_2023_data/attainment_8_scores.csv');
CREATE TABLE ebacc_subjects AS SELECT * FROM read_csv_auto('~/miniconda3/Notebooks/GCSE_2023_data/ebacc_subjects.csv');
CREATE TABLE student_grades AS SELECT * FROM read_csv_auto('~/miniconda3/Notebooks/GCSE_2023_data/student_grades_new.csv');
CREATE TABLE subject_inclusion AS SELECT * FROM read_csv_auto('~/miniconda3/Notebooks/GCSE_2023_data/subject_inclusion.csv');
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



### Data cleaning
- Remove null grades (student did not sit that exam)
- Update U values to 0
- Duplicated the records for trilogy science to account for the fact it can take up two buckets
- Removed entries for subjects that are not included in the A8 measure
- Removed entried for music btec where the student has also sat music GCSE, as the btec is not included in the A8 measure in this case


```sql
%%sql

DELETE FROM student_grades
WHERE grade IS NULL;

-- Set U values to 0.0
UPDATE student_grades
SET grade = 0.0::numeric
WHERE grade = 'U';

-- Delete values for subjects that are not included in the A8 measure
DELETE FROM student_grades
WHERE subject IN (
    SELECT DISTINCT subject
    FROM subject_inclusion
    WHERE included = 0
    OR included = null);

-- Delete entry for music btec where music GCSE has also been sat
DELETE FROM student_grades
WHERE candidate_number IN (
    SELECT candidate_number
    FROM student_grades
    WHERE subject = 'btc1awd_music_studies'
    OR subject = 'gcse_(9-1)_music'
    GROUP BY candidate_number
    HAVING COUNT(*) > 1)
AND subject = 'btc1awd_music_studies';

-- Set all grades to numeric for consistent processing
UPDATE student_grades
SET grade = grade::numeric;

-- Add the second double award entry to student_grades, and the subject_inclusion tables
INSERT INTO subject_inclusion
    SELECT 'gcse(9-1)_da_science_double_awd2', 2;

INSERT INTO student_grades
    SELECT candidate_number, CONCAT(subject, '2'), grade
    FROM student_grades
    WHERE subject = 'gcse(9-1)_da_science_double_awd';
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



# Producing A8 table
This table will show the A8 scores for all students across all 8 buckets


```sql
%%sql --save A8_table
WITH eng_bucket AS (
    SELECT candidate_number, MAX(grade)::numeric * 2 AS grade
    FROM student_grades
    WHERE subject = 'gcse_(9-1)_english_literature'
    OR subject = 'gcse_(9-1)_english_language'
    GROUP BY candidate_number
    ),

top_eng AS (
    SELECT sg.candidate_number,
        CASE WHEN e_lang.grade >= e_lit.grade THEN 'gcse_(9-1)_english_language'
        WHEN e_lang.grade < e_lit.grade THEN 'gcse_(9-1)_english_literature' END AS subject
    FROM student_grades as sg
    INNER JOIN (
        SELECT sub.candidate_number, grade
        FROM student_grades as sub
        WHERE subject = 'gcse_(9-1)_english_language') AS e_lang
    ON sg.candidate_number = e_lang.candidate_number
    INNER JOIN (
        SELECT sub.candidate_number, grade
        FROM student_grades as sub
        WHERE subject = 'gcse_(9-1)_english_literature') AS e_lit
    ON sg.candidate_number = e_lit.candidate_number
    ),
    
maths_bucket AS (
    SELECT candidate_number, MAX(grade)::numeric * 2 AS grade
    FROM student_grades
    WHERE subject = 'gcse_(9-1)_mathematics'
    GROUP BY candidate_number
    ),

ebacc_bucket_1 AS (
    SELECT DISTINCT candidate_number, sub.subject, sub.grade
    FROM student_grades as stu_gra
    LEFT JOIN(
        SELECT candidate_number, subject, grade
        FROM student_grades as sub
        INNER JOIN ebacc_subjects
        USING(subject)
        WHERE ebacc = 1
        AND subject NOT IN ('gcse_(9-1)_english_language', 'gcse_(9-1)_mathematics', 'gcse_(9-1)_english_literature')
        AND candidate_number = stu_gra.candidate_number
        ORDER BY grade DESC, subject
        LIMIT 1) AS sub
    USING(candidate_number)
    ),

ebacc_bucket_2 AS (
    SELECT DISTINCT candidate_number, sub.subject, sub.grade
    FROM student_grades as stu_gra
    LEFT JOIN(
        SELECT candidate_number, subject, grade
        FROM student_grades as sub
        INNER JOIN ebacc_subjects
        USING(subject)
        WHERE ebacc = 1
        AND subject NOT IN ('gcse_(9-1)_english_language', 'gcse_(9-1)_mathematics', 'gcse_(9-1)_english_literature')
        AND candidate_number = stu_gra.candidate_number
        ORDER BY grade DESC, subject
        LIMIT 1 OFFSET 1) AS sub
    USING(candidate_number)
    ),

ebacc_bucket_3 AS (
    SELECT DISTINCT candidate_number, sub.subject, sub.grade
    FROM student_grades as stu_gra
    LEFT JOIN(
        SELECT candidate_number, subject, grade
        FROM student_grades as sub
        INNER JOIN ebacc_subjects
        USING(subject)
        WHERE ebacc = 1
        AND subject NOT IN ('gcse_(9-1)_english_language', 'gcse_(9-1)_mathematics', 'gcse_(9-1)_english_literature')
        AND candidate_number = stu_gra.candidate_number
        ORDER BY grade DESC, subject 
        LIMIT 1 OFFSET 2) AS sub
    USING(candidate_number)
    ),

first_5 AS (
    SELECT e.candidate_number,
        e.grade AS eng_points, 
        m.grade AS maths_points,
        eb1.subject AS ebacc_1_subject,
        eb1.grade AS ebacc_1_grade,
        eb2.subject AS ebacc_2_subject,
        eb2.grade AS ebacc_2_grade,
        eb3.subject AS ebacc_3_subject,
        eb3.grade AS ebacc_3_grade
    FROM eng_bucket as e
    INNER JOIN maths_bucket as m
    USING(candidate_number)
    INNER JOIN ebacc_bucket_1 as eb1
    USING(candidate_number)
    INNER JOIN ebacc_bucket_2 as eb2
    USING(candidate_number)
    INNER JOIN ebacc_bucket_3 as eb3
    USING(candidate_number)
    ),

open_bucket_1 AS (
    SELECT first_5.candidate_number, sub.subject, sub.grade
    FROM first_5
    LEFT JOIN (
        SELECT candidate_number, sg.subject, grade
        FROM student_grades as sg
        WHERE sg.subject NOT IN ('gcse_(9-1)_mathematics',
        (SELECT subject
        FROM top_eng
        WHERE sg.candidate_number = top_eng.candidate_number),
        (SELECT subject
        FROM ebacc_bucket_1 as sub
        WHERE sub.candidate_number = sg.candidate_number), 
        (SELECT subject
        FROM ebacc_bucket_2 as sub
        WHERE sub.candidate_number = sg.candidate_number),
        (SELECT subject
        FROM ebacc_bucket_3 as sub
        WHERE sub.candidate_number = sg.candidate_number)
        )
        AND sg.candidate_number = first_5.candidate_number
        ORDER BY grade DESC, sg.subject
        LIMIT 1 ) AS sub
    USING (candidate_number)
    ),

open_bucket_2 AS (
    SELECT first_5.candidate_number, sub.subject, sub.grade
    FROM first_5
    LEFT JOIN (
        SELECT candidate_number, sg.subject, grade
        FROM student_grades as sg
        WHERE sg.subject NOT IN ('gcse_(9-1)_mathematics',
        (SELECT subject
        FROM top_eng
        WHERE sg.candidate_number = top_eng.candidate_number),
        (SELECT subject
        FROM ebacc_bucket_1 as sub
        WHERE sub.candidate_number = sg.candidate_number), 
        (SELECT subject
        FROM ebacc_bucket_2 as sub
        WHERE sub.candidate_number = sg.candidate_number),
        (SELECT subject
        FROM ebacc_bucket_3 as sub
        WHERE sub.candidate_number = sg.candidate_number)
        )
        AND candidate_number = first_5.candidate_number
        ORDER BY grade DESC, sg.subject
        LIMIT 1 OFFSET 1 ) AS sub
    USING (candidate_number)
    ),

open_bucket_3 AS (
    SELECT first_5.candidate_number, sub.subject, sub.grade
    FROM first_5
    LEFT JOIN (
        SELECT candidate_number, sg.subject, grade
        FROM student_grades as sg
        WHERE sg.subject NOT IN ('gcse_(9-1)_mathematics',
        (SELECT subject
        FROM top_eng
        WHERE sg.candidate_number = top_eng.candidate_number),
        (SELECT subject
        FROM ebacc_bucket_1 as sub
        WHERE sub.candidate_number = sg.candidate_number), 
        (SELECT subject
        FROM ebacc_bucket_2 as sub
        WHERE sub.candidate_number = sg.candidate_number),
        (SELECT subject
        FROM ebacc_bucket_3 as sub
        WHERE sub.candidate_number = sg.candidate_number)
        )
        AND candidate_number = first_5.candidate_number
        ORDER BY grade DESC, sg.subject
        LIMIT 1 OFFSET 2 ) AS sub
    USING (candidate_number)
    )

SELECT f5.candidate_number,
    f5.eng_points AS Eng_pts,
    f5.maths_points AS Maths_pts,
    f5.ebacc_1_grade AS E_1_pts,
    f5.ebacc_2_grade AS E_2_pts,
    f5.ebacc_3_grade AS E_3_pts,
    ob1.grade AS O_1_pts,
    ob2.grade AS O_2_pts,
    ob3.grade AS O_3_pts,
    'English' AS English,
    'Maths' AS Maths,
    REPLACE(REPLACE(f5.ebacc_1_subject, 'gcse_(9-1)_', ''), 'gcse(9-1)_da_', '') AS E_1_sub,
    REPLACE(REPLACE(f5.ebacc_2_subject, 'gcse_(9-1)_', ''), 'gcse(9-1)_da_', '') AS E_2_sub,
    REPLACE(REPLACE(f5.ebacc_3_subject, 'gcse_(9-1)_', ''), 'gcse(9-1)_da_', '') AS E_3_sub,
    REPLACE(REPLACE(ob1.subject, 'gcse_(9-1)_', ''), 'gcse(9-1)_da_', '') AS O_1_sub,
    REPLACE(REPLACE(ob2.subject, 'gcse_(9-1)_', ''), 'gcse(9-1)_da_', '') AS O_2_sub,
    REPLACE(REPLACE(ob3.subject, 'gcse_(9-1)_', ''), 'gcse(9-1)_da_', '') AS O_3_sub
FROM first_5 AS f5
INNER JOIN open_bucket_1 AS ob1
USING (candidate_number)
INNER JOIN open_bucket_2 AS ob2
USING (candidate_number)
INNER JOIN open_bucket_3 AS ob3
USING (candidate_number)
ORDER BY candidate_number
```


<span style="None">Running query in &#x27;duckdb://&#x27;</span>





<table>
    <thead>
        <tr>
            <th>candidate_number</th>
            <th>Eng_pts</th>
            <th>Maths_pts</th>
            <th>E_1_pts</th>
            <th>E_2_pts</th>
            <th>E_3_pts</th>
            <th>O_1_pts</th>
            <th>O_2_pts</th>
            <th>O_3_pts</th>
            <th>English</th>
            <th>Maths</th>
            <th>E_1_sub</th>
            <th>E_2_sub</th>
            <th>E_3_sub</th>
            <th>O_1_sub</th>
            <th>O_2_sub</th>
            <th>O_3_sub</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>1232470</td>
            <td>16.000</td>
            <td>12.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>french</td>
            <td>geography</td>
            <td>english_language</td>
            <td>psychology</td>
            <td>chemistry</td>
        </tr>
        <tr>
            <td>1232471</td>
            <td>6.000</td>
            <td>2.000</td>
            <td>2.000</td>
            <td>2.000</td>
            <td>1.000</td>
            <td>5.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>art&des_:_fine_art</td>
            <td>drama_&_theat.stds</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232476</td>
            <td>10.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>3.500</td>
            <td>3.500</td>
            <td>5.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>art&des_:_fine_art</td>
            <td>camnatcer_childcare_skills</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232480</td>
            <td>12.000</td>
            <td>10.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>8.500</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>geography</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>btctechawd_health_studies</td>
            <td>camnatcer_childcare_skills</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232485</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>5.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>sport/p.e._studies</td>
            <td>bus._studs:single</td>
            <td>design&_technology</td>
        </tr>
        <tr>
            <td>1232492</td>
            <td>18.000</td>
            <td>14.000</td>
            <td>9.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>7.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>english_literature</td>
            <td>religious_studies</td>
            <td>german</td>
        </tr>
        <tr>
            <td>1232494</td>
            <td>10.000</td>
            <td>8.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>4.500</td>
            <td>5.000</td>
            <td>4.500</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>geography</td>
            <td>science_double_awd</td>
            <td>english_literature</td>
            <td>science_double_awd2</td>
            <td>bus._studs:single</td>
        </tr>
        <tr>
            <td>1232496</td>
            <td>6.000</td>
            <td>8.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>art&des_:_fine_art</td>
            <td>history</td>
            <td>btctechawd_multimedia</td>
        </tr>
        <tr>
            <td>1232505</td>
            <td>12.000</td>
            <td>10.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>chemistry</td>
            <td>history</td>
            <td>biology</td>
            <td>psychology</td>
            <td>english_language</td>
            <td>french</td>
        </tr>
        <tr>
            <td>1232507</td>
            <td>12.000</td>
            <td>12.000</td>
            <td>8.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>design&_technology</td>
            <td>geography</td>
            <td>spanish</td>
        </tr>
        <tr>
            <td>1232509</td>
            <td>18.000</td>
            <td>16.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>chemistry</td>
            <td>history</td>
            <td>art&des_:_fine_art</td>
            <td>english_language</td>
            <td>physics</td>
        </tr>
        <tr>
            <td>1232510</td>
            <td>8.000</td>
            <td>6.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
        </tr>
        <tr>
            <td>1232512</td>
            <td>12.000</td>
            <td>12.000</td>
            <td>8.000</td>
            <td>7.500</td>
            <td>7.500</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>geography</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>french</td>
            <td>music</td>
            <td>btctechawd_multimedia</td>
        </tr>
        <tr>
            <td>1232513</td>
            <td>10.000</td>
            <td>12.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>7.000</td>
            <td>5.500</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>btctechawd_travel_&_tourism</td>
            <td>btc1awd_music_studies</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232514</td>
            <td>10.000</td>
            <td>14.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>chemistry</td>
            <td>physics</td>
            <td>bus._studs:single</td>
            <td>english_literature</td>
            <td>geography</td>
        </tr>
        <tr>
            <td>1232521</td>
            <td>6.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>2.000</td>
            <td>2.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>2.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>chemistry</td>
            <td>history</td>
            <td>d&t_food_technolgy</td>
            <td>sport/p.e._studies</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232522</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>5.000</td>
            <td>4.500</td>
            <td>4.500</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>sport/p.e._studies</td>
            <td>btc1awd_music_studies</td>
            <td>d&t_food_technolgy</td>
        </tr>
        <tr>
            <td>1232524</td>
            <td>10.000</td>
            <td>10.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>geography</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>art&des_:_fine_art</td>
            <td>german</td>
            <td>english_language</td>
        </tr>
        <tr>
            <td>1232530</td>
            <td>8.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>btctechawd_travel_&_tourism</td>
            <td>english_literature</td>
            <td>religious_studies</td>
        </tr>
        <tr>
            <td>1232541</td>
            <td>12.000</td>
            <td>12.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>bus._studs:single</td>
            <td>english_language</td>
            <td>spanish</td>
        </tr>
        <tr>
            <td>1232546</td>
            <td>6.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>2.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>btctechawd_health_studies</td>
            <td>btctechawd_travel_&_tourism</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232547</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>chemistry</td>
            <td>physics</td>
            <td>btctechawd_small_business</td>
            <td>english_literature</td>
            <td>french</td>
        </tr>
        <tr>
            <td>1232548</td>
            <td>14.000</td>
            <td>12.000</td>
            <td>8.000</td>
            <td>7.500</td>
            <td>7.500</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>french</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>bus._studs:single</td>
            <td>com.stds/computing</td>
        </tr>
        <tr>
            <td>1232553</td>
            <td>16.000</td>
            <td>12.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>geography</td>
            <td>history</td>
            <td>biology</td>
            <td>german</td>
            <td>physics</td>
            <td>chemistry</td>
        </tr>
        <tr>
            <td>1232557</td>
            <td>12.000</td>
            <td>10.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>chemistry</td>
            <td>physics</td>
            <td>design&_technology</td>
            <td>english_literature</td>
            <td>geography</td>
        </tr>
        <tr>
            <td>1232559</td>
            <td>12.000</td>
            <td>12.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>chemistry</td>
            <td>geography</td>
            <td>art&des_:_fine_art</td>
            <td>history</td>
            <td>physics</td>
        </tr>
        <tr>
            <td>1232562</td>
            <td>14.000</td>
            <td>10.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>geography</td>
            <td>history</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>english_literature</td>
            <td>psychology</td>
        </tr>
        <tr>
            <td>1232565</td>
            <td>18.000</td>
            <td>14.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>8.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>drama_&_theat.stds</td>
            <td>english_literature</td>
            <td>geography</td>
        </tr>
        <tr>
            <td>1232566</td>
            <td>10.000</td>
            <td>8.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>physics</td>
            <td>chemistry</td>
            <td>english_language</td>
            <td>btctechawd_health_studies</td>
            <td>bus._studs:single</td>
        </tr>
        <tr>
            <td>1232569</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>3.500</td>
            <td>3.500</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>english_literature</td>
            <td>history</td>
            <td>religious_studies</td>
        </tr>
        <tr>
            <td>1232574</td>
            <td>4.000</td>
            <td>6.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
        </tr>
        <tr>
            <td>1232575</td>
            <td>10.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>spanish</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>english_literature</td>
            <td>geography</td>
        </tr>
        <tr>
            <td>1232576</td>
            <td>8.000</td>
            <td>10.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>chemistry</td>
            <td>german</td>
            <td>physics</td>
            <td>biology</td>
            <td>english_literature</td>
            <td>bus._studs:single</td>
        </tr>
        <tr>
            <td>1232577</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
        </tr>
        <tr>
            <td>1232579</td>
            <td>10.000</td>
            <td>8.000</td>
            <td>4.000</td>
            <td>3.500</td>
            <td>3.500</td>
            <td>6.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>spanish</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>religious_studies</td>
            <td>btctechawd_travel_&_tourism</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232581</td>
            <td>14.000</td>
            <td>14.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>religious_studies</td>
            <td>btctechawd_health_studies</td>
            <td>english_language</td>
        </tr>
        <tr>
            <td>1232583</td>
            <td>12.000</td>
            <td>10.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>8.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>geography</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>design&_technology</td>
            <td>bus._studs:single</td>
            <td>sport/p.e._studies</td>
        </tr>
        <tr>
            <td>1232584</td>
            <td>6.000</td>
            <td>8.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>bus._studs:single</td>
            <td>design&_technology</td>
            <td>religious_studies</td>
        </tr>
        <tr>
            <td>1232586</td>
            <td>12.000</td>
            <td>14.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>french</td>
            <td>physics</td>
            <td>chemistry</td>
            <td>english_literature</td>
            <td>sport/p.e._studies</td>
        </tr>
        <tr>
            <td>1232589</td>
            <td>18.000</td>
            <td>14.000</td>
            <td>9.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>chemistry</td>
            <td>french</td>
            <td>art&des_:_fine_art</td>
            <td>english_literature</td>
            <td>physics</td>
        </tr>
        <tr>
            <td>1232592</td>
            <td>10.000</td>
            <td>10.000</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>English</td>
            <td>Maths</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
        </tr>
        <tr>
            <td>1232600</td>
            <td>14.000</td>
            <td>12.000</td>
            <td>8.500</td>
            <td>8.500</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>french</td>
            <td>btc1awd_music_studies</td>
            <td>english_language</td>
            <td>geography</td>
        </tr>
        <tr>
            <td>1232602</td>
            <td>16.000</td>
            <td>10.000</td>
            <td>6.000</td>
            <td>5.500</td>
            <td>5.500</td>
            <td>8.500</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>btc1awd_music_studies</td>
            <td>english_literature</td>
            <td>art&des_:_fine_art</td>
        </tr>
        <tr>
            <td>1232604</td>
            <td>16.000</td>
            <td>18.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>german</td>
            <td>drama_&_theat.stds</td>
            <td>history</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232608</td>
            <td>16.000</td>
            <td>12.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>religious_studies</td>
            <td>spanish</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232613</td>
            <td>8.000</td>
            <td>2.000</td>
            <td>2.000</td>
            <td>2.000</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
        </tr>
        <tr>
            <td>1232616</td>
            <td>14.000</td>
            <td>12.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>bus._studs:single</td>
            <td>geography</td>
            <td>english_language</td>
        </tr>
        <tr>
            <td>1232617</td>
            <td>18.000</td>
            <td>12.000</td>
            <td>6.500</td>
            <td>6.500</td>
            <td>5.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>art&des_:_fine_art</td>
            <td>religious_studies</td>
            <td>psychology</td>
        </tr>
        <tr>
            <td>1232619</td>
            <td>16.000</td>
            <td>12.000</td>
            <td>8.000</td>
            <td>7.500</td>
            <td>7.500</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>geography</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>bus._studs:single</td>
            <td>design&_technology</td>
            <td>sport/p.e._studies</td>
        </tr>
        <tr>
            <td>1232620</td>
            <td>12.000</td>
            <td>12.000</td>
            <td>7.000</td>
            <td>6.500</td>
            <td>6.500</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>geography</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>bus._studs:single</td>
            <td>french</td>
            <td>sport/p.e._studies</td>
        </tr>
        <tr>
            <td>1232621</td>
            <td>14.000</td>
            <td>12.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>design&_technology</td>
            <td>english_literature</td>
            <td>bus._studs:single</td>
        </tr>
        <tr>
            <td>1232622</td>
            <td>10.000</td>
            <td>6.000</td>
            <td>4.000</td>
            <td>3.500</td>
            <td>3.500</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>sport/p.e._studies</td>
            <td>english_language</td>
            <td>btctechawd_small_business</td>
        </tr>
        <tr>
            <td>1232623</td>
            <td>12.000</td>
            <td>16.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>history</td>
            <td>religious_studies</td>
            <td>spanish</td>
        </tr>
        <tr>
            <td>1232624</td>
            <td>10.000</td>
            <td>8.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>6.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>d&t_food_technolgy</td>
            <td>btctechawd_health_studies</td>
            <td>camnatcer_childcare_skills</td>
        </tr>
        <tr>
            <td>1232635</td>
            <td>10.000</td>
            <td>10.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>6.000</td>
            <td>5.500</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>d&t_food_technolgy</td>
            <td>btc1awd_music_studies</td>
            <td>btctechawd_small_business</td>
        </tr>
        <tr>
            <td>1232636</td>
            <td>12.000</td>
            <td>12.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>sport/p.e._studies</td>
            <td>design&_technology</td>
            <td>bus._studs:single</td>
        </tr>
        <tr>
            <td>1232637</td>
            <td>8.000</td>
            <td>10.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>1.000</td>
            <td>5.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>art&des_:_fine_art</td>
            <td>bus._studs:single</td>
            <td>design&_technology</td>
        </tr>
        <tr>
            <td>1232640</td>
            <td>10.000</td>
            <td>8.000</td>
            <td>3.500</td>
            <td>3.500</td>
            <td>3.000</td>
            <td>5.500</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>camnatcer_childcare_skills</td>
            <td>art&des_:_fine_art</td>
            <td>btctechawd_health_studies</td>
        </tr>
        <tr>
            <td>1232642</td>
            <td>12.000</td>
            <td>10.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>6.000</td>
            <td>5.500</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>english_literature</td>
            <td>btctechawd_health_studies</td>
            <td>religious_studies</td>
        </tr>
        <tr>
            <td>1232643</td>
            <td>8.000</td>
            <td>10.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>french</td>
            <td>camnatcer_childcare_skills</td>
            <td>d&t_food_technolgy</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232644</td>
            <td>16.000</td>
            <td>14.000</td>
            <td>9.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>psychology</td>
            <td>english_language</td>
            <td>geography</td>
        </tr>
        <tr>
            <td>1232646</td>
            <td>14.000</td>
            <td>10.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>geography</td>
            <td>chemistry</td>
            <td>art&des_:_fine_art</td>
            <td>design&_technology</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232648</td>
            <td>16.000</td>
            <td>16.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>8.000</td>
            <td>8.500</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>btc1awd_music_studies</td>
            <td>history</td>
            <td>spanish</td>
        </tr>
        <tr>
            <td>1232649</td>
            <td>16.000</td>
            <td>12.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>chemistry</td>
            <td>biology</td>
            <td>history</td>
            <td>psychology</td>
            <td>english_literature</td>
            <td>physics</td>
        </tr>
        <tr>
            <td>1232653</td>
            <td>16.000</td>
            <td>10.000</td>
            <td>8.000</td>
            <td>6.500</td>
            <td>6.500</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>bus._studs:single</td>
            <td>design&_technology</td>
            <td>art&des_:_fine_art</td>
        </tr>
        <tr>
            <td>1232654</td>
            <td>18.000</td>
            <td>16.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>english_literature</td>
            <td>psychology</td>
            <td>religious_studies</td>
        </tr>
        <tr>
            <td>1232655</td>
            <td>16.000</td>
            <td>10.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>french</td>
            <td>geography</td>
            <td>biology</td>
            <td>art&des_:_fine_art</td>
            <td>english_literature</td>
            <td>chemistry</td>
        </tr>
        <tr>
            <td>1232657</td>
            <td>12.000</td>
            <td>14.000</td>
            <td>6.500</td>
            <td>6.500</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>design&_technology</td>
            <td>art&des_:_fine_art</td>
            <td>bus._studs:single</td>
        </tr>
        <tr>
            <td>1232663</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>4.500</td>
            <td>4.500</td>
            <td>3.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>art&des_:_fine_art</td>
            <td>drama_&_theat.stds</td>
            <td>design&_technology</td>
        </tr>
        <tr>
            <td>1232667</td>
            <td>10.000</td>
            <td>12.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>french</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>art&des_:_fine_art</td>
            <td>com.stds/computing</td>
        </tr>
        <tr>
            <td>1232668</td>
            <td>18.000</td>
            <td>10.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>6.500</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>6.500</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>german</td>
            <td>science_double_awd</td>
            <td>drama_&_theat.stds</td>
            <td>english_literature</td>
            <td>science_double_awd2</td>
        </tr>
        <tr>
            <td>1232671</td>
            <td>16.000</td>
            <td>16.000</td>
            <td>9.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>polish</td>
            <td>biology</td>
            <td>chemistry</td>
            <td>history</td>
            <td>physics</td>
            <td>french</td>
        </tr>
        <tr>
            <td>1232673</td>
            <td>18.000</td>
            <td>10.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>5.500</td>
            <td>6.000</td>
            <td>5.500</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>geography</td>
            <td>history</td>
            <td>science_double_awd</td>
            <td>english_literature</td>
            <td>science_double_awd2</td>
            <td>sport/p.e._studies</td>
        </tr>
        <tr>
            <td>1232677</td>
            <td>12.000</td>
            <td>12.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>physics</td>
            <td>biology</td>
            <td>chemistry</td>
            <td>bus._studs:single</td>
            <td>com.stds/computing</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232678</td>
            <td>16.000</td>
            <td>12.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>9.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>chemistry</td>
            <td>history</td>
            <td>art&des_:_fine_art</td>
            <td>english_literature</td>
            <td>physics</td>
        </tr>
        <tr>
            <td>1232682</td>
            <td>16.000</td>
            <td>14.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>physics</td>
            <td>chemistry</td>
            <td>english_literature</td>
            <td>history</td>
            <td>psychology</td>
        </tr>
        <tr>
            <td>1232683</td>
            <td>14.000</td>
            <td>10.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>spanish</td>
            <td>drama_&_theat.stds</td>
            <td>english_language</td>
            <td>psychology</td>
        </tr>
        <tr>
            <td>1232684</td>
            <td>12.000</td>
            <td>12.000</td>
            <td>7.000</td>
            <td>6.500</td>
            <td>6.500</td>
            <td>6.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>french</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>sport/p.e._studies</td>
            <td>bus._studs:single</td>
            <td>english_language</td>
        </tr>
        <tr>
            <td>1232685</td>
            <td>8.000</td>
            <td>6.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>religious_studies</td>
            <td>btctechawd_travel_&_tourism</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232686</td>
            <td>12.000</td>
            <td>10.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>english_literature</td>
            <td>religious_studies</td>
            <td>sport/p.e._studies</td>
        </tr>
        <tr>
            <td>1232690</td>
            <td>12.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>french</td>
            <td>history</td>
            <td>biology</td>
            <td>english_literature</td>
            <td>bus._studs:single</td>
            <td>chemistry</td>
        </tr>
        <tr>
            <td>1232694</td>
            <td>14.000</td>
            <td>14.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>polish</td>
            <td>chemistry</td>
            <td>bus._studs:single</td>
            <td>geography</td>
            <td>physics</td>
        </tr>
        <tr>
            <td>1232695</td>
            <td>10.000</td>
            <td>8.000</td>
            <td>5.000</td>
            <td>4.500</td>
            <td>4.500</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>geography</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>design&_technology</td>
            <td>art&des_:_fine_art</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232696</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>4.500</td>
            <td>4.500</td>
            <td>3.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>com.stds/computing</td>
            <td>btctechawd_small_business</td>
            <td>history</td>
            <td>sport/p.e._studies</td>
        </tr>
        <tr>
            <td>1232699</td>
            <td>14.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>5.500</td>
            <td>5.500</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>religious_studies</td>
            <td>sport/p.e._studies</td>
            <td>english_language</td>
        </tr>
        <tr>
            <td>1232701</td>
            <td>10.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>6.000</td>
            <td>5.500</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>religious_studies</td>
            <td>btctechawd_health_studies</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232702</td>
            <td>18.000</td>
            <td>12.000</td>
            <td>9.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>spanish</td>
            <td>history</td>
            <td>english_literature</td>
            <td>psychology</td>
            <td>chemistry</td>
        </tr>
        <tr>
            <td>1232703</td>
            <td>12.000</td>
            <td>12.000</td>
            <td>6.500</td>
            <td>6.500</td>
            <td>6.000</td>
            <td>8.500</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>btc1awd_music_studies</td>
            <td>bus._studs:single</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232704</td>
            <td>16.000</td>
            <td>12.000</td>
            <td>9.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>chemistry</td>
            <td>biology</td>
            <td>geography</td>
            <td>physics</td>
            <td>french</td>
            <td>com.stds/computing</td>
        </tr>
        <tr>
            <td>1232705</td>
            <td>14.000</td>
            <td>14.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>chemistry</td>
            <td>history</td>
            <td>english_literature</td>
            <td>music</td>
            <td>bus._studs:single</td>
        </tr>
        <tr>
            <td>1232706</td>
            <td>10.000</td>
            <td>10.000</td>
            <td>9.000</td>
            <td>5.500</td>
            <td>5.500</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>polish</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>art&des_:_fine_art</td>
            <td>design&_technology</td>
            <td>religious_studies</td>
        </tr>
        <tr>
            <td>1232707</td>
            <td>8.000</td>
            <td>6.000</td>
            <td>3.500</td>
            <td>3.500</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>btctechawd_multimedia</td>
            <td>btctechawd_small_business</td>
            <td>english_language</td>
        </tr>
        <tr>
            <td>1232712</td>
            <td>10.000</td>
            <td>8.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>8.500</td>
            <td>7.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>btctechawd_health_studies</td>
            <td>camnatcer_childcare_skills</td>
            <td>design&_technology</td>
        </tr>
        <tr>
            <td>1232713</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>polish</td>
            <td>geography</td>
            <td>physics</td>
            <td>english_literature</td>
            <td>biology</td>
            <td>bus._studs:single</td>
        </tr>
        <tr>
            <td>1232714</td>
            <td>12.000</td>
            <td>12.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>english_literature</td>
            <td>design&_technology</td>
            <td>art&des_:_fine_art</td>
        </tr>
        <tr>
            <td>1232716</td>
            <td>12.000</td>
            <td>14.000</td>
            <td>7.500</td>
            <td>7.500</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>btctechawd_health_studies</td>
            <td>english_literature</td>
            <td>psychology</td>
        </tr>
        <tr>
            <td>1232719</td>
            <td>14.000</td>
            <td>12.000</td>
            <td>8.000</td>
            <td>7.500</td>
            <td>7.500</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>geography</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>music</td>
            <td>art&des_:_fine_art</td>
            <td>design&_technology</td>
        </tr>
        <tr>
            <td>1232720</td>
            <td>10.000</td>
            <td>16.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>chemistry</td>
            <td>biology</td>
            <td>physics</td>
            <td>bus._studs:single</td>
            <td>geography</td>
            <td>design&_technology</td>
        </tr>
        <tr>
            <td>1232723</td>
            <td>18.000</td>
            <td>14.000</td>
            <td>9.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>french</td>
            <td>biology</td>
            <td>geography</td>
            <td>bus._studs:single</td>
            <td>english_literature</td>
            <td>chemistry</td>
        </tr>
        <tr>
            <td>1232724</td>
            <td>16.000</td>
            <td>14.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>chemistry</td>
            <td>physics</td>
            <td>geography</td>
            <td>english_literature</td>
            <td>design&_technology</td>
        </tr>
        <tr>
            <td>1232725</td>
            <td>10.000</td>
            <td>10.000</td>
            <td>4.000</td>
            <td>3.500</td>
            <td>3.500</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>geography</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>english_literature</td>
            <td>bus._studs:single</td>
            <td>sport/p.e._studies</td>
        </tr>
        <tr>
            <td>1232732</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>2.500</td>
            <td>2.500</td>
            <td>1.000</td>
            <td>5.000</td>
            <td>3.000</td>
            <td>2.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>d&t_food_technolgy</td>
            <td>art&des_:_fine_art</td>
            <td>sport/p.e._studies</td>
        </tr>
        <tr>
            <td>1232734</td>
            <td>14.000</td>
            <td>12.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>geography</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>bus._studs:single</td>
            <td>english_language</td>
            <td>sport/p.e._studies</td>
        </tr>
        <tr>
            <td>1232736</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>4.500</td>
            <td>4.500</td>
            <td>4.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>drama_&_theat.stds</td>
            <td>bus._studs:single</td>
            <td>design&_technology</td>
        </tr>
        <tr>
            <td>1232738</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>4.000</td>
            <td>3.500</td>
            <td>3.500</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>btctechawd_multimedia</td>
            <td>btctechawd_small_business</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232742</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>french</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>camnatcer_childcare_skills</td>
            <td>bus._studs:single</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232743</td>
            <td>10.000</td>
            <td>6.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>spanish</td>
            <td>art&des_:_fine_art</td>
            <td>btctechawd_multimedia</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232746</td>
            <td>12.000</td>
            <td>6.000</td>
            <td>3.000</td>
            <td>2.000</td>
            <td>2.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>5.500</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>art&des_:_fine_art</td>
            <td>drama_&_theat.stds</td>
            <td>btctechawd_health_studies</td>
        </tr>
        <tr>
            <td>1232749</td>
            <td>10.000</td>
            <td>10.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>geography</td>
            <td>history</td>
            <td>science_double_awd</td>
            <td>design&_technology</td>
            <td>science_double_awd2</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232751</td>
            <td>16.000</td>
            <td>12.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>9.000</td>
            <td>8.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>geography</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>bus._studs:single</td>
            <td>religious_studies</td>
            <td>english_language</td>
        </tr>
        <tr>
            <td>1232756</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>3.000</td>
            <td>2.500</td>
            <td>2.500</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>art&des_:_fine_art</td>
            <td>drama_&_theat.stds</td>
            <td>btctechawd_health_studies</td>
        </tr>
        <tr>
            <td>1232763</td>
            <td>4.000</td>
            <td>2.000</td>
            <td>2.500</td>
            <td>2.500</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
        </tr>
        <tr>
            <td>1232764</td>
            <td>18.000</td>
            <td>16.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>8.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>chemistry</td>
            <td>history</td>
            <td>physics</td>
            <td>psychology</td>
            <td>german</td>
        </tr>
        <tr>
            <td>1232767</td>
            <td>14.000</td>
            <td>10.000</td>
            <td>9.000</td>
            <td>8.000</td>
            <td>6.500</td>
            <td>7.000</td>
            <td>6.500</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>polish</td>
            <td>geography</td>
            <td>science_double_awd</td>
            <td>art&des_:_fine_art</td>
            <td>science_double_awd2</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232770</td>
            <td>14.000</td>
            <td>8.000</td>
            <td>5.000</td>
            <td>4.500</td>
            <td>4.500</td>
            <td>6.000</td>
            <td>5.500</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>french</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>english_language</td>
            <td>btc1awd_music_studies</td>
            <td>psychology</td>
        </tr>
        <tr>
            <td>1232773</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>4.500</td>
            <td>4.500</td>
            <td>4.000</td>
            <td>5.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>religious_studies</td>
            <td>english_language</td>
            <td>geography</td>
        </tr>
        <tr>
            <td>1232775</td>
            <td>8.000</td>
            <td>10.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>5.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>geography</td>
            <td>history</td>
            <td>science_double_awd</td>
            <td>religious_studies</td>
            <td>sport/p.e._studies</td>
            <td>science_double_awd2</td>
        </tr>
        <tr>
            <td>1232779</td>
            <td>10.000</td>
            <td>10.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>chemistry</td>
            <td>physics</td>
            <td>sport/p.e._studies</td>
            <td>btctechawd_small_business</td>
            <td>com.stds/computing</td>
        </tr>
        <tr>
            <td>1232783</td>
            <td>16.000</td>
            <td>12.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>geography</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>bus._studs:single</td>
            <td>english_literature</td>
            <td>sport/p.e._studies</td>
        </tr>
        <tr>
            <td>1232784</td>
            <td>12.000</td>
            <td>12.000</td>
            <td>6.500</td>
            <td>6.500</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>english_literature</td>
            <td>sport/p.e._studies</td>
            <td>psychology</td>
        </tr>
        <tr>
            <td>1232785</td>
            <td>10.000</td>
            <td>10.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>biology</td>
            <td>chemistry</td>
            <td>bus._studs:single</td>
            <td>english_language</td>
            <td>geography</td>
        </tr>
        <tr>
            <td>1232786</td>
            <td>14.000</td>
            <td>14.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>sport/p.e._studies</td>
            <td>geography</td>
            <td>design&_technology</td>
        </tr>
        <tr>
            <td>1232789</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>5.500</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>btc1awd_music_studies</td>
            <td>bus._studs:single</td>
            <td>english_language</td>
        </tr>
        <tr>
            <td>1232794</td>
            <td>10.000</td>
            <td>10.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>7.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>btc1awd_music_studies</td>
            <td>psychology</td>
            <td>english_language</td>
        </tr>
        <tr>
            <td>1232798</td>
            <td>12.000</td>
            <td>8.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>design&_technology</td>
            <td>english_language</td>
            <td>german</td>
        </tr>
        <tr>
            <td>1232800</td>
            <td>12.000</td>
            <td>8.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>spanish</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>english_literature</td>
            <td>religious_studies</td>
            <td>history</td>
        </tr>
        <tr>
            <td>1232801</td>
            <td>6.000</td>
            <td>8.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>2.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>chemistry</td>
            <td>physics</td>
            <td>btctechawd_small_business</td>
            <td>sport/p.e._studies</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232802</td>
            <td>4.000</td>
            <td>8.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
        </tr>
        <tr>
            <td>1232807</td>
            <td>12.000</td>
            <td>8.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>chemistry</td>
            <td>geography</td>
            <td>bus._studs:single</td>
            <td>design&_technology</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232808</td>
            <td>2.000</td>
            <td>2.000</td>
            <td>1.000</td>
            <td>1.000</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
        </tr>
        <tr>
            <td>1232809</td>
            <td>14.000</td>
            <td>10.000</td>
            <td>6.000</td>
            <td>5.500</td>
            <td>5.500</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>btc1awd_music_studies</td>
            <td>english_literature</td>
            <td>psychology</td>
        </tr>
        <tr>
            <td>1232811</td>
            <td>8.000</td>
            <td>6.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>physics</td>
            <td>chemistry</td>
            <td>btctechawd_small_business</td>
            <td>english_language</td>
            <td>history</td>
        </tr>
        <tr>
            <td>1232813</td>
            <td>16.000</td>
            <td>16.000</td>
            <td>9.000</td>
            <td>8.500</td>
            <td>8.500</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>french</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>art&des_:_fine_art</td>
            <td>religious_studies</td>
            <td>geography</td>
        </tr>
        <tr>
            <td>1232819</td>
            <td>8.000</td>
            <td>10.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>3.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>spanish</td>
            <td>bus._studs:single</td>
            <td>english_literature</td>
            <td>sport/p.e._studies</td>
        </tr>
        <tr>
            <td>1232826</td>
            <td>10.000</td>
            <td>10.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>5.500</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>btc1awd_music_studies</td>
            <td>btctechawd_health_studies</td>
            <td>design&_technology</td>
        </tr>
        <tr>
            <td>1232830</td>
            <td>12.000</td>
            <td>10.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>design&_technology</td>
            <td>history</td>
            <td>btctechawd_small_business</td>
        </tr>
        <tr>
            <td>1232832</td>
            <td>12.000</td>
            <td>14.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>8.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>geography</td>
            <td>physics</td>
            <td>design&_technology</td>
            <td>chemistry</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232834</td>
            <td>2.000</td>
            <td>4.000</td>
            <td>2.500</td>
            <td>2.500</td>
            <td>1.000</td>
            <td>2.000</td>
            <td>1.750</td>
            <td>1.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>sport/p.e._studies</td>
            <td>btc1awd_music_studies</td>
            <td>bus._studs:single</td>
        </tr>
        <tr>
            <td>1232840</td>
            <td>10.000</td>
            <td>10.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>4.500</td>
            <td>5.000</td>
            <td>4.500</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>french</td>
            <td>science_double_awd</td>
            <td>english_literature</td>
            <td>science_double_awd2</td>
            <td>btctechawd_travel_&_tourism</td>
        </tr>
        <tr>
            <td>1232842</td>
            <td>8.000</td>
            <td>12.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>5.500</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>btc1awd_music_studies</td>
            <td>btctechawd_multimedia</td>
            <td>english_language</td>
        </tr>
        <tr>
            <td>1232843</td>
            <td>14.000</td>
            <td>6.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>religious_studies</td>
            <td>art&des_:_fine_art</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232844</td>
            <td>16.000</td>
            <td>16.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>8.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>art&des_:_fine_art</td>
            <td>religious_studies</td>
            <td>french</td>
        </tr>
        <tr>
            <td>1232845</td>
            <td>10.000</td>
            <td>8.000</td>
            <td>5.500</td>
            <td>5.500</td>
            <td>4.000</td>
            <td>7.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>sport/p.e._studies</td>
            <td>english_literature</td>
            <td>bus._studs:single</td>
        </tr>
        <tr>
            <td>1232848</td>
            <td>16.000</td>
            <td>12.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>biology</td>
            <td>chemistry</td>
            <td>geography</td>
            <td>design&_technology</td>
            <td>english_literature</td>
            <td>physics</td>
        </tr>
        <tr>
            <td>1232849</td>
            <td>8.000</td>
            <td>4.000</td>
            <td>2.500</td>
            <td>2.500</td>
            <td>2.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>art&des_:_fine_art</td>
            <td>religious_studies</td>
            <td>btctechawd_health_studies</td>
        </tr>
        <tr>
            <td>1232851</td>
            <td>6.000</td>
            <td>2.000</td>
            <td>2.000</td>
            <td>2.000</td>
            <td>2.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>2.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>btctechawd_multimedia</td>
            <td>camnatcer_childcare_skills</td>
            <td>btctechawd_small_business</td>
        </tr>
        <tr>
            <td>1232852</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>2.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>2.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>btctechawd_multimedia</td>
            <td>btctechawd_small_business</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232853</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>2.500</td>
            <td>2.500</td>
            <td>1.000</td>
            <td>4.000</td>
            <td>2.000</td>
            <td>2.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>com.stds/computing</td>
            <td>btctechawd_small_business</td>
            <td>bus._studs:single</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232856</td>
            <td>14.000</td>
            <td>14.000</td>
            <td>8.000</td>
            <td>8.000</td>
            <td>7.000</td>
            <td>8.500</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>german</td>
            <td>btc1awd_music_studies</td>
            <td>btctechawd_travel_&_tourism</td>
            <td>english_literature</td>
        </tr>
        <tr>
            <td>1232857</td>
            <td>12.000</td>
            <td>8.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>bus._studs:single</td>
            <td>english_literature</td>
            <td>design&_technology</td>
        </tr>
        <tr>
            <td>1232860</td>
            <td>4.000</td>
            <td>6.000</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>English</td>
            <td>Maths</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
        </tr>
        <tr>
            <td>1232861</td>
            <td>8.000</td>
            <td>6.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>3.000</td>
            <td>7.000</td>
            <td>5.500</td>
            <td>5.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>camnatcer_childcare_skills</td>
            <td>btctechawd_health_studies</td>
            <td>d&t_food_technolgy</td>
        </tr>
        <tr>
            <td>1232862</td>
            <td>16.000</td>
            <td>12.000</td>
            <td>9.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>history</td>
            <td>biology</td>
            <td>chemistry</td>
            <td>art&des_:_fine_art</td>
            <td>english_language</td>
            <td>physics</td>
        </tr>
        <tr>
            <td>1232867</td>
            <td>14.000</td>
            <td>10.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>7.000</td>
            <td>8.500</td>
            <td>7.000</td>
            <td>6.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>camnatcer_childcare_skills</td>
            <td>btctechawd_health_studies</td>
            <td>english_language</td>
        </tr>
        <tr>
            <td>1232870</td>
            <td>6.000</td>
            <td>8.000</td>
            <td>4.000</td>
            <td>4.000</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
            <td>None</td>
        </tr>
        <tr>
            <td>1232873</td>
            <td>12.000</td>
            <td>8.000</td>
            <td>4.500</td>
            <td>4.500</td>
            <td>4.000</td>
            <td>6.000</td>
            <td>5.000</td>
            <td>4.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>geography</td>
            <td>sport/p.e._studies</td>
            <td>english_literature</td>
            <td>bus._studs:single</td>
        </tr>
        <tr>
            <td>1232874</td>
            <td>16.000</td>
            <td>16.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>9.000</td>
            <td>7.000</td>
            <td>English</td>
            <td>Maths</td>
            <td>science_double_awd</td>
            <td>science_double_awd2</td>
            <td>history</td>
            <td>design&_technology</td>
            <td>psychology</td>
            <td>english_language</td>
        </tr>
    </tbody>
</table>



# Dropped subjects
This table shows subjects that were dropped from all 8 buckets


```sql
%%sql

WITH eng_bucket AS (
    SELECT candidate_number, MAX(grade)::numeric * 2 AS grade
    FROM student_grades
    WHERE subject = 'gcse_(9-1)_english_literature'
    OR subject = 'gcse_(9-1)_english_language'
    GROUP BY candidate_number
    ),

top_eng AS (
    SELECT sg.candidate_number,
        CASE WHEN e_lang.grade >= e_lit.grade THEN 'gcse_(9-1)_english_language'
        WHEN e_lang.grade < e_lit.grade THEN 'gcse_(9-1)_english_literature' END AS subject
    FROM student_grades as sg
    INNER JOIN (
        SELECT sub.candidate_number, grade
        FROM student_grades as sub
        WHERE subject = 'gcse_(9-1)_english_language') AS e_lang
    ON sg.candidate_number = e_lang.candidate_number
    INNER JOIN (
        SELECT sub.candidate_number, grade
        FROM student_grades as sub
        WHERE subject = 'gcse_(9-1)_english_literature') AS e_lit
    ON sg.candidate_number = e_lit.candidate_number
    ),
    
maths_bucket AS (
    SELECT candidate_number, MAX(grade)::numeric * 2 AS grade
    FROM student_grades
    WHERE subject = 'gcse_(9-1)_mathematics'
    GROUP BY candidate_number
    ),

ebacc_bucket_1 AS (
    SELECT DISTINCT candidate_number, sub.subject, sub.grade
    FROM student_grades as stu_gra
    LEFT JOIN(
        SELECT candidate_number, subject, grade
        FROM student_grades as sub
        INNER JOIN ebacc_subjects
        USING(subject)
        WHERE ebacc = 1
        AND subject NOT IN ('gcse_(9-1)_english_language', 'gcse_(9-1)_mathematics', 'gcse_(9-1)_english_literature')
        AND candidate_number = stu_gra.candidate_number
        ORDER BY grade DESC, subject
        LIMIT 1) AS sub
    USING(candidate_number)
    ),

ebacc_bucket_2 AS (
    SELECT DISTINCT candidate_number, sub.subject, sub.grade
    FROM student_grades as stu_gra
    LEFT JOIN(
        SELECT candidate_number, subject, grade
        FROM student_grades as sub
        INNER JOIN ebacc_subjects
        USING(subject)
        WHERE ebacc = 1
        AND subject NOT IN ('gcse_(9-1)_english_language', 'gcse_(9-1)_mathematics', 'gcse_(9-1)_english_literature')
        AND candidate_number = stu_gra.candidate_number
        ORDER BY grade DESC, subject
        LIMIT 1 OFFSET 1) AS sub
    USING(candidate_number)
    ),

ebacc_bucket_3 AS (
    SELECT DISTINCT candidate_number, sub.subject, sub.grade
    FROM student_grades as stu_gra
    LEFT JOIN(
        SELECT candidate_number, subject, grade
        FROM student_grades as sub
        INNER JOIN ebacc_subjects
        USING(subject)
        WHERE ebacc = 1
        AND subject NOT IN ('gcse_(9-1)_english_language', 'gcse_(9-1)_mathematics', 'gcse_(9-1)_english_literature')
        AND candidate_number = stu_gra.candidate_number
        ORDER BY grade DESC, subject 
        LIMIT 1 OFFSET 2) AS sub
    USING(candidate_number)
    ),

first_5 AS (
    SELECT e.candidate_number,
        e.grade AS eng_points, 
        m.grade AS maths_points,
        eb1.subject AS ebacc_1_subject,
        eb1.grade AS ebacc_1_grade,
        eb2.subject AS ebacc_2_subject,
        eb2.grade AS ebacc_2_grade,
        eb3.subject AS ebacc_3_subject,
        eb3.grade AS ebacc_3_grade
    FROM eng_bucket as e
    INNER JOIN maths_bucket as m
    USING(candidate_number)
    INNER JOIN ebacc_bucket_1 as eb1
    USING(candidate_number)
    INNER JOIN ebacc_bucket_2 as eb2
    USING(candidate_number)
    INNER JOIN ebacc_bucket_3 as eb3
    USING(candidate_number)
    ),

open_bucket_1 AS (
    SELECT first_5.candidate_number, sub.subject, sub.grade
    FROM first_5
    LEFT JOIN (
        SELECT candidate_number, sg.subject, grade
        FROM student_grades as sg
        WHERE sg.subject NOT IN ('gcse_(9-1)_mathematics',
        (SELECT subject
        FROM top_eng
        WHERE sg.candidate_number = top_eng.candidate_number),
        (SELECT subject
        FROM ebacc_bucket_1 as sub
        WHERE sub.candidate_number = sg.candidate_number), 
        (SELECT subject
        FROM ebacc_bucket_2 as sub
        WHERE sub.candidate_number = sg.candidate_number),
        (SELECT subject
        FROM ebacc_bucket_3 as sub
        WHERE sub.candidate_number = sg.candidate_number)
        )
        AND sg.candidate_number = first_5.candidate_number
        ORDER BY grade DESC, sg.subject
        LIMIT 1 ) AS sub
    USING (candidate_number)
    ),

open_bucket_2 AS (
    SELECT first_5.candidate_number, sub.subject, sub.grade
    FROM first_5
    LEFT JOIN (
        SELECT candidate_number, sg.subject, grade
        FROM student_grades as sg
        WHERE sg.subject NOT IN ('gcse_(9-1)_mathematics',
        (SELECT subject
        FROM top_eng
        WHERE sg.candidate_number = top_eng.candidate_number),
        (SELECT subject
        FROM ebacc_bucket_1 as sub
        WHERE sub.candidate_number = sg.candidate_number), 
        (SELECT subject
        FROM ebacc_bucket_2 as sub
        WHERE sub.candidate_number = sg.candidate_number),
        (SELECT subject
        FROM ebacc_bucket_3 as sub
        WHERE sub.candidate_number = sg.candidate_number)
        )
        AND candidate_number = first_5.candidate_number
        ORDER BY grade DESC, sg.subject
        LIMIT 1 OFFSET 1 ) AS sub
    USING (candidate_number)
    ),

open_bucket_3 AS (
    SELECT first_5.candidate_number, sub.subject, sub.grade
    FROM first_5
    LEFT JOIN (
        SELECT candidate_number, sg.subject, grade
        FROM student_grades as sg
        WHERE sg.subject NOT IN ('gcse_(9-1)_mathematics',
        (SELECT subject
        FROM top_eng
        WHERE sg.candidate_number = top_eng.candidate_number),
        (SELECT subject
        FROM ebacc_bucket_1 as sub
        WHERE sub.candidate_number = sg.candidate_number), 
        (SELECT subject
        FROM ebacc_bucket_2 as sub
        WHERE sub.candidate_number = sg.candidate_number),
        (SELECT subject
        FROM ebacc_bucket_3 as sub
        WHERE sub.candidate_number = sg.candidate_number)
        )
        AND candidate_number = first_5.candidate_number
        ORDER BY grade DESC, sg.subject
        LIMIT 1 OFFSET 2 ) AS sub
    USING (candidate_number)
    )

SELECT sg.subject, COUNT(*) AS count_dropped
FROM student_grades as sg
WHERE sg.subject NOT IN (
    SELECT subject
    FROM ebacc_bucket_1 as sub
    WHERE sub.candidate_number = sg.candidate_number
    )
AND sg.subject NOT IN (
    SELECT subject
    FROM ebacc_bucket_2 as sub
    WHERE sub.candidate_number = sg.candidate_number
    )
AND sg.subject NOT IN (
    SELECT subject
    FROM ebacc_bucket_3 as sub
    WHERE sub.candidate_number = sg.candidate_number
    )
AND sg.subject NOT IN (
    SELECT subject
    FROM open_bucket_1 as sub
    WHERE sub.candidate_number = sg.candidate_number
    )
AND sg.subject NOT IN (
    SELECT subject
    FROM open_bucket_2 as sub
    WHERE sub.candidate_number = sg.candidate_number
    )
AND sg.subject NOT IN (
    SELECT subject
    FROM open_bucket_3 as sub
    WHERE sub.candidate_number = sg.candidate_number
    )
AND sg.subject NOT IN (
    SELECT subject
    FROM top_eng
    WHERE top_eng.candidate_number = sg.candidate_number)
AND sg.subject NOT IN ('gcse_(9-1)_mathematics')
GROUP BY subject
ORDER BY count_dropped DESC
```


<span style="None">Running query in &#x27;duckdb://&#x27;</span>





<table>
    <thead>
        <tr>
            <th>subject</th>
            <th>count_dropped</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>gcse_(9-1)_english_literature</td>
            <td>34</td>
        </tr>
        <tr>
            <td>gcse_(9-1)_english_language</td>
            <td>15</td>
        </tr>
        <tr>
            <td>gcse_(9-1)_spanish</td>
            <td>13</td>
        </tr>
        <tr>
            <td>gcse_(9-1)_german</td>
            <td>12</td>
        </tr>
        <tr>
            <td>gcse_(9-1)_psychology</td>
            <td>11</td>
        </tr>
        <tr>
            <td>gcse_(9-1)_physics</td>
            <td>10</td>
        </tr>
        <tr>
            <td>gcse_(9-1)_bus._studs:single</td>
            <td>10</td>
        </tr>
        <tr>
            <td>gcse_(9-1)_geography</td>
            <td>9</td>
        </tr>
        <tr>
            <td>gcse_(9-1)_sport/p.e._studies</td>
            <td>8</td>
        </tr>
        <tr>
            <td>gcse_(9-1)_history</td>
            <td>8</td>
        </tr>
        <tr>
            <td>gcse_(9-1)_design&_technology</td>
            <td>7</td>
        </tr>
        <tr>
            <td>gcse_(9-1)_com.stds/computing</td>
            <td>5</td>
        </tr>
        <tr>
            <td>gcse_(9-1)_music</td>
            <td>2</td>
        </tr>
        <tr>
            <td>gcse_(9-1)_french</td>
            <td>2</td>
        </tr>
        <tr>
            <td>camnatcer_childcare_skills</td>
            <td>2</td>
        </tr>
        <tr>
            <td>btc1awd_music_studies</td>
            <td>1</td>
        </tr>
        <tr>
            <td>gcse_(9-1)_drama_&_theat.stds</td>
            <td>1</td>
        </tr>
        <tr>
            <td>gcse_(9-1)_chemistry</td>
            <td>1</td>
        </tr>
        <tr>
            <td>gcse_(9-1)_art&des_:_fine_art</td>
            <td>1</td>
        </tr>
        <tr>
            <td>btctechawd_small_business</td>
            <td>1</td>
        </tr>
        <tr>
            <td>gcse_(9-1)_religious_studies</td>
            <td>1</td>
        </tr>
        <tr>
            <td>btctechawd_multimedia</td>
            <td>1</td>
        </tr>
    </tbody>
</table>




```sql
%%sql
-- Error checking, table shows results that do not match the official figures, empty table is good!
SELECT candidate_number,
    E_1_pts AS calc_e_1,
    "1st_ebacc"::numeric AS actual_e_1,
    E_2_pts AS calc_e_2,
    "2nd_ebacc"::numeric AS actual_e_2,
    E_3_pts AS calc_e3,
    "3rd_ebacc"::numeric AS actual_e_3,
    O_1_pts AS calc_o_1,
    "1st_open"::numeric AS actual_o_1,
    O_2_pts AS calc_o_2,
    "2nd_open"::numeric AS actual_o_2,
    O_3_pts AS calc_o_3,
    "3rd_open"::numeric AS actual_o_3
FROM A8_table
INNER JOIN attainment_8_scores as A8
USING (candidate_number) 
WHERE O_1_pts <> "1st_open"::numeric
OR O_2_pts <> "2nd_open"::numeric
OR O_3_pts <> "3rd_open"::numeric
OR E_1_pts <> "1st_ebacc"::numeric
OR E_2_pts <> "2nd_ebacc"::numeric
OR E_3_pts <> "3rd_ebacc"::numeric
```


<span style="None">Generating CTE with stored snippets: &#x27;A8_table&#x27;</span>



<span style="None">Running query in &#x27;duckdb://&#x27;</span>





<table>
    <thead>
        <tr>
            <th>candidate_number</th>
            <th>calc_e_1</th>
            <th>actual_e_1</th>
            <th>calc_e_2</th>
            <th>actual_e_2</th>
            <th>calc_e3</th>
            <th>actual_e_3</th>
            <th>calc_o_1</th>
            <th>actual_o_1</th>
            <th>calc_o_2</th>
            <th>actual_o_2</th>
            <th>calc_o_3</th>
            <th>actual_o_3</th>
        </tr>
    </thead>
    <tbody>
    </tbody>
</table>


