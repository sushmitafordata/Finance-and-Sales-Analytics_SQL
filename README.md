# Finance-and-Sales-Analytics_SQL

**1. Getting the customer code of Croma India**
```
SELECT *
  FROM dim_customer
  WHERE customer like "%croma%"
  AND market="india";
```
  Query Result:


![Screenshot 2024-05-12 at 10 40 26 PM](https://github.com/sushmitafordata/Finance-and-Sales-Analytics_SQL/assets/135410984/d963790d-030c-4dde-96d6-d2e244f8444f)




**2. Getting all the sales transaction data from fact_sales_monthly table for (croma: 90002002) in the fiscal_year 2021**


```

SELECT * FROM fact_sales_monthly 
	WHERE 
            customer_code=90002002 AND
            YEAR(DATE_ADD(date, INTERVAL 4 MONTH))=2021 
	ORDER BY date asc
	LIMIT 100000;

```
  Query Result:



![Screenshot 2024-05-12 at 10 41 20 PM](https://github.com/sushmitafordata/Finance-and-Sales-Analytics_SQL/assets/135410984/d41e0fe6-645e-4f1f-bbcb-170f89abae4d)

**3. Create a function 'get_fiscal_year' to get fiscal year by passing the date**

```
	CREATE FUNCTION `get_fiscal_year`(calendar_date DATE) 
	RETURNS int
    	DETERMINISTIC
	BEGIN
        	DECLARE fiscal_year INT;
        	SET fiscal_year = YEAR(DATE_ADD(calendar_date, INTERVAL 4 MONTH));
        	RETURN fiscal_year;
	END
```

Query Result:

![Screenshot 2024-05-12 at 10 51 11 PM](https://github.com/sushmitafordata/Finance-and-Sales-Analytics_SQL/assets/135410984/bcdcea6b-dd14-42a0-b812-e32cf6711955)

 **4. Implementing the function created in the previous step**

 ```
	SELECT * FROM fact_sales_monthly 
	WHERE 
            customer_code=90002002 AND
            get_fiscal_year(date)=2021 
	ORDER BY date asc
	LIMIT 100000;
```
Query Result:

![Screenshot 2024-05-12 at 10 54 10 PM](https://github.com/sushmitafordata/Finance-and-Sales-Analytics_SQL/assets/135410984/5f53c57b-e90f-4a19-9b91-d19235e1e246)



### GROSS SALES REPORT :Monthly Product Transactions

**5. Performing joins to pull product information**

```
SELECT s.date, s.product_code, p.product, p.variant, s.sold_quantity 
	FROM fact_sales_monthly s
	JOIN dim_product p
        ON s.product_code=p.product_code
	WHERE 
            customer_code=90002002 AND 
    	    get_fiscal_year(date)=2021     
	LIMIT 1000000;
```
  Query Result:


![Screenshot 2024-05-12 at 11 34 36 PM](https://github.com/sushmitafordata/Finance-and-Sales-Analytics_SQL/assets/135410984/c5b4ac32-0962-4f4c-ab1e-5a4321d70f01)


**6. Performing join with 'fact_gross_price' table with the above query and generating required fields**

```
SELECT 
    	    s.date, 
            s.product_code, 
            p.product, 
            p.variant, 
            s.sold_quantity, 
            g.gross_price,
            ROUND(s.sold_quantity*g.gross_price,2) as gross_price_total
	FROM fact_sales_monthly s
	JOIN dim_product p
            ON s.product_code=p.product_code
	JOIN fact_gross_price g
            ON g.fiscal_year=get_fiscal_year(s.date)
    	AND g.product_code=s.product_code
	WHERE 
    	    customer_code=90002002 AND 
            get_fiscal_year(s.date)=2021     
	LIMIT 1000000;
```

Query Result:

![Screenshot 2024-05-12 at 11 41 02 PM](https://github.com/sushmitafordata/Finance-and-Sales-Analytics_SQL/assets/135410984/e8bd314d-9d87-44e8-b82b-36a806e18358)


### GROSS SALES REPORT :Total Sales Amount

**7. Generate monthly gross sales report for Croma India for all the years**
```

	SELECT 
            s.date, 
    	    SUM(ROUND(s.sold_quantity*g.gross_price,2)) as monthly_sales
	FROM fact_sales_monthly s
	JOIN fact_gross_price g
        ON g.fiscal_year=get_fiscal_year(s.date) AND g.product_code=s.product_code
	WHERE 
             customer_code=90002002
	GROUP BY date;
```
Query Result:

![Screenshot 2024-05-12 at 11 43 24 PM](https://github.com/sushmitafordata/Finance-and-Sales-Analytics_SQL/assets/135410984/372ad19b-c14d-4ff3-af84-7bcc90c09dd6)

#### Monthly Gross Sales Report

**8. Generate monthly gross sales report for any customer using stored procedure**

```
	CREATE PROCEDURE `get_monthly_gross_sales_for_customer`(
        	in_customer_codes TEXT
	)
	BEGIN
        	SELECT 
                    s.date, 
                    SUM(ROUND(s.sold_quantity*g.gross_price,2)) as monthly_sales
        	FROM fact_sales_monthly s
        	JOIN fact_gross_price g
               	    ON g.fiscal_year=get_fiscal_year(s.date)
                    AND g.product_code=s.product_code
        	WHERE 
                    FIND_IN_SET(s.customer_code, in_customer_codes) > 0
        	GROUP BY s.date
        	ORDER BY s.date DESC;
	END;
```

Query Result:


![Screenshot 2024-05-12 at 11 48 21 PM](https://github.com/sushmitafordata/Finance-and-Sales-Analytics_SQL/assets/135410984/f6fc23a5-45c9-4339-8db9-102b906b6377)

**9. Including pre-invoice deduction information for different customers**

```
SELECT 
    	   s.date, 
           s.product_code, 
           p.product, 
	   p.variant, 
           s.sold_quantity, 
           g.gross_price as gross_price_per_item,
           ROUND(s.sold_quantity*g.gross_price,2) as gross_price_total,
           pre.pre_invoice_discount_pct
	FROM fact_sales_monthly s
	JOIN dim_product p
            ON s.product_code=p.product_code
	JOIN fact_gross_price g
    	    ON g.fiscal_year=s.fiscal_year
    	    AND g.product_code=s.product_code
	JOIN fact_pre_invoice_deductions as pre
            ON pre.customer_code = s.customer_code AND 
            pre.fiscal_year=s.fiscal_year	
	WHERE  
    	    s.fiscal_year=2021     
	LIMIT 1000000;
```
Query Result:



![Screenshot 2024-05-13 at 12 07 15 AM](https://github.com/sushmitafordata/Finance-and-Sales-Analytics_SQL/assets/135410984/cff4d1b8-8a97-4970-83b2-80b4cc46d9ea)

**10. Calculating net_invoice_sales amount using the CTE's**
```
	WITH cte1 AS (
		SELECT 
    		    s.date, 
    		    s.customer_code,
    		    s.product_code, 
                    p.product, p.variant, 
                    s.sold_quantity, 
                    g.gross_price as gross_price_per_item,
                    ROUND(s.sold_quantity*g.gross_price,2) as gross_price_total,
                    pre.pre_invoice_discount_pct
		FROM fact_sales_monthly s
		JOIN dim_product p
        		ON s.product_code=p.product_code
		JOIN fact_gross_price g
    			ON g.fiscal_year=s.fiscal_year
    			AND g.product_code=s.product_code
		JOIN fact_pre_invoice_deductions as pre
        		ON pre.customer_code = s.customer_code AND
    			pre.fiscal_year=s.fiscal_year
		WHERE 
    			s.fiscal_year=2021) 
	SELECT 
      	    *, 
    	    (gross_price_total-pre_invoice_discount_pct*gross_price_total) as net_invoice_sales
	FROM cte1
	LIMIT 1500000;
```
Query Result:


![Screenshot 2024-05-13 at 12 15 26 AM](https://github.com/sushmitafordata/Finance-and-Sales-Analytics_SQL/assets/135410984/c3938655-45bd-447a-8251-e55545d19aa1)

