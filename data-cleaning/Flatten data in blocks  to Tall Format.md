# Flatten matrixed block data to Tall Format

**Purpose**: Transform a Excel matrix data containing blocks, groupings, and metric values into a flat, analysis-ready long table.

---

## Use Case

Often, data has to be extracted from existing formats including reports formatted for visual appeal instead of ease of data extraction. Below is one such example.

![Initial data structure](../images/library_transform_start.png)

The source Excel file contains multiple "Areas" (e.g., Area 1, Area 2, etc.), each with several "Branch" blocks. Each Branch includes multiple rows of metrics across ten columns. However, the format is grouped and not suitable for analysis or visualization in tools like Power BI or Excel PivotTables.

This script:
- Extracts area and Branch labels
- Flattens the matrix structure into row-based long format
- Outputs a clean table with Area, Branch, Metric, Column (as Attribute), and Value

Tall tables are preferred in Power BI as they are easier to model, scale better, and work well with Power BI’s features like slicers, visual interactions, and row-level filters. Also, tall/long tables support normalized data model (star schema)/ The code follows the below screenshot.

![Final table preview](../images/library_transform_result.png)

---

## M Code

```m
let
    // Step 1: Load source Excel file (change path if needed)
    Source = Excel.Workbook(File.Contents("D:\divya\Documents\Sample.xlsx"), null, true),

    // Step 2: Get the table from the sheet named "Sheet1"
    Sheet1_Sheet = Source{[Item="Sheet1", Kind="Sheet"]}[Data],

    // Step 3: Fill down 'Column1' to propagate Area names (they appear only once above each block)
    FillArea = Table.FillDown(Sheet1_Sheet, {"Column1"}),

    // Step 4: Create a new column "Area" — keep only rows that begin with the word "AREA"
    AddArea = Table.AddColumn(FillArea, "Area", each if Text.StartsWith(Text.Upper([Column1]), "AREA") then [Column1] else null),

    // Step 5: Fill down the Area names to all relevant rows below each "AREA"
    FillAreaDown = Table.FillDown(AddArea, {"Area"}),

    // Step 6: Remove the rows that *only* define the "Area" (we already propagated their value)
    RemoveAreaRows = Table.SelectRows(FillAreaDown, each not Text.StartsWith(Text.Upper([Column1]), "AREA")),

    // Step 7: Mark rows that define "Branch" blocks (these will become our "Branch" labels later)
    MarkBranchRows = Table.AddColumn(RemoveAreaRows, "BranchMarker", each if Text.Contains([Column1], "Branch") then [Column1] else null),

    // Step 8: Fill down Branch names to associate them with their respective data blocks
    FillBranchDown = Table.FillDown(MarkBranchRows, {"BranchMarker"}),

    // Step 9: Promote first actual data row as header
    PromoteHeaders = Table.PromoteHeaders(FillBranchDown, [PromoteAllScalars=true]),

    // Step 10: Change column types — use type any for flexibility during transformation
    ChangedTypes = Table.TransformColumnTypes(PromoteHeaders, {
        {"Branch 1", type text},
        {"Column1", type any}, {"Column2", type any}, {"Column3", type any},
        {"Column4", type any}, {"Column5", type any}, {"Column6", type any},
        {"Column7", type any}, {"Column8", type any}, {"Column9", type any}, {"Column10", type any},
        {"Area 1", type text}, {"Branch 1_1", type text}
    }),

    // Step 11: Remove rows that are still Branch labels (we already filled them down)
    RemoveBranchLabelRows = Table.SelectRows(ChangedTypes, each not Text.Contains([Branch 1], "Branch")),

    // Step 12: Unpivot all data columns to turn wide table into tall format
    Unpivoted = Table.UnpivotOtherColumns(RemoveBranchLabelRows, {"Branch 1", "Area 1", "Branch 1_1"}, "Attribute", "Value"),

    // Step 13: Rename columns for final output
    RenamedColumns = Table.RenameColumns(Unpivoted, {
        {"Branch 1", "Metric"},
        {"Area 1", "Area"},
        {"Branch 1_1", "Branch"}
    }),

    // Step 14: Keep only relevant columns in final output
    Final = Table.SelectColumns(RenamedColumns, {"Area", "Branch", "Metric", "Attribute", "Value"})
in
    Final
