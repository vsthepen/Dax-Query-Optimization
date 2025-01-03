# Dax-Query-Optimization

Here are examples of inefficient queries I have encountered that caused slow report performance when working with large datasets, along with the optimizations I implemented to enhance performance.

It's important to note that the optimized queries are based on the specific calculation context. These optimizations may not be effective if applied in a different context.

## 1. Calculating Total Sales

#### Inefficient Query

Total Sales = SUMX(Sales, Sales[UnitPrice] * Sales[Quantity]) 

**Why:** This query recalculates UnitPrice * Quantity row by row unnecessarily.

#### Optimised Query

Pre-calculate LineTotalSales in Power Query, then sum the column.

Total Sales = SUM(Sales[LineTotalSales])  

## 2. Calculating Distinct Customers

#### Inefficient Query

a)	Distinct Customers = COUNTROWS(SUMMARIZE(Sales, Sales[CustomerID])) 

**Why:** Creating large tables with SUMMARIZE() can consume significant memory, especially if many unique combinations are generated. It’s best used for summarizing specific and limited data rather than creating large aggregation tables.

b)	Distinct Customers = SUMX(VALUES(Sales[CustomerID],1))

**Why:** VALUES generates a temporary table containing all distinct values in memory, which increases overhead. SUMX iterates row-by-row over the generated table, introducing additional computational load.

#### Optimised Query

Use DISTINCTCOUNT directly for efficiency. It handles millions of rows more efficiently with lower memory consumption.

Distinct Customers = DISTINCTCOUNT(Sales[CustomerID]) 

## 3. Filtered Total Sales by Region

#### Inefficient Query

Sales by Region = SUMX(FILTER(Sales, Sales[Region] = "North"), [Total Sales]) 

**Why:** The FILTER function creates a row context by scanning the Sales table to find rows where Sales[Region] = "North". This operation creates an intermediate table in memory containing only the rows where the condition is true. Creating an intermediate table adds additional memory and computational overhead.
 
#### Optimised Query

Leverage CALCULATE function to filter directly. CALCULATE modifies the filter context by applying the condition Sales[Region] = "North" directly to the data model.

Sales by Region = CALCULATE([Total Sales], Sales[Region] = "North")  

## 4. Cumulative Sales

#### Inefficient Query

Cumulative Sales = SUMX(FILTER(Sales, Sales[Date] <= MAX(Sales[Date])), [Total Sales])  

**Why:** This query iterates over all rows unnecessarily.

#### Optimised Query

Use CALCULATE and DATESBETWEEN for better performance.

Cumulative Sales = CALCULATE([Total Sales], DATESBETWEEN(Dates[Date], BLANK(), MAX(Dates[Date])))  

## 5. Average Sales Per Customer

#### Inefficient Query

Avg Sales Per Customer = SUMX(SUMMARIZE(Sales, Sales[CustomerID], "Total", SUM(Sales[LineTotal])), [Total]) / DISTINCTCOUNT(Sales[CustomerID]) 

**Why:** The goal is to find the total sales and divide it by the distinct customer count. This query performs redundant steps (SUMMARIZE and SUMX) to achieve the same result that could be calculated directly.

#### Optimised Query

Avoids unnecessary operations and does not generate any intermediate tables or temporary row contexts. This keeps memory usage low, making it better suited for large datasets.

Avg Sales Per Customer = DIVIDE([Total Sales], DISTINCTCOUNT(Sales[CustomerID]), 0) 

## 6. Top N Products by Sales

#### Inefficient Query

Top Products = TOPN(5, SUMMARIZE(Sales, Products[ProductName], "Sales", SUM(Sales[LineTotal])), [Sales], DESC)

**Why:** Uses SUMMARIZE, which is expensive to process.

#### Optimised Query

This query leverages relationships between the Sales and Products tables to retrieve pre-aggregated totals.

Top Products = TOPN(5, Products, [Total Sales], DESC) 

## 7. Calculating Profit Margin

#### Inefficient Query

Profit Margin = DIVIDE(SUMX(Sales, Sales[LineTotal] - Sales[Cost]), SUMX(Sales, Sales[LineTotal]))

**Why:** Does row-by-row calculations multiple times.

#### Optimised Query

Use pre-aggregated values for efficiency.

Profit Margin = DIVIDE([Total Profit], [Total Sales])  

## 8. Year-to-Date Sales

#### Inefficient Query

YTD Sales = CALCULATE(SUM(Sales[LineTotal]), 

FILTER(Dates, Dates[Date] <= MAX(Dates[Date]) && Dates[Year] = YEAR(MAX(Dates[Date]))))  

**Why:** For each row in the Dates table, it evaluates:

•	Dates[Date] <= MAX(Dates[Date]): A range condition

•	Dates[Year] = YEAR(MAX(Dates[Date])): A conditional check for the year requiring iteration over the entire Dates table, which is computationally expensive.

#### Optimised Query

Leverage built-in functions or direct filtering that avoids row context

a)	YTD Sales = CALCULATE( SUM(Sales[LineTotal]),DATESYTD(Dates[Date]), Dates[Year] = YEAR(MAX(Dates[Date])))

Using MAX(Dates[Date]) and YEAR(MAX(Dates[Date])) multiple times causes redundant calculations. By storing them in variables, we reduce overhead.

b)	YTD Sales = 

VAR MaxDate = MAX(Dates[Date])

VAR CurrentYear = YEAR(MaxDate)

RETURN

CALCULATE(

    SUM(Sales[LineTotal]),
    
    FILTER(
    
        Dates,
        
        Dates[Date] <= MaxDate && Dates[Year] = CurrentYear ))

## 9. Sales Growth Percentage

#### Inefficient Query

Sales Growth = DIVIDE([Total Sales] - CALCULATE([Total Sales], Dates[Year] = YEAR(MAX(Dates[Year])) - 1), 

CALCULATE([Total Sales], Dates[Year] = YEAR(MAX(Dates[Year])) - 1))  

**Why:** The MAX(Dates[Year]) function is evaluated twice for the same context, leading to unnecessary repeated evaluations.

#### Optimised Query

To improve the performance, we can use variables to store the previous and current year sales values so that they are calculated only once. This removes the need for multiple calls to CALCULATE and MAX(Dates[Year]), which will reduce redundant computations.

Sales Growth = 

VAR CurrentYearSales = [Total Sales]

VAR PreviousYearSales = 

    CALCULATE(
    
        [Total Sales],
        
        FILTER(
        
            Dates,
            
            Dates[Year] = YEAR(MAX(Dates[Year])) – 1))
            
RETURN

    DIVIDE(CurrentYearSales - PreviousYearSales, PreviousYearSales)
 
## 10. Sales Contribution by Product

#### Inefficient Query

Sales by Products = DIVIDE(SUMX(Sales, Sales[LineTotal]), SUM(Sales[LineTotal]))

**Why:** Iterates over each row calculating the LineTotal for each row individually.
 
#### Optimised Query

This query avoids the iteration and directly calculates totals at a higher aggregation level, leading to faster execution.

Sales by Products = DIVIDE([Total Sales], CALCULATE([Total Sales], ALL(Products))) 

Optimisations often cut query times by **30-70%**, depending on complexity. In addition, optimised queries consume **20-50%** less memory by avoiding unnecessary row evaluations.

Optimisations also improve scalability, enabling models to handle datasets that are **2-5** times larger without performance degradation.


