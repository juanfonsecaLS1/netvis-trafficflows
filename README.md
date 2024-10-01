# Visualisation of Traffic Flows
Juan P. Fonseca-Zamora

*This is a quick demo on visualising traffic volumes on the road network
that was prepared for the [Network Visualisation
Hackathon](https://github.com/Robinlovelace/netvishack). The aim is to
visualise the typical daily flows of an undirected network.*

``` r
options(repos = c(CRAN = "https://cloud.r-project.org"))
if (!require("remotes")) install.packages("remotes")
pkgs = c(
    "sf",
    "tidyverse",
    "tmap",
    "zonebuilder",
    "BAMMtools"
)
remotes::install_cran(pkgs)
sapply(pkgs, require, character.only = TRUE)
```

             sf   tidyverse        tmap zonebuilder   BAMMtools 
           TRUE        TRUE        TRUE        TRUE        TRUE 

## Loading the data

We will be using some traffic estimates for Edinburgh that where
produced with a GLM. The spatial data correspond to [OS Open
Roads](https://www.ordnancesurvey.co.uk/products/os-open-roads).

``` r
sf_net <- st_read("preliminary_traffic_estimates_edinburgh.gpkg")
```

    Reading layer `preliminary_traffic_estimates_edinburgh' from data source 
      `C:\Users\ts18jpf\OneDrive - University of Leeds\03_PhD\00_Misc_projects\netvis-trafficflows\preliminary_traffic_estimates_edinburgh.gpkg' 
      using driver `GPKG'
    Simple feature collection with 58592 features and 18 fields
    Geometry type: LINESTRING
    Dimension:     XY
    Bounding box:  xmin: 289184 ymin: 637185.4 xmax: 365020.9 ymax: 717199.1
    Projected CRS: OSGB36 / British National Grid

``` r
plot(sf_net["geom"])
```

![](README_files/figure-commonmark/unnamed-chunk-2-1.png)

The original file covers a wide area. Let’s clip the `sf` using the
`zonebuilder` package to focus on Edinburgh.

``` r
bounds <- zb_zone("Edinburgh",n_circles = 4) |>
  st_transform(27700) # From Pseudo-mercator to Birtish Grid

edinburgh_AADT <- sf_net[bounds,] # Clipping

plot(edinburgh_AADT["geom"]) # A basic visualisation
```

![](README_files/figure-commonmark/net-clipping-1.png)

## Plotting the traffic flows

We are interested in the typical flows, in this case, Annaul Average
Daily Flows (AADF). For that purpose, let’s use base R to plot them.

``` r
plot(edinburgh_AADT["pred_flows"])
```

![](README_files/figure-commonmark/unnamed-chunk-4-1.png)

This is fine as an initial visualisation. However, we might be able to
improve it if we consider other elements of the road network. For
example, road class.

By using `tmap` or `ggplot,` we can easily tune some elements of the
visualisation. We are going to use `ggplog` in this case.

``` r
ggplot(edinburgh_AADT)+
  geom_sf(aes(col = pred_flows))
```

![](README_files/figure-commonmark/unnamed-chunk-5-1.png)

We can make the visualisation tidier if we eliminate the background.

``` r
ggplot(edinburgh_AADT)+
  geom_sf(aes(col = pred_flows))+
  theme_void()
```

![](README_files/figure-commonmark/unnamed-chunk-6-1.png)

The continuous scale for the flows is OK, but we could also use a binned
scale to better differentiate the traffic levels on different roads.

``` r
ggplot(edinburgh_AADT)+
  geom_sf(aes(col = pred_flows))+
  theme_void()+
  scale_color_binned()
```

![](README_files/figure-commonmark/unnamed-chunk-7-1.png)

We can also use a different palette

``` r
ggplot(edinburgh_AADT)+
  geom_sf(aes(col = pred_flows))+
  theme_void()+
  scale_color_binned(type = "viridis")
```

![](README_files/figure-commonmark/unnamed-chunk-8-1.png)

Let’s bring other element to improve the visualisation. Since roads are
not equally important in the road network, it would make sense to show
them using their relative importance (e.g. high volumes are expected to
be present on major roads). As a starting point, we can use the road
class. The following line of code extracts the road classes in the
dataset.

``` r
edinburgh_AADT |> pull(road_classification) |> unique()
```

    [1] "Unclassified"          "Classified Unnumbered" "Not Classified"       
    [4] "Unknown"               "B Road"                "A Road"               
    [7] "Motorway"             

We can create an ordered factor for the road classification and use it
to modify the line width.

``` r
roadclass_levels <- c("Not Classified",
                      "Unclassified",
                      "Unknown",
                      "Classified Unnumbered",
                      "B Road",
                      "A Road",
                      "Motorway") 

edinburgh_AADT |> 
  mutate(road_classification = factor(road_classification,
                                      levels = roadclass_levels,
                                      ordered = T)) |> 
  ggplot()+
  geom_sf(aes(col = pred_flows,linewidth = road_classification))+
  theme_void()+
  scale_color_binned(type = "viridis")
```

![](README_files/figure-commonmark/unnamed-chunk-10-1.png)

We can fine-tune the line-width scale to get a better output.

``` r
edinburgh_AADT |> 
  mutate(road_classification = factor(road_classification,
                                      levels = roadclass_levels,
                                      ordered = T)) |> 
  ggplot()+
  geom_sf(aes(col = pred_flows,linewidth = road_classification))+
  theme_void()+
  scale_color_binned(type = "viridis")+
  scale_linewidth_manual(values = 2*c(0.1,0.15,0.2,0.25,0.3,0.4,0.6))
```

![](README_files/figure-commonmark/unnamed-chunk-11-1.png)

By changing the `alpha` of the different road classes, we can get a
better result in areas with high road density e.g. residential areas.

``` r
edinburgh_AADT |> 
  mutate(road_classification = factor(road_classification,
                                      levels = roadclass_levels,
                                      ordered = T)) |> 
  ggplot()+
  geom_sf(aes(col = pred_flows,
              linewidth = road_classification,
              alpha = road_classification))+
  theme_void()+
  scale_color_binned(type = "viridis",)+
  scale_linewidth_manual(values = 2*c(0.1,0.15,0.2,0.25,0.3,0.4,0.6))
```

![](README_files/figure-commonmark/unnamed-chunk-12-1.png)

The binning for the flows can also be improved. Let’s try exploring the
distribution of the values and defining some arbitrary breaks. You can
always use common binning algorithms as [Jenks natural
breaks](https://search.r-project.org/CRAN/refmans/BAMMtools/html/getJenksBreaks.html).
Some of them are already built-in in `tmap`.

``` r
edinburgh_AADT |> 
  ggplot(aes(pred_flows))+
  geom_histogram(bins = 50)
```

![](README_files/figure-commonmark/unnamed-chunk-13-1.png)

``` r
breaks = c(0,1000,2000,5000,10000,25e3,1e5)

cols <- viridis::viridis(6)


edinburgh_AADT |> 
  mutate(road_classification = factor(road_classification,
                                      levels = roadclass_levels,
                                      ordered = T)) |> 
  ggplot()+
  geom_sf(aes(col = pred_flows,
              linewidth = road_classification,
              alpha = road_classification))+
  theme_void()+
  scale_color_stepsn(colours = cols,
                     breaks = breaks,
                     values = scales::rescale(breaks,to = c(0, 1)))+
  scale_linewidth_manual(values = 2.5*c(0.1,0.15,0.2,0.25,0.3,0.4,0.5))+
  scale_alpha_manual(values = c(0.1,0.3,0.45,0.5,0.6,0.8,1))
```

![](README_files/figure-commonmark/unnamed-chunk-13-2.png)

## To consider

- road hierarchy do not always explain the levels of flow e.g. rat-runs
  or poorly connected corridors
- directed networks are trickier:
  - overlapping edges/links
  - a lot more links
- Temporal component (Daily profiles)
- OD flows (Where trips go from/to)
