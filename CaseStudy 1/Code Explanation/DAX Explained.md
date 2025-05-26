```m
partition _Measures = m
    mode: import
    source =
            let
                Source = Table.FromRows(Json.Document(Binary.Decompress(Binary.FromText("i44FAA==", BinaryEncoding.Base64), Compression.Deflate)), let _t = ((type nullable text) meta [Serialized.Text = true]) in type table [Column1 = _t]),
                #"Changed Type" = Table.TransformColumnTypes(Source,{{"Column1", type text}}),
                #"Removed Columns" = Table.RemoveColumns(#"Changed Type",{"Column1"})
            in
                #"Removed Columns"
```
*   `Binary.FromText("i44FAA==", BinaryEncoding.Base64)`: "i44FAA==" is the Base64 representation of a very small, compressed piece of data.
*   `Binary.Decompress(..., Compression.Deflate)`: Decompresses this binary data.
*   `Json.Document(...)`: Parses the decompressed content as JSON. The string "i44FAA==" typically decodes and decompresses to something like `[[]]` or `[{}]` representing a minimal JSON structure for an empty table or a table with one empty row and no defined columns.
*   `Table.FromRows(...)`: Creates a table from this. It often results in a table with one column named "Column1" and possibly one empty row.
*   `#"Removed Columns" = Table.RemoveColumns(#"Changed Type",{"Column1"})`: This final step removes "Column1", leaving an empty table with no columns and no rows. This is the standard and correct way to create a measure table in Power BI if doing it via M.

Now, for the DAX measures:

---

1.  **`'Total Cumulative Return'`**

    ```dax
    measure 'Total Cumulative Return' =
        CALCULATE (
            SUM ( FactProductPerformance[Value] ),
            FactProductPerformance[Metric Type] = "Cumulative Return"
        )
    formatString: 0.00%;-0.00%;0.00%
    ```

    *   **Purpose:** This measure calculates the sum of values from the `FactProductPerformance` table, but only for rows where the `Metric Type` is "Cumulative Return". This implies that your source data already contains pre-calculated cumulative return values.
    *   **`CALCULATE (expression, filter1, filter2, ...)`:** This is the most powerful and common DAX function. It evaluates an `expression` in a modified filter context.
        *   **`expression`**: `SUM ( FactProductPerformance[Value] )`
            *   `SUM ( FactProductPerformance[Value] )`: This aggregates (sums up) all values in the `Value` column of the `FactProductPerformance` table *that are visible in the current filter context after `CALCULATE`'s modifications*.
        *   **`filter1`**: `FactProductPerformance[Metric Type] = "Cumulative Return"`
            *   This is a boolean filter argument. It modifies the filter context by adding a new filter condition: the `Metric Type` column in the `FactProductPerformance` table must be equal to "Cumulative Return".
            *   It's syntactic sugar for `FILTER(ALL(FactProductPerformance[Metric Type]), FactProductPerformance[Metric Type] = "Cumulative Return")` or, more accurately if you intend to keep other filters on `FactProductPerformance`, `KEEPFILTERS(FactProductPerformance[Metric Type] = "Cumulative Return")`. In its simple form, it overrides any existing filter on `FactProductPerformance[Metric Type]` and applies this new one.
    *   **How it works:**
        1.  `CALCULATE` takes the current filter context (e.g., from slicers, visual rows/columns).
        2.  It applies the filter `FactProductPerformance[Metric Type] = "Cumulative Return"`.
        3.  Then, it evaluates `SUM ( FactProductPerformance[Value] )` within this new, modified filter context. So, it sums only those `Value`s associated with "Cumulative Return".
    *   **`formatString: 0.00%;-0.00%;0.00%`**: This defines how the result should be displayed: as a percentage with two decimal places. The three parts are for positive numbers; negative numbers; zero.

---

