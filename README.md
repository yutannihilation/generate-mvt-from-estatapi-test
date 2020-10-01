
<!-- README.md is generated from README.Rmd. Please edit that file -->

A test repo to generate MVT from estatapi
=========================================

出典
----

### データ

* 国土数値情報 行政区域データ（<https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-N03-v2_4.html>）
* e-Stat 作物統計調査 / 市町村別データ 平成30年産市町村別データ (<https://www.e-stat.go.jp/dbview?sid=0001803721>)

を加工して作成

### ベースの地図

<a href="https://maps.gsi.go.jp/vector/" target="_blank">地理院地図Vector（仮称）</a>提供のベクトルタイルを、<https://github.com/gsi-cyberjapan/gsivectortile-mapbox-gl-js>の`std.json`を使用して表示。

Get data
--------

### Polygon data

Download `N03-200101_GML.zip` from
<a href="https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-N03-v2_4.html" class="uri">https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-N03-v2_4.html</a>.

### Statistic data

    # Set API key of e-Stat API on ESTATAPI_KEY envvar
    d <- estatapi::estat_getStatsData(Sys.getenv("ESTATAPI_KEY"), "0001803721")
    arrow::write_parquet(d, "data_raw/0001803721.parquet")

Process data
------------

    library(dplyr, warn.conflicts = FALSE)

    adm <- kokudosuuchi::getKSJData("data_raw/N03-200101_GML.zip")
    #> 
    #> Details about this data can be found at http://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-N03-v2_3.html

    # We only need area_code to join
    g <- adm$`N03-20_200101` %>% 
      group_by(area_code = N03_007) %>% 
      summarise() %>% 
      mutate(area = sf::st_area(geometry))
    #> `summarise()` ungrouping output (override with `.groups` argument)
    #> Linking to GEOS 3.8.0, GDAL 3.0.4, PROJ 6.3.2
    g
    #> Simple feature collection with 1903 features and 2 fields
    #> geometry type:  GEOMETRY
    #> dimension:      XY
    #> bbox:           xmin: 122.9326 ymin: 20.42275 xmax: 153.9867 ymax: 45.55724
    #> geographic CRS: JGD2011
    #> # A tibble: 1,903 x 3
    #>    area_code                                                   geometry     area
    #>  * <chr>                                                  <POLYGON [°]>    [m^2]
    #>  1 01101     ((141.2574 42.99782, 141.2572 42.99781, 141.2569 42.99782…  464212…
    #>  2 01102     ((141.3333 43.07497, 141.3333 43.075, 141.3333 43.07505, …  635669…
    #>  3 01103     ((141.375 43.06851, 141.3737 43.06843, 141.3734 43.0684, …  569715…
    #>  4 01104     ((141.3664 43.05797, 141.3666 43.05823, 141.3667 43.05826…  344663…
    #>  5 01105     ((141.3637 42.94124, 141.3637 42.94138, 141.3637 42.94154…  462313…
    #>  6 01106     ((140.9982 42.98615, 140.9979 42.9867, 140.9977 42.98707,… 6574794…
    #>  7 01107     ((141.1681 43.08333, 141.1688 43.08299, 141.169 43.0829, …  751018…
    #>  8 01108     ((141.4475 43.02082, 141.4472 43.02094, 141.4465 43.02121…  243804…
    #>  9 01109     ((141.227 43.08333, 141.227 43.08302, 141.227 43.08281, 1…  567695…
    #> 10 01110     ((141.4136 42.89545, 141.4138 42.89511, 141.4141 42.89485…  598724…
    #> # … with 1,893 more rows

    d <- arrow::read_parquet("data_raw/0001803721.parquet")

    colnames(d) <- c(
      "category_code",
      "category_name",
      "plant_code",
      "plant_name",
      "area_code",
      "area_name",
      "unit",
      "value",
      "annotation"
    )

    d
    #> # A tibble: 51,570 x 9
    #>    category_code category_name plant_code plant_name area_code area_name unit 
    #>    <chr>         <chr>         <chr>      <chr>      <chr>     <chr>     <chr>
    #>  1 001           作付面積      001        春だいこん 01100     札幌市    ha   
    #>  2 001           作付面積      001        春だいこん 01202     函館市    ha   
    #>  3 001           作付面積      001        春だいこん 01203     小樽市    ha   
    #>  4 001           作付面積      001        春だいこん 01204     旭川市    ha   
    #>  5 001           作付面積      001        春だいこん 01205     室蘭市    ha   
    #>  6 001           作付面積      001        春だいこん 01206     釧路市    ha   
    #>  7 001           作付面積      001        春だいこん 01207     帯広市    ha   
    #>  8 001           作付面積      001        春だいこん 01208     北見市    ha   
    #>  9 001           作付面積      001        春だいこん 01209     夕張市    ha   
    #> 10 001           作付面積      001        春だいこん 01210     岩見沢市  ha   
    #> # … with 51,560 more rows, and 2 more variables: value <dbl>, annotation <chr>

    d <- d %>%
      filter(!is.na(value)) %>% 
      # convert ha to m^2
      mutate(value = if_else(unit == "ha", value * 10000, value))

    d_wide <- d %>%
      select(category_name, plant_code, area_code, value) %>% 
      tidyr::pivot_wider(names_from = category_name, values_from = value)
    d_wide
    #> # A tibble: 735 x 5
    #>    plant_code area_code 作付面積 収穫量 出荷量
    #>    <chr>      <chr>        <dbl>  <dbl>  <dbl>
    #>  1 001        01337      1050000   5250   5010
    #>  2 001        02207       500000   2600   2350
    #>  3 001        09208       160000    711    644
    #>  4 001        09216        60000    300    264
    #>  5 001        09364        30000    143    136
    #>  6 001        11201       210000   1040    970
    #>  7 001        11208       120000    583    480
    #>  8 001        11215       150000   1360   1190
    #>  9 001        11324       100000    484    460
    #> 10 001        12202      4440000  24400  23900
    #> # … with 725 more rows

TODO: There are some area codes that cannot be joined.

    d_distinct <- distinct(d, area_code, area_name)

    anti_join(d_distinct, g, by = "area_code")
    #> # A tibble: 5 x 2
    #>   area_code area_name
    #>   <chr>     <chr>    
    #> 1 15100     新潟市   
    #> 2 40130     福岡市   
    #> 3 12100     千葉市   
    #> 4 01100     札幌市   
    #> 5 22130     浜松市

    # This is because the geometry is more detailed.
    adm$`N03-20_200101` %>%
      filter(N03_003 == "新潟市")
    #> Simple feature collection with 34 features and 5 fields
    #> geometry type:  POLYGON
    #> dimension:      XY
    #> bbox:           xmin: 138.7842 ymin: 37.67897 xmax: 139.2668 ymax: 38.01986
    #> geographic CRS: JGD2011
    #> # A tibble: 34 x 6
    #>    N03_001 N03_002 N03_003 N03_004 N03_007                              geometry
    #>  * <chr>   <chr>   <chr>   <chr>   <chr>                           <POLYGON [°]>
    #>  1 新潟県  <NA>    新潟市  北区    15101   ((139.1476 37.91384, 139.1472 37.916…
    #>  2 新潟県  <NA>    新潟市  北区    15101   ((139.2323 37.96698, 139.2323 37.966…
    #>  3 新潟県  <NA>    新潟市  東区    15102   ((139.074 37.90595, 139.0739 37.9062…
    #>  4 新潟県  <NA>    新潟市  東区    15102   ((139.0694 37.95216, 139.0694 37.952…
    #>  5 新潟県  <NA>    新潟市  東区    15102   ((139.0697 37.95313, 139.0699 37.953…
    #>  6 新潟県  <NA>    新潟市  東区    15102   ((139.0698 37.95368, 139.0701 37.954…
    #>  7 新潟県  <NA>    新潟市  東区    15102   ((139.0698 37.95341, 139.0698 37.953…
    #>  8 新潟県  <NA>    新潟市  中央区  15103   ((139.0011 37.90833, 139.0011 37.908…
    #>  9 新潟県  <NA>    新潟市  中央区  15103   ((139.0633 37.946, 139.0633 37.94599…
    #> 10 新潟県  <NA>    新潟市  中央区  15103   ((139.0635 37.94605, 139.0635 37.946…
    #> # … with 24 more rows

    d_joined <- d_wide %>%
      inner_join(g, ., by = "area_code") %>% 
      mutate(
        harvest_per_planted_area = 収穫量 / 作付面積,
        across(作付面積:出荷量, ~ . / as.numeric(area))
      )
    d_joined
    #> Simple feature collection with 723 features and 7 fields
    #> geometry type:  GEOMETRY
    #> dimension:      XY
    #> bbox:           xmin: 127.533 ymin: 26.07447 xmax: 145.3376 ymax: 45.52647
    #> geographic CRS: JGD2011
    #> # A tibble: 723 x 8
    #>    area_code                  geometry     area plant_code 作付面積  収穫量
    #>  * <chr>                <GEOMETRY [°]>    [m^2] <chr>         <dbl>   <dbl>
    #>  1 01202     MULTIPOLYGON (((140.9622… 6778724… 002         4.28e-4 1.62e-6
    #>  2 01202     MULTIPOLYGON (((140.9622… 6778724… 003         5.46e-4 2.08e-6
    #>  3 01202     MULTIPOLYGON (((140.9622… 6778724… 005         8.26e-4 2.32e-6
    #>  4 01202     MULTIPOLYGON (((140.9622… 6778724… 007         5.90e-3 1.55e-5
    #>  5 01202     MULTIPOLYGON (((140.9622… 6778724… 008         5.90e-3 1.55e-5
    #>  6 01203     MULTIPOLYGON (((141.1307… 2438273… 007         5.74e-4 1.21e-6
    #>  7 01203     MULTIPOLYGON (((141.1307… 2438273… 008         5.74e-4 1.21e-6
    #>  8 01204     POLYGON ((142.2878 43.57… 7476627… 002         4.01e-5 8.03e-8
    #>  9 01204     POLYGON ((142.2878 43.57… 7476627… 007         2.15e-3 5.63e-6
    #> 10 01204     POLYGON ((142.2878 43.57… 7476627… 008         2.15e-3 5.63e-6
    #> # … with 713 more rows, and 2 more variables: 出荷量 <dbl>,
    #> #   harvest_per_planted_area <dbl>

    l <- d_joined %>% 
      split(.$plant_code)

    unlink("data", recursive = TRUE)
    dir.create("data")

    purrr::iwalk(l, function(data, nm) {
      data <- data %>% 
        select(-plant_code, -area_code)
      
      write_sf(
        data,
        dsn = file.path("data", nm), 
        layer = nm,
        driver = "MVT",
        dataset_options = c(
          "MINZOOM=4",
          "MAXZOOM=9",
          "TILE_EXTENSION=mvt",
          "COMPRESS=NO"
        )
      )
    })
