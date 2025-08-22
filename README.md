# RFM-Segmentation-Using-SQL
This project analyzes sales data to calculate Recency, Frequency, and Monetary values, assigns RFM scores, and segments customers into groups (Champions, Loyal, At Risk, Lost, etc.), providing insights into spending, orders, and customer behavior.
``` sql
USE RFM_SALES;

SELECT * FROM SALES_DATA;

-- DROP TABLE SAMPLE_SALES_DATA;
SELECT MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y')) FROM SALES_DATA; -- 2005-05-31 [LAST BUSINESS DAY]
SELECT MIN(STR_TO_DATE(ORDERDATE, '%d/%m/%y')) FROM SALES_DATA; -- 2003-01-06

SELECT
	CUSTOMERNAME,
    ROUND(SUM(SALES),0) AS CLV,
    COUNT(DISTINCT ORDERNUMBER) AS FREQUENCY,
    SUM(QUANTITYORDERED) AS TOTAL_QTY_ORDERED,
    MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y')) AS CUSTOMER_LAST_TRANSACTION_DATE,
    DATEDIFF((SELECT MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y')) FROM SALES_DATA), MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y'))) AS CUSTOMER_RECENCY
FROM SALES_DATA
GROUP BY CUSTOMERNAME;

-- RFM SEGMENTATION
DROP VIEW IF EXISTS RFM_SEGMENTATION_DATA;


CREATE VIEW RFM_SEGMENTATION_DATA AS
WITH CLV AS 
(SELECT
	CUSTOMERNAME,
	MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y')) AS CUSTOMER_LAST_TRANSACTION_DATE,
    DATEDIFF((SELECT MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y')) FROM SALES_DATA), MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y'))) AS RECENCY_VALUE,
    COUNT(DISTINCT ORDERNUMBER) AS FREQUENCY_VALUE,
    SUM(QUANTITYORDERED) AS TOTAL_QTY_ORDERED,
    ROUND(SUM(SALES),0) AS MONETARY_VALUE
FROM SALES_DATA
GROUP BY CUSTOMERNAME),

RFM_SCORE AS
(SELECT 
	C.*,
    NTILE(5) OVER(ORDER BY RECENCY_VALUE DESC) AS R_SCORE,
    NTILE(5) OVER(ORDER BY FREQUENCY_VALUE ASC) AS F_SCORE,
    NTILE(5) OVER(ORDER BY MONETARY_VALUE ASC) AS M_SCORE
FROM CLV AS C),

RFM_COMBINATION AS
(SELECT
	R.*,
    R_SCORE + F_SCORE + M_SCORE AS TOTAL_RFM_SCORE,
    CONCAT_WS('', R_SCORE, F_SCORE, M_SCORE) AS RFM_COMBINATION
FROM RFM_SCORE AS R)

SELECT
	RC.*,
    CASE
		WHEN RFM_COMBINATION IN (541,542, 543, 544,545,551, 552, 553,554, 555) THEN "Champions"
        WHEN RFM_COMBINATION IN (511,512,513,514,515,521,522,523,524,525,531,532,533,534,535) THEN "potential loyalists"
        WHEN RFM_COMBINATION IN (341, 342, 343, 344, 345,351, 352, 353, 354, 355,441,442,443,444,445,451,452,453.454,455) THEN 'Loyal Customers'
        WHEN RFM_COMBINATION IN (411,412,413,414,415,421,422,423,424,425,431,432,433,434,435) THEN 'Promising Customers'
        WHEN RFM_COMBINATION IN (311,312,313,314,315,321,322,323,324,325,331,332,333,334,335) THEN 'Needs Attention'
        WHEN RFM_COMBINATION IN (131,132,133,134,135,141,142,143,144,145,151,152,153,154,155,231,232,233,234,235,241,242,243,244,245,251,252,253,254,255) THEN 'At risk'
        WHEN RFM_COMBINATION IN (111,112,113,114,115,121,122,123,124,125,211,212,213,214,215,221,222,223,224,225) THEN "Lost"
        END AS CUSTOMER_SEGMENT
FROM RFM_COMBINATION RC;


SELECT * FROM RFM_SEGMENTATION_DATA;

SELECT
	CUSTOMER_SEGMENT,
    SUM(MONETARY_VALUE) AS TOTAL_SPENDING,
    ROUND(AVG(MONETARY_VALUE),0) AS AVERAGE_SPENDING,
    SUM(FREQUENCY_VALUE) AS TOTAL_ORDER,
    SUM(TOTAL_QTY_ORDERED) AS TOTAL_QTY_ORDERED
FROM RFM_SEGMENTATION_DATA
GROUP BY CUSTOMER_SEGMENT;

# 📊 RFM Segmentation Results

The following table summarizes customer segments based on **Recency, Frequency, and Monetary (RFM) analysis**:

| Customer Segment     | Total Spending | Average Spending | Total Orders | Total Qty Ordered |
|----------------------|---------------:|-----------------:|-------------:|------------------:|
| Promising Customers  | 299,639        | 74,910           | 11           | 3,067             |
| Needs Attention      | 1,146,733      | 81,910           | 42           | 11,702            |
| Lost                 | 2,653,233      | 78,010           | 73           | 26,272            |
| Loyal Customers      | 2,022,245      | 112,347          | 66           | 19,346            |
| Potential Loyalists  | 282,165        | 94,055           | 9            | 2,781             |
| Champions            | 3,169,655      | 211,310          | 91           | 31,414            |
| At Risk              | 459,856        | 114,964          | 15           | 4,485             |

---

### 🔑 Key Insights
- **Champions** spend the most (over 3.1M) with the highest average order value.  
- **Lost Customers** place many orders (73) but contribute less per order.  
- **Loyal Customers** maintain strong spending and consistent ordering.  
- **Needs Attention & At Risk** groups may require retention strategies.  
- **Promising & Potential Loyalists** show growth potential if nurtured.  

```
