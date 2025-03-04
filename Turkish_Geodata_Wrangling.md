Data Wrangling In Turkish: More than Meets the Dotted İ
================

## Introduction

In November 2023, the Turkish Presidency for Migration Management (PMM)
announced that more than 1,100 neighborhoods across the country would be
closed to residence permit registration/renewal for foreign nationals,
along with an Excel spreadsheet of affected neighborhoods. Wanting to
get a better understanding of the geographic concentration and breadth
of this policy, I knew that a map visualizing those closed neighborhoods
would offer a much more accessible look at the data.

This meant: 1. Extracting a list of affected neighborhoods from the
source data from the PMM 2. Finding a suitable data set which includes
neighborhood names for all of Turkey and their respective geometries (in
other words, their geographical boundaries) 3. Matching the PMM-derived
list of neighborhoods with those in the geographical data set

This RMarkdown notebook demonstrates the approach I took to solving this
problem, using fuzzy matching to produce a mappable `.geojson` file. For
a write-up of the overall process and final map, please take a look at
[my blog post](https://www.kevin-cole.com/blog/data-wrangling/).

## Loading Data

We start with the following source documents: 1. `Ilceler.csv`, a list
of all of Turkey’s subprovincial towns (İlçeler) and their IDs. 2.
`Mahalleler.csv`, a list of all of Turkey’s neighborhoods (Mahalleler)
and their IDs. 3. `Kapali-Mahalleler.xlsx`, an Excel spreadsheet from
the PMM listing neighborhoods which are affected by the regulations.

``` r
# Function to read a CSV file with necessary parameters
read_csv_file <- function(file_path) {
    read.csv(
        file = file_path,
        header = TRUE,
        sep = ",",
        row.names = NULL,
        stringsAsFactors = FALSE,
        encoding = "UTF-8"
    )
}

# Load data
districts <- read_csv_file("Ilceler.csv")
neighborhoods <- read_csv_file("Mahalleler.csv")

# Define the file paths
closed_neighborhoods_file <- "Kapali-Mahalleler.xlsx"
closed_neighborhoods_csv <- "Kapali-Mahalleler.csv"

# Download the Excel file if it does not exist
if (!file.exists(closed_neighborhoods_file)) {
    download.file(
        "https://www.goc.gov.tr/kurumlar/goc.gov.tr/DUYURULAR/2022/06-Haziran/UK/Kapali-Mahalleler.xlsx",
        closed_neighborhoods_file,
        mode = "wb"
    )
}

# Read the Excel file and write to a CSV file if it does not exist
if (!file.exists(closed_neighborhoods_csv)) {
    closed_neighborhoods_data <- read.xlsx(
        closed_neighborhoods_file,
        sheetIndex = 1,
        encoding = "UTF-8"
    )
    write.csv2(closed_neighborhoods_data, closed_neighborhoods_csv, row.names = FALSE, fileEncoding = "UTF-8")
}

# Read the CSV file ensuring correct encoding
closed_neighborhoods_data <- read.csv2(closed_neighborhoods_csv, fileEncoding = "UTF-8")
```

## Cleaning Data

After successfully loading the data, we’ll perform some clean up. In
particular, we’ll use the `stri_trans_general()` function from the
stringi package to convert Turkish characters into simplified Latin
characters. For example, “Çanakkale” will be transformed into
“Canakkale”. This will make it easier to match neighborhood, town and
province names across these datasets and avoid errors attributable to
typographic errors such as typing the Turkish dotted İ instead of the
undotted I.

We’ll also standardize abbreviations like “Mh.” for “Mahalle”
(neighborhood).

``` r
# Clean districts data
districts <- districts %>%
    mutate(
        town_clean = stringi::stri_trans_general(name, "Latin-ASCII"),
        city_id = cityId,
        town_id = townId,
        district_id = districtId,
        quarter_id = quarterId
    )

# Clean neighborhoods data
neighborhoods <- neighborhoods %>%
    mutate(
        name = gsub("Mah\\.", "Mh.", name),
        name = toupper(name),
        neighborhood_clean = toupper(stringi::stri_trans_general(name, "Latin-ASCII")),
        city_id = cityId,
        town_id = townId,
        quarter_id = quarterId
    )

# Clean data from closed neighborhoods file
closed_neighborhoods_data <- closed_neighborhoods_data %>%
    mutate(
        neighborhood = gsub("MAHALLESİ", "MH.", MAHALLE),
        subdivisions = ifelse(grepl(",", neighborhood), gsub(".*,\\s", "", neighborhood), NA),
        neighborhood = gsub(",.*", "", neighborhood),
        district = gsub("\\/.*", "", ILCE),
        town = district,
        town_clean = stri_trans_general(district, "Latin-ASCII"),
        province_clean = stri_trans_general(IL, "Latin-ASCII")
    )

# Preview the transformed data
head(closed_neighborhoods_data)
```

    ##   Sıra.No    IL     ILCE              MAHALLE NA.   neighborhood subdivisions
    ## 1       1 Adana   Ceyhan       BOTA MAHALLESİ  NA       BOTA MH.         <NA>
    ## 2       2 Adana Çukurova    MEMİŞLİ MAHALLESİ  NA    MEMİŞLİ MH.         <NA>
    ## 3       3 Adana Çukurova      FADIL MAHALLESİ  NA      FADIL MH.         <NA>
    ## 4       4 Adana Çukurova   BOZCALAR MAHALLESİ  NA   BOZCALAR MH.         <NA>
    ## 5       5 Adana Çukurova      ÖRCÜN MAHALLESİ  NA      ÖRCÜN MH.         <NA>
    ## 6       6 Adana Çukurova KÜÇÜKÇINAR MAHALLESİ  NA KÜÇÜKÇINAR MH.         <NA>
    ##   district     town town_clean province_clean
    ## 1   Ceyhan   Ceyhan     Ceyhan          Adana
    ## 2 Çukurova Çukurova   Cukurova          Adana
    ## 3 Çukurova Çukurova   Cukurova          Adana
    ## 4 Çukurova Çukurova   Cukurova          Adana
    ## 5 Çukurova Çukurova   Cukurova          Adana
    ## 6 Çukurova Çukurova   Cukurova          Adana

## Matching Turkish Provinces with their ISO Code

To make this dataset more portable for potential future use cases, we’ll
go ahead and match provinces with their corresponding ISO code (e.g.,
Adana is “TR-01”).

``` r
iso_codes <- ISO_3166_2
# Function to match names to ISO codes
name_to_iso_code <- function(name) {
    match <- iso_codes[stringi::stri_trans_general(iso_codes$Name, "Latin-ASCII") == name, ]
    if (nrow(match) == 0) {
        return("No Match Found for this Name")
    }
    return(as.integer(gsub("TR\\-", "", match$Code)))
}

closed_neighborhoods_data$city_id <- sapply(closed_neighborhoods_data$province_clean, FUN = name_to_iso_code)
```

## Matching Towns Across Datasets

With initital data loading and transformation steps completed, we can
now move on to matching up the town names listed in the PMM’s Excel
spreadsheet with their corresponding entries in the source geographic
dataset. This will enable us to map the boundaries of the affected
neighborhoods. However, we can’t rely completely on 1:1 fidelity matches
between these datasets, due to both typographic errors and irregular
spelling/abbreviation conventions. Instead, we’ll use the Jarowinkler
Distance Algorithm to calculate the distance between PMM’s town names
and candidates in the geographic datasets. “Distance” is calculated by
counting the number of character transformations necessary to transform
a source string into the target string.

``` r
# Function to find town match
find_town_match <- function(city_town, districts, threshold = 0.90, verbose = FALSE) {
    town_clean <- gsub("\\d*\\s\\-\\s", "", city_town)
    city_id <- as.integer(gsub("\\s\\-\\s.*", "", city_town))

    districts_subset <- districts %>% filter(city_id == city_id)
    scores <- RecordLinkage::jarowinkler(town_clean, districts_subset$town_clean)
    max_score <- max(scores)
    if (max_score >= threshold) {
        if (verbose) {
            print(paste(districts_subset$town_clean[which.max(scores)], max_score))
        }
        return(as.integer(districts_subset$id[which.max(scores)]))
    }
    return(NA)
}

closed_neighborhoods_data$city_town <- paste(closed_neighborhoods_data$city_id, closed_neighborhoods_data$town_clean, sep = " - ")
closed_neighborhoods_data$town_id <- sapply(closed_neighborhoods_data$city_town,
    FUN = find_town_match, districts = districts
)
```

It’s worth noting that the above approach is not optimized for
performance on particularly large datasets, because we rely on running
the Jarowinkler algorithm on each candidate string and then selecting
the highest candidate. However, for the size of dataset we’re working
with, this computation can still be completed in just a few seconds.

## Matching Neighborhoods Across Datasets

After finding affected towns, we move down to the higher granularity
level of individual neighborhoods within these towns.

``` r
# Filter affected neighborhoods
affected_neighborhoods <- neighborhoods %>%
    filter(town_id %in% closed_neighborhoods_data$town_id)

# Function to find neighborhood match
find_match <- function(district_neighborhood, affected_neighborhoods,
                       threshold = 0.90, verbose = FALSE) {
    neighborhood_name <- gsub("\\d*\\s\\-\\s", "", district_neighborhood)
    district <- as.integer(gsub("\\s\\-\\s.*", "", district_neighborhood))

    top_score <- 0
    top_candidate <- NA

    for (j in seq_len(nrow(affected_neighborhoods))) {
        jw_score <- RecordLinkage::jarowinkler(neighborhood_name, affected_neighborhoods$neighborhood_clean[j])
        if (district == affected_neighborhoods$town_id[j] && jw_score >= threshold) {
            if (verbose) {
                print(paste(affected_neighborhoods$neighborhood_clean[j], jw_score))
            }
            if (jw_score > top_score) {
                top_score <- jw_score
                top_candidate <- affected_neighborhoods$id[j]
            }
        }
    }
    return(top_candidate)
}

closed_neighborhoods_data$district_neighborhood <- paste(closed_neighborhoods_data$town_id,
    toupper(stringi::stri_trans_general(closed_neighborhoods_data$neighborhood, "Latin-ASCII")),
    sep = " - "
)
closed_neighborhoods_data$neighborhood_id <- sapply(closed_neighborhoods_data$district_neighborhood,
    FUN = find_match, affected_neighborhoods = affected_neighborhoods, threshold = 0.93
)

# Rerun match function with lower threshold for neighborhoods still missing a match
missing <- closed_neighborhoods_data %>% filter(is.na(neighborhood_id))
missing$neighborhood_id <- sapply(missing$district_neighborhood, FUN = find_match, affected_neighborhoods = affected_neighborhoods, threshold = 0.85)
closed_neighborhoods_data <- closed_neighborhoods_data %>% filter(!is.na(neighborhood_id))
export_data <- bind_rows(closed_neighborhoods_data, missing)
```

## Final Adjustments and Export

Before finishing, there are a few manual adjustments that have to be
made due to incomplete and/or out of date information in the source
geospatial dataset.

``` r
# Select and convert columns
export_data <- export_data %>%
    select(city_id, town_id, city_town, district_neighborhood, neighborhood_id, subdivisions) %>%
    mutate(
        city_id = as.integer(city_id),
        town_id = as.integer(town_id),
        neighborhood_id = as.integer(neighborhood_id)
    )

# Manual fixes
export_data$neighborhood_id[export_data$district_neighborhood == "605 - B.HUSEYINBEY MH."] <- 30796 # Abbreviated in PMM Data
export_data$neighborhood_id[export_data$district_neighborhood == "799 - TURKMENLER MH."] <- 42029 # Renamed in 2021 from 'Evrenpasa Mh.'
export_data$neighborhood_id[export_data$district_neighborhood == "838 - MENDE KOYU"] <- 45004 # Renamed from 'Kasbelen'
export_data$neighborhood_id[export_data$district_neighborhood == "940 - TOSYALI MH."] <- 50449 # Renamed in 2021 to 'Izzettin Iyigun Pasa Mh.'

# Export to CSV
write.csv2(export_data, "Exported_Data.csv")
```

Finally, we export the resulting data to a CSV file. This file contains
the IDs of each affected neighborhood, which we can use as a filter in
geospatial software like QGIS in order to produce a new `.geojson` file
which only contains the polygons corresponding to those neighborhoods
affected by the closure policy.
