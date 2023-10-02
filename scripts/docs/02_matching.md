# (PART\*) Impact analysis {.unnumbered}

# Matching

In this R Markdown are performed the different steps to obtain a matched dataset, i.e a dataset with control and treated observational units to eventually compute the treatment effect. The treatment here is to be under protected area status, and we look at the impact on deforestation.

The steps are the following.

1.  Pre-processing : in a loop for each country,

    1.  create a gridding of the country;

    2.  import geospatial data on protected areas (PAs) from the World Dataset on Protected Areas (WDPA) and assign each observation unit/pixel to a group : PA of interest and analyzed (treated), PA of interest but not analyzed, PA not of interest, buffer (pixel closed to but not in a PA), other (so potential control). A PA of interest can be a PA known to be supported by the Agence Française de Développement (AFD) for instance. Some PAs are of interest but cannot be analyzed due to the design of the methodology (e.g marine protected areas when the focus is on deforestation);

    3.  compute the covariates and outcome of interest in all pixels thanks to the mapme.biodiversity package;

    4.  build the matching data frame : each pixel is assigned to a group and has covariates and outcome values.

2.  Post-processing : in each country,

    1.  Load the matching dataframe obtained at the end of pre-processing for a given country, and extract the list of protected areas to process.

    2.  For each protected area,

        1.  perform the matching;

        2.  plot covariate balance and density plots to assess the quality of the match;

        3.  panelize the dataframe;

        4.  plot the evolution of forest cover in treated and control areas, before and after matching;

        5.  map the matched treated and control units.

    3.  Map all matched treated and control units in the country.

The methodology is not extensively described here to keep the documentation concise. The interested reader can refer to the working paper for more details.

## Initial settings

Configuring the Rmarkdown

\#`{r setup, include=FALSE, eval = FALSE} #knitr::opts_knit$set(root.dir = rprojroot::find_rstudio_root_file())  #`

Downloading and installing the relevant packages


```r
#Install some libraries
## CRAN version
install.packages(c("tictoc", "geodata", "wdpar", "exactextractr", "MatchIt", "fixest", "cobalt", "future", "progressr", "mapme.biodiversity", "future.callr", "janitor", "geomtextpath", "rstac"))
## Github version (can be relevant if some features have not made it to CRAN version yet)
#remotes::install_github("mapme-initiative/mapme.biodiversity", upgrade="always")
#remotes::install_github("prioritizr/wdpar", upgrade="always")

#Install the web driver to download wdpa data directly
webdriver::install_phantomjs()  

# Load Libraries
library(dplyr)
library(janitor) #Functions to automate name cleaning
library(tictoc) #For timing
library(xtable) #Export dataframes as tables
library(tidyr)
library(stringr) #String specific functions
library(ggplot2) # For plotting
library(geomtextpath) #For annoted vertical lines in ggplot
library(RColorBrewer) #Improved color palettes for plot legends
library(ggrepel) #Refine labelling of some figures
library(sf) # For handling vector data
library(terra) # For handling raster data
library(raster) # For handling raster data
library(rgeos) 
library(geodata) # For getting country files
library(wdpar) # For getting protected areas
library(exactextractr) # For zonal statistics
library(mapme.biodiversity) #Download geospatial data and compute specific indicators
library(rstac) #To downlad NASA SRTM data
library(aws.s3) #Access to storage
library(MatchIt) #For matching
library(fixest) #For estimating the models
library(cobalt) #To visualize density plots and covariate balance from MatchIt outcomes
library(future) #For parallel computing in mapme.biodiversity
library(future.callr)  #For parallel computing in mapme.biodiversity
library(progressr) # To display progress bar   
```

Load the R functions called in the data processing


```r
#Import functions
source("scripts/functions/02_fns_matching.R")      
```

## Datasets and critical parameters


