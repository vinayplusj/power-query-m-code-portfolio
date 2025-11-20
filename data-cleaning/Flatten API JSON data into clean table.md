# Flatten and Enrich Earthquake Data from USGS GeoJSON Feed

**Purpose**: Data is often provided as JSON through APIs because that combination makes it easy for many different systems and tools to talk to each other in a safe and flexible way. 
JSON supports nested data. APIs keep one central source of truth and support automated data refresh.
Connect to the public GeoJSON API from the United States Geological Survey (USGS) and transform nested earthquake data into a clean, long-format, analysis-ready table using Power Query M. 
This code connects to the [USGS FDSN Event API](https://earthquake.usgs.gov/fdsnws/event/1/) to retrieve and flatten a full year of global earthquake data. 
The script template dynamically adjusts to the previous calendar year, supports pagination (up to 60,000 records), and outputs a flat table with enriched time and location fields.

---

## Use Case

The USGS provides a live feed of earthquake events for the past 30 days in GeoJSON format. Each record includes:
- Magnitude
- Time
- Coordinates
- Location description

However, to build trend dashboards, maps, or severity reports in tools like Power BI or Tableau, one must:

- Connect to and parse a **GeoJSON API**
- Extract nested objects and lists `features`, `properties`, and `geometry`
- Convert epoch timestamps into readable datetime
- Handle paging (limit + offset)
- Normalize coordinates and data types
- Adding **Day of Week** and **Hour of Day**
- Extracting a **Region Keyword** (e.g., country/state/zone) from the `Location` field
- Flattening nested fields and arrays

---

## 🔗 API Endpoint

**Base URL**:  
`https://earthquake.usgs.gov/fdsnws/event/1/query?format=geojson`

**Dynamic Parameters Used**:
- `starttime` = first day of previous year
- `endtime` = last day of previous year
- `limit` = 20,000
- `offset` = 1, 20001, 40001, etc.

---

## 🔁 Key Transformations

- Dynamic date generation using `DateTime.LocalNow()`
- Pagination using `List.Generate` and `List.Transform`
- Flattening nested structures (`properties`, `geometry`)
- Epoch-to-datetime conversion
- Extraction of:
  - Latitude, Longitude, Depth
  - Day of Week
  - Hour of Day
  - Region (parsed from Location)
- Type conversion for all key fields

---

## 🧪 Output Example

| TimeUTC            | DayOfWeek | HourOfDay | Magnitude | Region              | Depth (km) | Latitude | Longitude |
|--------------------|-----------|-----------|-----------|----------------------|------------|----------|-----------|
| 2024-01-13 09:42:00| Saturday  | 9         | 5.1       | Papua New Guinea     | 10.0       | -6.29    | 147.48    |
| 2024-03-22 14:30:00| Friday    | 14        | 4.3       | Alaska               | 8.2        | 60.1     | -149.2    |

---

## ✅ M Code

```m
let
    Today = DateTime.Date(DateTime.LocalNow()),
    LastYear = Date.Year(Today) - 1,
    StartDate = Date.ToText(#date(LastYear, 1, 1), "yyyy-MM-dd"),
    EndDate = Date.ToText(#date(LastYear, 12, 31), "yyyy-MM-dd"),

    GetPage = (offset as number) =>
        let
            url = "https://earthquake.usgs.gov/fdsnws/event/1/query?" &
                  "format=geojson" &
                  "&starttime=" & StartDate &
                  "&endtime=" & EndDate &
                  "&limit=20000" &
                  "&offset=" & Number.ToText(offset),
            raw = Json.Document(Web.Contents(url)),
            features = raw[features]
        in
            features,

    Offsets = List.Generate(() => 1, each _ <= 60001, each _ + 20000),
    AllFeatures = List.Combine(List.Transform(Offsets, each GetPage(_))),
    FeatureTable = Table.FromList(AllFeatures, each { _ }, {"Record"}),

    ExpandRecord = Table.ExpandRecordColumn(FeatureTable, "Record", {"properties", "geometry", "id"}),
    ExpandProps = Table.ExpandRecordColumn(ExpandRecord, "properties", {
        "mag", "place", "time", "updated", "tsunami", "type", "magType", "felt", "url"
    }),
    ExpandGeometry = Table.ExpandRecordColumn(ExpandProps, "geometry", {"coordinates"}),

    AddLongitude = Table.AddColumn(ExpandGeometry, "Longitude", each [coordinates]{0}),
    AddLatitude = Table.AddColumn(AddLongitude, "Latitude", each [coordinates]{1}),
    AddDepth = Table.AddColumn(AddLatitude, "Depth (km)", each [coordinates]{2}),
    RemoveCoordList = Table.RemoveColumns(AddDepth, {"coordinates"}),

    ConvertTimestamps = Table.TransformColumns(RemoveCoordList, {
        {"time", each #datetime(1970,1,1,0,0,0) + #duration(0,0,0, Number.From(_) / 1000), type datetime},
        {"updated", each #datetime(1970,1,1,0,0,0) + #duration(0,0,0, Number.From(_) / 1000), type datetime}
    }),

    Renamed = Table.RenameColumns(ConvertTimestamps, {
        {"time", "TimeUTC"},
        {"place", "Location"},
        {"mag", "Magnitude"}
    }),

    AddDayOfWeek = Table.AddColumn(Renamed, "DayOfWeek", each Date.DayOfWeekName(Date.From([TimeUTC])), type text),
    AddHourOfDay = Table.AddColumn(AddDayOfWeek, "HourOfDay", each Time.Hour(Time.From([TimeUTC])), type number),

    AddRegion = Table.AddColumn(AddHourOfDay, "Region", each 
        let
            loc = [Location],
            parts = Text.Split(loc, " of "),
            region = if List.Count(parts) > 1 then List.Last(parts) else loc,
            cleaned = Text.Trim(region)
        in
            if cleaned = "" or cleaned = null then loc else cleaned,
        type text
    ),

    SetTypes = Table.TransformColumnTypes(AddRegion, {
        {"TimeUTC", type datetime},
        {"Magnitude", type number},
        {"Location", type text},
        {"Region", type text},
        {"Latitude", type number},
        {"Longitude", type number},
        {"Depth (km)", type number},
        {"magType", type text},
        {"type", type text},
        {"tsunami", type number},
        {"felt", type number},
        {"url", type text},
        {"DayOfWeek", type text},
        {"HourOfDay", type number}
    }),

    Final = Table.SelectColumns(SetTypes, {
        "TimeUTC", "DayOfWeek", "HourOfDay",
        "Magnitude", "Location", "Region",
        "Latitude", "Longitude", "Depth (km)",
        "magType", "type", "tsunami", "felt", "url"
    })
in
    Final
