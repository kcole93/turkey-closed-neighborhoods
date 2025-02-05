# turkey-closed-neighborhoods
GeoJSON Data of Neighborhoods Closed to Registration by Foreign Nationals in Turkey

This repository contains a GeoJSON file of the neighborhoods which were [announced](https://www.goc.gov.tr/mahalle-kapatma-duyurusu-hk2) by the Turkish Presidency of Migration Management as being closed to registration by foreign nationals as of June 30th, 2022.


## Notes and Caveats
The list of affected neighborhoods published by the Presidency of Migration Management contains a total of 1,169 rows of neighborhoods and in this source data, there are a number of duplicate entries. 

Fuzzy matching was used to match this list to an existing data set representing Turkish neighborhood spatial data. A total of 1,152 unique neighborhoods were identified through this approach, while the PMM's officially communicated count of affected neighborhoods is 1,160.

An RMarkdown notebook detailing the steps used to produce the final GeoJSON file can be found [here](/Turkish_Geodata_Wrangling.rmd).

### Renamed & Missing Neighborhoods
- **Mehmet Abdi Bulut Mahallesi** in Kilis is missing from the mapped data. The neighborhood was created in 2021 and is not yet reflected in the neighborhood spatial data set.
- **Türkmenler Mahallesi** is the name given in 2021 to the neighborhood formerly known as Evrenpaşa Mahallesi in Tokat/Yeşilyurt. The town's name has been updated in the map, but was not yet updated in the spatial data set.
- **Tosyalı Mahallesi** is the name given to the neighborhood formerly known as İzzettin İyigün Paşa Mahallesi in Kilis. The town's name has been updated in the map, but was not yet updated in the spatial data set.
- **Mende Köyü** is the name given to the neighborhood formerly known as Kaşbelen in Uşak. The town's name has been updated in the map, but was not yet updated in the spatial data set.

## Acknowledgements
The raw neighborhood data used to map this list was adapted from [Çağrı Özkurt](https://github.com/cagriozkurt)'s scraped collection, available [on Github](https://github.com/cagriozkurt/TurkeyGeoJSON).