```r
# Define working directories
## Define the path to a temporary, working directory processing steps.
tmp_pre = paste(tempdir(), "matching_pre", sep = "/")
tmp_post = paste(tempdir(), "matching_post", sep = "/")
## Define a directory where outputs are stored in SSPCloud.
save_dir = paste("impact_analysis/matching", Sys.Date(), sep = "/") #Today's date
# save_dir = paste("impact_analysis/matching", "2023-08-29", sep = "/") #A specific date

# Load datasets
## WDPA database   
## Download and save
# wdpa_wld_raw = wdpa_fetch(x = "global", wait = TRUE, download_dir = tmp_pre, page_wait = 2, verbose = TRUE)
# s3write_using(wdpa_wld_raw,
#               sf::st_write,
#               delete_dsn = TRUE,
#               object = paste0("data_raw/wdpa/wdpa_shp_global_raw.gpkg"),
#               bucket = "projet-afd-eva-ap",   
#               opts = list("region" = ""))

##Load
wdpa_wld_raw = s3read_using(
              sf::st_read,
              object = "data_raw/wdpa/wdpa_shp_global_raw.gpkg",
              bucket = "projet-afd-eva-ap",
              opts = list("region" = ""))

## Dataset specific to the PAs portfolio to analyze. Only one is selected depending on the analysis one wants to perform. 

### PAs supported by the AFD
# data_pa =
#   #fread("data_tidy/BDD_PA_AFD_ie.csv" , encoding = "UTF-8")
#   aws.s3::s3read_using(
#   FUN = data.table::fread,
#   encoding = "UTF-8",
#   object = "data_tidy/BDD_PA_AFD_ie.csv",
#   bucket = "projet-afd-eva-ap",
#   opts = list("region" = "")) %>%
#   #Sangha trinational (555547988) created in 2012 actually gathers three former PAs
#   #in CAF (31458), CMR (1245) and COG (72332) implemented in
#   #1990, 2001 and 1993 respectively.
#   # Evaluating the trinational PA is not relevant here : our method relies on pre-treatment obervsations (for matching and DiD) and the outcome is likely to be affected by the initial PAs. On the other hand, evaluating the three earlier PAs might be irrelevant for us : are they funded by the AFD ?? In a first approach, the trinational is removed.
#   filter(is.na(wdpaid) == TRUE | wdpaid != 555547988)

## PAs supported by the FAPBM
# data_pa =
#   #fread("data_tidy/BDD_PA_AFD_ie.csv" , encoding = "UTF-8")
#   aws.s3::s3read_using(
#   FUN = data.table::fread,
#   encoding = "UTF-8",
#   object = "data_tidy/BDD_PA_FAPBM.csv",
#   bucket = "projet-afd-eva-ap",
#   opts = list("region" = ""))

## All PAs in Madagascar
data_pa =
  #fread("data_tidy/BDD_PA_AFD_ie.csv" , encoding = "UTF-8")
  aws.s3::s3read_using(
  FUN = data.table::fread,
  encoding = "UTF-8",
  object = "data_tidy/BDD_PA_MDG.csv",
  bucket = "projet-afd-eva-ap",
  opts = list("region" = ""))

# The list of countries (ISO3 codes) to analyze. This can be define manually or from the the dataset loaded.
#List of countries in the sample
# list_iso_africa = unique(data_pa[data_pa$region == "Africa", iso3])
list_iso = "MDG"
      
# Specify buffer width in meter
buffer_m = 10000
# Specify the grid cell size in meter
#gridSize = 10000 
# Specify to sampling : the ideal, minimal number of pixels in a protected area. 
## Note that by design this number is indicative, as the pixel size is defined from the protected area reported surface and sampling number, but the area considered in the analysis is the terrestrial area. For PAs with a marine part, the area analyzed is smaller and the number of pixels mechanically lower.
n_sampling = 1000 

#Specify the period of study to create the mapme.bidiversity portfolio
## Start year
yr_first = 2000
## End year
yr_last = 2021

#Minimum treatment year
#At least two pre-treatment periods of forest cover are needed to compute average pre-treatment deforestation, used as a matching variable.
yr_min = yr_first+2

# Define column names of matching covariates
colname.travelTime = "minutes_median_5k_110mio"
colname.clayContent = "clay_0_5cm_mean"
colname.elevation = "elevation_mean"
colname.tri = "tri_mean"
colname.fcIni = "treecover_2000"
colname.flAvg = "avgLoss_pre_treat"

#Matching 
## Parameters
match_method = "cem"
cutoff_method = "sturges"
k2k_method = "mahalanobis"
is_k2k = TRUE
## Criteria to assess matching quality
### Standardized absolte mean difference : threshold
th_mean = 0.1
### Variance ratio : thresholds
th_var_min = 0.5
th_var_max = 2
```