2.  **`'Total Monthly Return'`**

    ```dax
    measure 'Total Monthly Return' =
        CALCULATE (
            SUM ( FactProductPerformance[Value] ),
            FILTER (
                FactProductPerformance,
                FactProductPerformance[Metric Type] = "Monthly Return"
            )
        )
    formatString: 0.00%;-0.00%;0.00%
    ```

    *   **Purpose:** This measure calculates the sum of values from `FactProductPerformance` for rows where `Metric Type` is "Monthly Return".
    *   **`CALCULATE (expression, filter1)`:**
        *   **`expression`**: `SUM ( FactProductPerformance[Value] )` (Same as above).
        *   **`filter1`**: `FILTER ( FactProductPerformance, FactProductPerformance[Metric Type] = "Monthly Return" )`
            *   `FILTER (table, filterExpression)`: This is an iterator function that returns a table.
                *   `table`: `FactProductPerformance`. `FILTER` iterates over all rows of the `FactProductPerformance` table *that are visible in the current filter context*.
                *   `filterExpression`: `FactProductPerformance[Metric Type] = "Monthly Return"`. For each row of `FactProductPerformance` being iterated, this condition is evaluated.
            *   The `FILTER` function returns a new table containing only those rows from `FactProductPerformance` where the `Metric Type` is "Monthly Return". This resulting table is then used by `CALCULATE` to modify the filter context.
    *   **Difference from `'Total Cumulative Return'` filter:**
        *   `'Total Cumulative Return'` used a simple boolean filter: `FactProductPerformance[Metric Type] = "Cumulative Return"`. This is generally more efficient and idiomatic when filtering a single column based on a static value.
        *   `'Total Monthly Return'` uses an explicit `FILTER` function: `FILTER ( FactProductPerformance, FactProductPerformance[Metric Type] = "Monthly Return" )`. While it achieves a similar outcome in this specific case, `FILTER` is more versatile for complex conditions involving multiple columns or calculations within the filter expression. For this particular scenario, the simpler boolean filter `FactProductPerformance[Metric Type] = "Monthly Return"` would likely have been sufficient and potentially more performant.
    *   **`formatString`**: Same percentage formatting.

---

3.  **`'Total AUM'`**

    ```dax
    measure 'Total AUM' =
        CALCULATE (
            SUM ( FactProductPerformance[Value] ),
            FactProductPerformance[Metric Type] = "AUM"
        )
    formatString: \$#,0.00;(\$#,0.00);\$#,0.00
    ```

    *   **Purpose:** Calculates the sum of Assets Under Management (AUM) values.
    *   **Logic:** Identical to `'Total Cumulative Return'`, but the filter is `FactProductPerformance[Metric Type] = "AUM"`.
    *   **`formatString: \$#,0.00;(\$#,0.00);\$#,0.00`**:
        *   This is a currency format string.
        *   `\$`: Displays the dollar sign.
        *   `#,0`: Displays numbers with a thousands separator and at least one digit before the decimal.
        *   `.00`: Displays two decimal places.
        *   `(\$#,0.00)`: For negative numbers, encloses them in parentheses (a common accounting format).
    *   **`annotation PBI_FormatHint = {"currencyCulture":"en-US"}`**: Provides additional metadata to Power BI to help it understand the currency formatting, specifically that it's for US dollars.

---

