# Power BI Financial Performance Explorer 🚀

Hey there! Welcome to this Power BI project where we dive into the world of investment performance. We're looking at two imaginary investment firms, Firm A and Firm B, and trying to make sense of how their different products (some in stocks/Equity, some in bonds/Fixed Income) have done over time.

The main goal? To build some cool, interactive dashboards in Power BI that tell a clear story about:
1.  **Cumulative Returns:** How has an initial investment in each product grown (or shrunk!) over the months and years?
2.  **Assets Under Management (AUM):** How much money are these products managing, and how has that changed?

We want to be able to slice and dice this information easily, so filters are a big part of this.

## The Raw Material: Our Data

Our adventure starts with a CSV file: `Sample Time Series Data.csv`.

**What's inside this data file?**
*   It's basically a monthly diary of financial numbers.
*   We've got entries for "Month," "Year," and the specific "Date" (always the 1st of the month, it seems).
*   We know which "Firm" (A or B) and which "Product Code" (like AE1, A1, BE2, etc.) each number belongs to.
*   Each product also has a "Strategy" – either "Equity" or "Fixed Income."
*   And the numbers themselves? They represent:
    *   **Monthly Return %:** For some products, how much they went up or down each month.
    *   **AUM $:** For other products, the total amount of money they were managing.

**The Catch:** The CSV is a bit messy to start with – it's got headers spread over multiple rows and data laid out wide. So, a good chunk of our initial work in Power BI involves some data janitor duty in Power Query to get it into a nice, usable shape.

## Our Mission: The Dashboards

We're building two main dashboards:

**Dashboard 1: The Cumulative Return Story**
*   **What we want to see:** A timeline showing how the cumulative return for each product has evolved.
*   **Making it interactive:** We need filters so we can easily look at:
    *   Just Firm A's products.
    *   All Equity products from both Firm A and Firm B.
    *   Only the Equity products from Firm B.
*   **Special Handling:** For "Firm A - Equity" (that's Product A1), we'll use the cumulative return numbers straight from the data file. For everything else, we'll have to calculate the cumulative return ourselves based on their monthly ups and downs.

**Dashboard 2: Tracking the AUM Journey**
*   **What we want to see:** A timeline showing the AUM for each product.
*   **Visual options:** Maybe a stacked bar chart to see how AUM is composed within each firm, or side-by-side bars/lines for direct comparisons.
*   **Making it interactive:** Similar filters as the first dashboard:
    *   Firm A's products.
    *   Equity products from both firms.
    *   Firm B's Equity products only.

## How We're Building This in Power BI

Here's a peek under the hood at our Power BI Desktop process:

**1. Wrestling the Data into Shape (Power Query Magic):**
    *   First things first, we grab that CSV.
    *   **Header Gymnastics:** Those tricky multi-row headers? We use Power Query's transpose, fill-down, and merge features to sort them into proper column names.
    *   **Going Long, Not Wide:** We "unpivot" the data. This means instead of having lots of columns for each product/metric, we make the table longer with dedicated columns like `Firm`, `Product Code`, `Metric Type`, and `Value`. It's much easier to work with this way.
    *   **Cleaning Up:**
        *   AUM values come with `$` and commas – we strip those out.
        *   Return values have `%` signs – those go too.
    *   **Getting the Numbers Right:**
        *   Returns become actual decimal numbers (so 5% turns into 0.05).
        *   AUM becomes a plain number.
        *   Dates become proper `Date` types.
    *   **Helpful New Columns:**
        *   `Unified Product Name`: We create friendlier names like "Firm A - Equity" to make legends and slicers easier to read.
        *   `Product Category`: A simple "Equity" or "Fixed Income" tag.

**2. Building the Data Model (The Blueprint):**
    *   **The Main Table:** We call our cleaned-up data `FactProductPerformance`.
    *   **The Calendar:** We create a `DimDate` table. This is super important for anything time-related in Power BI.
        *   It has a unique `Date` for every single day we care about.
        *   It's packed with useful stuff like Year, Month, Month Name, Quarter, Start of Month, etc.
    *   **Connecting Them:** We draw a line (a relationship) from `DimDate[Date]` to `FactProductPerformance[Date]`. It's a "one-to-many" link: one day in our calendar can have many performance records (for different products/metrics).

**3. Writing the Brains (DAX Measures):**
    *   **The Basics:**
        *   `Monthly Return Value`: Grabs the monthly return number.
        *   `Total AUM`: Adds up the AUM.
        *   `Calculated Cumulative Return from Monthly`: This is where the magic happens for most products. It takes their monthly returns and compounds them day by day (well, month by month in our case) to figure out the cumulative return over time using `PRODUCTX`.

**4. Making it Look Good (Reports & Visuals):**
    *   **Dashboard 1 (Cumulative Return):**
        *   Usually a **Line Chart** (good for trends) or a **Clustered Column Chart** (good for comparing side-by-side).
        *   Dates go on the bottom (X-axis).
        *   Our `[Cumulative Return (Display)]` measure goes on the side (Y-axis).
        *   Each `Unified Product Name` gets its own line/column.
    *   **Dashboard 2 (AUM):**
        *   Could be a **Stacked Bar Chart** (shows how AUM is split among products each month) or a **Clustered Column/Line Chart**.
        *   Dates on the bottom.
        *   `[Total AUM]` on the side.
        *   Each `Unified Product Name` is a segment or its own column/line.
    *   **The Controls (Slicers):**
        *   Dropdown for `Product Category`.
        *   A neat hierarchy slicer for Firm > Category > Product.
        *   Checkboxes for `Year`.

**5. Checking Our Work (DAX Query View):**
    *   Before we built all the fancy visuals, we used the DAX Query View in Power BI a LOT. It's like a scratchpad for running little data experiments. We used it to look at slices of data, test if our filters were working, and make sure our basic calculations made sense. Super handy!

## What We Learned (And a Few Head-Scratchers)

*   Dealing with messy CSV headers is a rite of passage in Power Query. Transpose is your friend!
*   DAX filter context is king. Understanding how filters flow (or don't flow) is key to getting calculations right, especially for running totals like cumulative return.
*   Making DAX run fast, especially with iterative functions like `PRODUCTX`, sometimes needs a bit of thought and optimization.
*   The DAX Query View is a lifesaver for poking around in your data and testing ideas before you commit to a complex measure.