## Matching process

The following code is divided into pre- and post-processing steps (see above). At pre-processing stage, computations are done country-by-country. At post-proccessing stage, computations are done country-by-country and protected areas by protected areas. To facilitate the reading, each step consists in a call of a function define in an other R script.

During the process, a text file (so-called log) is edited to keep track of the differents steps. Then after each critical step, the code checks whether an error occured by interrogating the variable is_ok (defined in the function corresponding to the step). If the step is ok (is_ok = TRUE) then the processing continues. Otherwise, the code goes to the next iteration (next country for pre-processing, next protected area for post-processing). This is useful in a multi-country, multi-PA analysis, to avoid the code to stop when an error occurs. Instead, the code continue and the analyst can see in the log whether there have been errors during the processing, where it happened and whether he or she needs to launch the analysis again for a specific country/PA. Generally speaking, this log is useful to remember what has been analyzed and assess everything was fine after the processing (warnings, processing of all the countries and PAs, etc.).

For more details about the each step, please refer to the definition of the functions.


```r
##########
### PRE-PROCESSING
##########

#For each country in the list, the different steps of the pre-processing are performed, and the process duration computed
count = 0 #Initialize counter
max_i = length(list_iso) #Max value of the counter
tic_pre = tic() #Start timer

#Create a log to track progress of the analysis
log = fn_pre_log(list_iso,
                 buffer = buffer_m,
                 sampling = n_sampling,
                 yr_first = yr_first,
                 yr_last = yr_last,
                 yr_min = yr_min,
                 name = paste0("log-", Sys.Date(), "-NAME.txt"),
                 notes = "Specific notes or remarks.")

# Perform pre-processing steps country-by-country
for (i in list_iso)            
{
  #Update counter and display progress
  count = count+1
  print(paste0(i, " : country ", count, "/", max_i))
  
  #Append the log to track progress of the process on country i
  cat(paste("#####\nCOUNTRY :", i, "\n#####\n\n"), file = log, append = TRUE)
     
  #Generate observation units
  print("--Generating observation units")
  output_grid = fn_pre_grid(iso = i, 
                            yr_min = yr_min,
                            path_tmp = tmp_pre, 
                            data_pa = data_pa,
                            sampling = n_sampling,
                            log = log,
                            save_dir = save_dir)
  if(output_grid$is_ok == FALSE) {next}  
  
  #Load the outputs 
  utm_code = output_grid$utm_code #UTM code 
  gadm_prj = output_grid$ctry_shp_prj #The country polygon with relevant projection
  grid = output_grid$grid #The country gridding
  gridSize = output_grid$gridSize #The spatial resolution of the gridding
  
  #Determining Group IDs and WDPA IDs for all observation units
  print("--Determining Group IDs and WDPA IDs")
  output_group = fn_pre_group(iso = i, wdpa_raw = wdpa_wld_raw,
                              status = c("Proposed", "Designated", "Inscribed", "Established"),
                            yr_min = yr_min,
                            path_tmp = tmp_pre, utm_code = utm_code,
                            buffer_m = buffer_m, data_pa = data_pa,
                            gadm_prj = gadm_prj, grid = grid, 
                            gridSize = gridSize,
                            log = log,
                            save_dir = save_dir)
  if(output_group$is_ok == FALSE) {next} else grid_param = output_group$grid.param

  #Calculating outcome and other covariates for all observation units
  print("--Calculating outcome and other covariates")
  output_mf = 
    fn_pre_mf_parallel(grid.param = grid_param, 
                       path_tmp = tmp_pre, 
                       iso = i,
                       name_output = paste0("matching_frame_spling", n_sampling),
                       ext_output = ".gpkg",
                       yr_first = yr_first, yr_last = yr_last,  
                       log = log,
                       save_dir = save_dir)  
  if(output_mf$is_ok == FALSE) {next}                                            
  
  #Remove files in the session memory, to avoid saturation
  tmp_files = list.files(tmp_pre, include.dirs = T, full.names = T, recursive = T)
  file.remove(tmp_files)
                                  
}                            
  
  #End timer for pre-processing
  toc_pre = toc()
  
  #Append the log
  cat(paste("END OF PRE-PROCESSING :", toc_pre$callback_msg, "\n\n"), 
      file = log, append = TRUE)

  
##########
### POST-PROCESSING
##########
  
           
#For each country in the list, the different steps of the post-processing are performed, and duration of the processing computed
count_i = 0 #Initialize counter
max_i = length(list_iso) #Max value of the counter
tic_post = tic() #start timer

#Append the log, and specify matching parameters and quality assessment
cat(paste("##########\nPOST-PROCESSING\n##########\n\nPARAMETERS :\nMatching\n#Parameters\n##Method :", match_method, "\n##Automatic cutoffs :", cutoff_method, "\n##Is it K2K matching ?", is_k2k, "\n##K2K method :", k2k_method, "\n#Quality assessement\n##Absolute standardized mean difference (threshold)", th_mean, "\n##Variance ratio between", th_var_min, "and", th_var_max), 
    file = log, append = TRUE)
  
# Perform post-processing steps country-by-country, area-by-area
## Loop over country
for (i in list_iso)
{
  #Update counter and show progress
  count_i = count_i+1
  print(paste0(i, " : country ", count_i, "/", max_i))
  
  #Append the log to track progress of the process on country i
  cat(paste("#####\nCOUNTRY :", i, "\n"), file = log, append = TRUE)
  
  #Load the matching frame
  print("--Loading the matching frame")
  output_load = fn_post_load_mf(iso = i, 
                           yr_min = yr_min,
                           name_input = paste0("matching_frame_spling", n_sampling),
                           ext_input = ".gpkg",
                           log = log,
                           save_dir = save_dir)
  if(output_load$is_ok == FALSE) {next} else mf_ini = output_load$mf
  
  list_pa = unique(mf_ini[mf_ini$wdpaid != 0, ]$wdpaid)
  
    #Append the log : list of PAs analyzed in the matching frame
  cat(paste("LIST OF WDPAIDs :", paste(list_pa, collapse = ", "), "\n#####\n\n"), 
      file = log, append = TRUE)
    
  #Initialization
  ##Counter
  count_j = 0
  max_j = length(list_pa)
  ##List of control and treatment pixels matched
  df_pix_matched = data.frame()
  
  #Loop over the different PAs
  for (j in list_pa)
  {
    #Update counter and show progress
    count_j = count_j+1
    print(paste0("WDPAID : ", j, " : ", count_j, "/", max_j))
    
    #Append the log to track progress of the process on PA j
    cat(paste("###\nWDPAID :", j, "\n###\n\n"), file = log, append = TRUE)
  
    mf_ini_j = mf_ini %>%
      filter(group == 1 | (group == 2 & wdpaid == j))
    
    #Add average pre-loss
    print("--Add covariate : average tree loss pre-funding")
    output_avgLoss = fn_post_avgLoss_prefund(mf = mf_ini_j, 
                                             log = log)
    if(output_avgLoss$is_ok == FALSE) {next} else mf_j = output_avgLoss$mf
    
    #Run Coarsened Exact Matching
    print("--Run CEM")
    output_cem = fn_post_match_auto(mf = mf_j, iso = i, 
                                   dummy_int = FALSE,
                                   match_method = match_method,
                                   cutoff_method = cutoff_method,
                                   is_k2k = is_k2k,
                                   k2k_method = k2k_method,
                                     th_mean = th_mean, 
                                     th_var_min = th_var_min, th_var_max = th_var_max,
                                   colname.travelTime = colname.travelTime, 
                                   colname.clayContent = colname.clayContent, 
                                   colname.elevation = colname.elevation,
                                   colname.tri = colname.tri, 
                                   colname.fcIni = colname.fcIni, 
                                   colname.flAvg = colname.flAvg,
                                   log = log)
    if(output_cem$is_ok == FALSE) {next} else out_cem_j = output_cem$out.cem
    
    #Plots : covariates
    print("--Some plots : covariates")
    print("----Covariate balance")
    output_covbal = fn_post_covbal(out.cem = out_cem_j,
                   mf = mf_j,
                   colname.travelTime = colname.travelTime, 
                   colname.clayContent = colname.clayContent,
                   colname.fcIni = colname.fcIni, 
                   colname.flAvg = colname.flAvg,
                   colname.tri = colname.tri,
                   colname.elevation = colname.elevation,
                   iso = i,
                   path_tmp = tmp_post,
                   wdpaid = j,
                   log = log,
                   save_dir = save_dir)
  if(output_covbal$is_ok == FALSE) {next}
    
    print("----Density plots")
    output_density = fn_post_plot_density(out.cem = out_cem_j,  
                                         mf = mf_j,
                                      colname.travelTime = colname.travelTime, 
                                       colname.clayContent = colname.clayContent,
                                       colname.fcIni = colname.fcIni, 
                                       colname.flAvg = colname.flAvg,
                                    colname.tri = colname.tri,
                                   colname.elevation = colname.elevation,
                                      iso = i,
                                      path_tmp = tmp_post,
                                      wdpaid = j,
                                   log = log,
                                   save_dir = save_dir)
     if(output_density$is_ok == FALSE) {next}
    
    #Panelize dataframes
    print("----Panelize (Un-)Matched Dataframe")
    output_panel = fn_post_panel(out.cem = out_cem_j, 
                                  mf = mf_j, 
                                  ext_output = ".csv", 
                                  iso = i,
                                  wdpaid = j,
                                  log = log,
                                 save_dir = save_dir)
     if(output_panel$is_ok == FALSE) {next}
    
    matched.wide.j = output_panel$matched.wide
    unmatched.wide.j = output_panel$unmatched.wide
    matched.long.j = output_panel$matched.long
    unmatched.long.j = output_panel$unmatched.long 
    
    #Extract matched units and plot them on a grid
    print("----Extract matched units and plot them on a grid")
    
    ##Extract ID of treated and control pixels
    df_pix_matched_j = matched.wide.j %>%
      st_drop_geometry() %>%
      as.data.frame() %>%
      dplyr::select(c(group, assetid)) %>%
      rename("group_matched" = "group") 
    df_pix_matched = rbind(df_pix_matched, df_pix_matched_j)
    
    ##Plot the grid with matched control and treated for the PA
    output_grid = fn_post_plot_grid(iso = i, wdpaid = j,
                      is_pa = TRUE,
                      df_pix_matched = df_pix_matched_j,
                      path_tmp = tmp_post,
                      log = log,
                      save_dir = save_dir)
     if(output_grid$is_ok == FALSE) {next}

    #Plots the evolution of forest cover for treated and control units, before and after matching
    print("----Plots again : trend")
    output_trend = fn_post_plot_trend(matched.long = matched.long.j, 
                       unmatched.long = unmatched.long.j, 
                       mf = mf_j,
                       data_pa = data_pa,
                       iso = i,
                       wdpaid = j,
                       log = log,
                       save_dir = save_dir)
    if(output_trend$is_ok == FALSE) {next}
  }
    
  # Plot the grid with matched control and treated for the country 
  output_grid = fn_post_plot_grid(iso = i, wdpaid = j,
                    is_pa = FALSE,
                    df_pix_matched = df_pix_matched,
                    path_tmp = tmp_post,
                    log = log,
                    save_dir = save_dir)
   if(output_grid$is_ok == FALSE) {next}
  
}       

#End post-processing timer
toc_post = toc()

#Append the log and save it
cat(paste("END OF POST-PROCESSING :", toc_post$callback_msg, "\n\n"),
    file = log, append = TRUE)
aws.s3::put_object(file = log,
                   bucket = paste("projet-afd-eva-ap", save_dir, sep = "/"),
                   region = "",
                   show_progress = FALSE)
                 
                               
#Notes on what to do next
## Automate the definition of cutoffs for CEM
### Coder 5.5.3 de Iacus et al. 2012 ? Permet de savoir le gain de matched units pour une modification des seuils d'une variable
## Allow to enter a list of any covariates to perform the matching
## Function to plot Fig. 3 in Iacus et al. 2012
## On veut ATE ou ATT ?? Je dirai ATT car on ne veut pas estimer l'effet de mettre une AP, mais l'effet des AP financés par l'AFD 
```