4.  **`'Cumulative Return'`**

    ```dax
    measure 'Cumulative Return' =
        VAR MaxActualDateInVisual =
            MAX ( DimDate[Date] )
        VAR MaxSOMForIteration =
            DATE ( YEAR ( MaxActualDateInVisual ), MONTH ( MaxActualDateInVisual ), 1 )
        VAR RelevantStartOfMonthDates =
            FILTER (
                SUMMARIZE ( ALLSELECTED ( DimDate ), DimDate[StartOfMonthDate] ),
                DimDate[StartOfMonthDate] <= MaxSOMForIteration
            )
        RETURN
            IF (
                HASONEVALUE ( FactProductPerformance[Unified Product Name] ),
                CALCULATE (
                    PRODUCTX (
                        RelevantStartOfMonthDates,
                        VAR CurrentIterationSOMDate = DimDate[StartOfMonthDate]
                        RETURN
                            1
                                + CALCULATE (
                                    SUM ( FactProductPerformance[Value] ),
                                    FILTER ( ALL ( DimDate ), DimDate[StartOfMonthDate] = CurrentIterationSOMDate ),
                                    FactProductPerformance[Metric Type] = "Monthly Return"
                                )
                    ) - 1
                )
            )
    formatString: 0.00%;-0.00%;0.00%
    ```

    *   **Purpose:** This is a more complex measure designed to calculate a true compounded cumulative return *dynamically* based on monthly returns. Unlike `'Total Cumulative Return'` which sums pre-calculated values, this one calculates it from scratch using individual monthly returns. It calculates the cumulative return up to the latest date visible in the current filter context (e.g., in a visual or slicer).

    *   **Variables (`VAR`)**: Variables are used to store intermediate results, making the DAX more readable and potentially more efficient as a sub-expression is calculated only once.

        *   `VAR MaxActualDateInVisual = MAX ( DimDate[Date] )`
            *   `MAX ( DimDate[Date] )`: Finds the latest (maximum) date in the `DimDate[Date]` column *within the current filter context*. If a visual is showing data up to June 2023, this would be June 30, 2023 (or the latest date in June visible).
            *   **Purpose:** To determine the endpoint for the cumulative calculation.

        *   `VAR MaxSOMForIteration = DATE ( YEAR ( MaxActualDateInVisual ), MONTH ( MaxActualDateInVisual ), 1 )`
            *   `YEAR ( MaxActualDateInVisual )`: Extracts the year from `MaxActualDateInVisual`.
            *   `MONTH ( MaxActualDateInVisual )`: Extracts the month number from `MaxActualDateInVisual`.
            *   `DATE ( year, month, 1 )`: Constructs a date value representing the first day of the month of `MaxActualDateInVisual`.
            *   **Purpose:** To get a clean "end month" marker (as the start-of-month date) for the iteration.

        *   `VAR RelevantStartOfMonthDates = FILTER ( SUMMARIZE ( ALLSELECTED ( DimDate ), DimDate[StartOfMonthDate] ), DimDate[StartOfMonthDate] <= MaxSOMForIteration )`
            *   `ALLSELECTED ( DimDate )`: This function is crucial. It returns a table containing all rows of `DimDate`, but it *respects filters applied from outside the current visual* (e.g., slicers) while *ignoring filters applied from within the visual itself* (e.g., if this measure is in a matrix row/column context for specific dates).
            *   `SUMMARIZE ( ALLSELECTED ( DimDate ), DimDate[StartOfMonthDate] )`: Takes the `DimDate` table (as modified by `ALLSELECTED`) and creates a new table with a single column, `DimDate[StartOfMonthDate]`, containing all the unique start-of-month dates present.
            *   `FILTER ( ..., DimDate[StartOfMonthDate] <= MaxSOMForIteration )`: Filters this table of unique start-of-month dates, keeping only those that are less than or equal to `MaxSOMForIteration`.
            *   **Purpose:** To create a list (as a single-column table) of all start-of-month dates that should be included in the compounding calculation. This list will start from the earliest available start-of-month and go up to the `MaxSOMForIteration`.

    *   **`RETURN` Clause**:
        *   `IF ( HASONEVALUE ( FactProductPerformance[Unified Product Name] ), ... )`
            *   `HASONEVALUE ( FactProductPerformance[Unified Product Name] )`: Returns `TRUE` if the `Unified Product Name` column has exactly one distinct value in the current filter context.
            *   **Purpose:** This is a guard condition. Calculating a product-specific cumulative return usually only makes sense if a single product is in context. If multiple products (or no product) are selected, the `IF` condition is false, and the measure will return BLANK (as there's no `else` part).

        *   `CALCULATE ( PRODUCTX ( ... ) - 1 )` (This is executed if `HASONEVALUE` is TRUE)
            *   The outer `CALCULATE` here doesn't add explicit filters but ensures the `PRODUCTX` expression is evaluated within the established filter context (including the single product filter implied by `HASONEVALUE`).
            *   `PRODUCTX ( table, expression ) - 1`:
                *   `PRODUCTX` is an iterator function. It iterates over each row of the specified `table` and evaluates the `expression` for each row. Then, it multiplies all these results together.
                *   `table`: `RelevantStartOfMonthDates` (the list of start-of-month dates we prepared).
                *   `expression`:
                    ```dax
                    VAR CurrentIterationSOMDate = DimDate[StartOfMonthDate] // Captures the StartOfMonthDate for the current row of RelevantStartOfMonthDates
                    RETURN
                        1 + CALCULATE ( // This will be (1 + Monthly Return for CurrentIterationSOMDate)
                                SUM ( FactProductPerformance[Value] ),
                                FILTER ( ALL ( DimDate ), DimDate[StartOfMonthDate] = CurrentIterationSOMDate ), // Isolates calculation to the current month
                                FactProductPerformance[Metric Type] = "Monthly Return" // Ensures we get the monthly return
                            )
                    ```
                    *   `VAR CurrentIterationSOMDate = DimDate[StartOfMonthDate]`: In each iteration of `PRODUCTX`, this line captures the specific `StartOfMonthDate` from the current row of `RelevantStartOfMonthDates`.
                    *   `1 + CALCULATE(...)`: This calculates `(1 + monthly return)` for the `CurrentIterationSOMDate`. For example, if a monthly return is 0.05 (5%), this part becomes 1.05.
                    *   The inner `CALCULATE` calculates the monthly return:
                        *   `SUM ( FactProductPerformance[Value] )`: Sums the values.
                        *   `FILTER ( ALL ( DimDate ), DimDate[StartOfMonthDate] = CurrentIterationSOMDate )`: This is critical.
                            *   `ALL ( DimDate )`: Removes any existing filters on the `DimDate` table. This is necessary because `PRODUCTX` creates a row context for each `StartOfMonthDate`, but the inner `CALCULATE` needs to establish a *filter context* for that specific month across the entire `DimDate` to correctly sum returns related to it.
                            *   `DimDate[StartOfMonthDate] = CurrentIterationSOMDate`: It then applies a new filter to `DimDate`, keeping only rows where `StartOfMonthDate` matches the month `PRODUCTX` is currently processing.
                        *   `FactProductPerformance[Metric Type] = "Monthly Return"`: Filters the fact table to only consider monthly returns.
                *   **How `PRODUCTX` works for cumulative return:** If you have monthly returns R1, R2, R3, the cumulative return is `(1+R1)*(1+R2)*(1+R3) - 1`. `PRODUCTX` computes the `(1+R1)*(1+R2)*(1+R3)` part.
                *   `- 1`: After `PRODUCTX` has multiplied all the `(1 + monthly return)` factors, subtracting 1 converts the result from a growth factor back into a percentage return (e.g., 1.15 becomes 0.15).

    *   **`formatString`**: Same percentage formatting.

    *   **In essence, the `'Cumulative Return'` measure:**
        1.  Determines the latest month to consider based on the current visual context.
        2.  Gets a list of all start-of-month dates up to this latest month (respecting external slicers).
        3.  If only one product is selected:
            a.  For each month in the list, it calculates `(1 + that month's "Monthly Return" for the selected product)`.
            b.  It multiplies all these `(1 + monthly return)` values together.
            c.  Subtracts 1 from the final product.
        4.  If more than one product (or no product) is selected, it returns BLANK.

This `'Cumulative Return'` measure is a robust way to calculate true compounded returns and is a common pattern in financial Power BI models.
