# Proactive Quality Control: Detecting Manufacturing Anomalies with SQL Window Functions

## Project Summary

This project demonstrates the use of advanced SQL, specifically window functions, to implement a Statistical Process Control (SPC) system for a manufacturing line. By analyzing historical production data, I developed an automated alert system to identify parts that fall outside acceptable quality limits. The final analysis pinpointed underperforming machinery, providing actionable recommendations to improve production consistency and reduce waste.

**Tools Used:** SQL (Window Functions, CTEs, CASE Statements)

**Skills Demonstrated:** Data Analysis, Statistical Process Control (SPC), Anomaly Detection, Business Insight Generation

---

## 1. Situation

A manufacturing company was struggling with inconsistent product quality, leading to increased material waste and potential customer dissatisfaction. The quality assurance (QA) team lacked a data-driven method for monitoring production output in real-time. Machine adjustments were often based on intuition rather than statistical evidence, making it difficult to identify the root cause of quality issues. The primary business goal was to establish a systematic approach to monitor machine performance and ensure all products meet strict dimensional specifications.

---

## 2. Task

As a Data Analyst, my objective was to develop an automated detection system using historical manufacturing data. My key responsibilities included:

*   **Establishing Dynamic Control Limits:** Calculate rolling quality control limits (Upper and Lower Control Limits) for product height, specific to each machine operator.
*   **Automating Anomaly Detection:** Write an efficient SQL query to automatically flag any product that deviates from these calculated limits.
*   **Performance Analysis:** Aggregate the results to identify which machine operators had the highest rate of anomalies, thereby highlighting machinery in need of maintenance or recalibration.

---

## 3. Action

My analytical process was executed entirely within SQL, focusing on efficiency and readability.

### Step 1: Calculating Rolling Statistics with Window Functions

I began by writing a query using a Common Table Expression (CTE) to calculate the rolling average and rolling standard deviation for the height of parts produced.

*   **`PARTITION BY operator`**: This was crucial to ensure that calculations were performed independently for each machine, accurately reflecting its unique performance signature.
*   **`ROWS BETWEEN 4 PRECEDING AND CURRENT ROW`**: I defined a moving window of the 5 most recent parts. This "rolling" approach ensures that the control limits are dynamic and adapt to the machine's immediate performance, rather than being skewed by outdated data.
*   **`WHERE row_number >= 5`**: I excluded the initial rows for each operator to ensure that every calculation was based on a complete window of 5 data points.

```sql
/* Query 1: Calculate rolling statistics and define control limits */
WITH ControlLimits AS (
    SELECT
        item_no,
        operator,
        height,
        ROW_NUMBER() OVER (PARTITION BY operator ORDER BY item_no) AS row_number,
        AVG(height) OVER (
            PARTITION BY operator 
            ORDER BY item_no 
            ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
        ) AS avg_height,
        STDDEV_SAMP(height) OVER (
            PARTITION BY operator 
            ORDER BY item_no 
            ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
        ) AS stddev_height
    FROM
        manufacturing_parts
)
SELECT
    operator,
    row_number,
    height,
    avg_height,
    stddev_height,
    (avg_height + 3 * (stddev_height / SQRT(5))) AS ucl,
    (avg_height - 3 * (stddev_height / SQRT(5))) AS lcl,
    CASE
        WHEN height > (avg_height + 3 * (stddev_height / SQRT(5)))
          OR height < (avg_height - 3 * (stddev_height / SQRT(5)))
        THEN TRUE
        ELSE FALSE
    END AS alert
FROM
    ControlLimits
WHERE
    row_number >= 5
ORDER BY
    item_no; 
```

### Step 2: Aggregating Results to Pinpoint Problem Areas

A second query aggregated the anomaly data to calculate the total parts checked, total anomalies, and the anomaly rate percentage for each machine operator. This transformed the raw alert data into a high-level performance summary for management.
```sql
/* Query 2: Aggregate alerts to calculate operator performance */
SELECT
    operator,
    COUNT(*) AS total_parts_checked,
    SUM(CASE WHEN is_anomaly = TRUE THEN 1 ELSE 0 END) AS anomaly_count,
    ROUND(AVG(CASE WHEN is_anomaly = TRUE THEN 1.0 ELSE 0.0 END) * 100, 2) AS anomaly_rate_percent
FROM
    alerts_table -- This would be the output of the first query
GROUP BY
    operator
ORDER BY
    anomaly_rate_percent DESC;
```

---

## 4. Result

The analysis yielded clear, data-driven insights into the performance of the manufacturing line.

### Key Findings:

*   **High-Alert Operators**: **Operator Op-4** was identified as the most inconsistent, with an alert rate of **23.5%**. **Op-5 (20.7%)**, **Op-2 (19.1%)**, and **Op-16 (19.1%)** also showed significantly high rates of deviation.
*   **Top-Performing Operators**: Conversely, **Operator Op-18** was the most stable with an alert rate of only **4.0%**, followed by **Op-12 (6.3%)** and **Op-6 (7.1%)**.

### Data Visualization:
<img width="600" height="350" alt="{C120E15C-D9DD-472A-B2B6-5C9AF499C0B4}" src="https://github.com/user-attachments/assets/1774dd9c-e370-4de4-bc98-0a6121b4c755" />


### Actionable Recommendations & Business Impact:

*  **Prioritize Maintenance**: Immediately schedule inspections and recalibration for the machines handled by operators **Op-4, Op-5, and Op-2**. This targeted approach focuses resources where they are most needed.
*  **Standardize Best Practices**: Investigate the processes and techniques used by the top-performing operators **(Op-18, Op-12, Op-6)**. Their methods could be documented and used as a benchmark for training across the entire production floor.

