Symptom:

Read the cell `6.2 External cultural alignment — ablation across all variants (paper Fig. 4)` in the `final\ICOCO_notebook_Colab.ipynb`

You can see all ablations appear in the same number = Full Fusion

Basic hypothesis:
- The vector is generated incorectly
+ Vector in old code, read about this: `old_code\10_geo_vector`, `old_code\11_entity_vector`, `old_code\12_odor_vector`

+ Vector in new code: `7_create_npy_files`

- Querying and matching incorectly:
+ In the part 1.8 of the notebook
+ the cell that contains this code
```
%%sql
SELECT 'Task A' AS task, count(*) AS n_inputs,
       sum(CASE WHEN len(final_list) > 0 THEN 1 ELSE 0 END) AS with_output
FROM metrics.v_ingredient_gold
UNION ALL
SELECT 'Task B', count(*),
       sum(CASE WHEN len(final_list) > 0 THEN 1 ELSE 0 END)
FROM metrics.v_scientific_gold;
```
+ this cell is suspicious, same to this cell in the part 1.9
```
%%sql
-- Task A · USDA full paper universe (foundation latest + legacy)
WITH valid AS (
    SELECT custom_id, final_list
    FROM metrics.v_ingredient_gold
    WHERE len(final_list) > 0
),
flagged AS (
    SELECT v.custom_id,
           max(CASE WHEN e.elem <> ''
                     AND EXISTS (SELECT 1 FROM metrics.v_ref_food_full r
                                 WHERE r.lower_desc LIKE '%' || e.elem || '%')
                    THEN 1 ELSE 0 END) AS is_queryable
    FROM valid v
    CROSS JOIN UNNEST(v.final_list) AS e(elem)
    GROUP BY v.custom_id
)
SELECT
    count(*)             AS with_output,
    sum(is_queryable)    AS queryable,
    round(100.0 * sum(is_queryable) / count(*), 2) AS queryable_pct
FROM flagged;
```

and the cell

```
%%sql
-- Task A · USDA foundation_food ONLY (mission-strict variant)
WITH valid AS (
    SELECT custom_id, final_list
    FROM metrics.v_ingredient_gold
    WHERE len(final_list) > 0
),
flagged AS (
    SELECT v.custom_id,
           max(CASE WHEN e.elem <> ''
                     AND EXISTS (SELECT 1 FROM metrics.v_ref_food_foundation r
                                 WHERE r.lower_desc LIKE '%' || e.elem || '%')
                    THEN 1 ELSE 0 END) AS is_queryable
    FROM valid v
    CROSS JOIN UNNEST(v.final_list) AS e(elem)
    GROUP BY v.custom_id
)
SELECT
    count(*)             AS with_output,
    sum(is_queryable)    AS queryable,
    round(100.0 * sum(is_queryable) / count(*), 2) AS queryable_pct
FROM flagged;
```

+ the output is not right

+ old code should read: `old_code\13_FoodUSDA_queryable`

+ new code should read: `5_confirm_methodology_number` and `investigate_data_mismatch`

Your mission: Fix this notebook and write a markdown file as note in the folder `final`

You should read other files in the folder `final` to understand more about the situation