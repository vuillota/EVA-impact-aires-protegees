# (APPENDIX) Appendix {-}

# Functions for matching pre- and post-processing


```r
#####
#Functions for matching process
#####

#For each function, the aim of the function, inputs, outputs, data saved and notes are detailed. This takes the following form :
#Aim of the function
##INPUTS : the arguments needed in the function
###INPUT 1 to N
##OUTPUTS : the information returned by the function (data frames, numeric, characters, etc.) and necessary to pursue to processing
### OUTPUT 1 to N
##DATA SAVED : information put in the storage but not necessarily need to pursue the processing (figures, tables, data frames, etc.)
### ...
##NOTES : any useful remark
### ...

#Remarks :
##most functions are adapted for errors handling using base::withCallingHandlers(). Basically, the computation steps are declared in a block of withCallingHandlers function, while two other blocks specify what to do in case the first block face a warning or error. In our case, errors led to return a boolean indicating an error has occured and append the log with the error message. Warnings return a boolean but do not block the iteration. They also edit the log with the warning message.
##PA is used for "protected area(s)".
##To save plots and tables : save on temporary folder in the R session then put the saved object in the storage. Indeed print() and ggplot::ggsave() cannot write directly on s3 storage
###


#Pre-processing
###

#Create a log to track progress of the processing (warnings, errors, parameters, country and PAs analyzed, etc.)
##INPUTS :
###list_iso : the list of ISO3 code corresponding to the countries analyzed
### buffer : the buffer width in meter
### sampling : the sampling specified by the user for the smaller area in the country considered
### yr_first : the first year of the period where the analysis takes place
### yr_last : the last year of the period where the analysis takes place
### yr_min : the minimum treatment year to be considered in the analysis. As some matching covariates are defined with pre-treatment data (e.g average tree cover loss before treatment), this minimal year is greater than yr_first
### name : specify the name of the log file to save
### notes : any notes on the analysis performed
##OUTPUTS :
###log : a text file in the R session memory that will be edited through the data processing
fn_pre_log = function(list_iso, buffer, sampling, yr_first, yr_last, yr_min, name, notes)
{
  str_iso = paste(list_iso, collapse = ", ")
  log = paste(tempdir(), name, sep = "/")
  file.create(log)
  #Do not forget to end the writing with a \n to avoid warnings
  #cat(paste("#####\nCOUNTRY :", iso, "\nTIME :", print(Sys.time(), tz = "UTC-2"), "\n#####\n\n###\nPRE-PROCESSING\n###\n\n"), file = log, append = TRUE)
  cat(paste("STARTING TIME :", print(Sys.time(), tz = "UTC-2"), "\nPARAMETERS : buffer =", buffer, "m, sampling size of", sampling, ", period of analysis", yr_first, "to", yr_last, ", minimum treatment year is", yr_min, "\nCOUNTRIES :", str_iso, "\nNOTES :", notes, "\n\n##########\nPRE-PROCESSING\n##########\n\n"), file = log, append = TRUE)
  
  return(log)
}


#Find the UTM code for a given set of coordinates
##INPUTS : 
### lonlat : coordinates
##OUTPUTS : 
### UTM code
fn_lonlat2UTM = function(lonlat) 
{
  utm = (floor((lonlat[1] + 180) / 6) %% 60) + 1
  if (lonlat[2] > 0) {
    utm + 32600
  } else{
    utm + 32700
  }
  
}


#Create the gridding of a given country. 
##INPUTS : 
### iso : ISO code 
### yr_min : the minimum treatment year to be considered in the analysis. As some matching covariates are defined with pre-treatment data (e.g average tree cover loss before treatment), this minimal year is greater than the first year in the period considered
### path_tmp : temporary path for saving figures
### data_pa : dataset with information on protected areas, and especially their surfaces
### sampling : Number of pixels that subdivide the protected area with lowest area in the country considered
### log : a log file to track progress of the processing
### save_dir : saving directory
##OUTPUTS (depending on potential errors)
### gadm_prj : country shapefile 
### grid : gridding of the country
### utm_code : UTM code of the country
### gridSize : the resolution of gridding, defined from the area of the PA with the lowest area
### is_ok : a boolean indicating whether or not an error occured inside the function
##DATA SAVED :
### Gridding of the country considered
fn_pre_grid = function(iso, yr_min, path_tmp, data_pa, sampling, log, save_dir)
{
  
  output = withCallingHandlers(
    
    {
      
  # Download country polygon
  gadm = gadm(country = iso, resolution = 1, level = 0, path = path_tmp) %>% 
    st_as_sf() %>%
    st_make_valid() #Necessary for some polygons : e.g BEN
  
  # Find UTM zone of the country centroid
  centroid = st_coordinates(st_centroid(gadm))
  utm_code = fn_lonlat2UTM(centroid)
  # Reproject GADM
  gadm_prj = gadm %>% 
    st_transform(crs = utm_code)
  
  #Determine relevant grid size
  ##Select the PA in the country with minimum area. PAs with null areas, marine or treatment year before 2000 are discarded (not analyzed anyway)
  pa_min = data_pa %>%
    filter(iso3 == iso & is.na(wdpaid) == FALSE & status_yr >= yr_min & marine %in% c(0,1)) %>%
    arrange(area_km2) %>%
    slice(1)
  ##From this minimum area, define the grid size. 
  ##It depends on the sampling of the minimal area, i.e how many pixels we want to subdivide the PA with lowest area
  ## To avoid a resolution higher than the one of our data, grid size is set to be 30m at least (resolution of tree cover data, Hansen et al. 2013)
  area_min = pa_min$area_km2 #in kilometer
  gridSize = max(1e3, round(sqrt(area_min/sampling)*1000, 0)) #Side of the pixel is expressed in meter and rounded, if above 1km. 
  
  # Make bounding box of projected country polygon
  bbox = st_bbox(gadm_prj) %>% st_as_sfc() %>% st_as_sf() 
  # Make a Grid to the extent of the bounding box
  grid.ini = st_make_grid(bbox, cellsize = c(gridSize,gridSize))
  # Crop Grid to the extent of country boundary by
  # subsetting to the grid cells that intersect with the country
  grid.sub = grid.ini %>% 
    st_intersects(gadm_prj, .) %>% 
    unlist()
  # Filter the grid to the subset
  grid = grid.ini[sort(grid.sub)] %>%
    st_as_sf() %>%
    mutate(gridID = seq(1:nrow(.))) # Add id for grid cells
  
  #Extract country name
  country.name = data_pa %>% 
    filter(iso3 == iso) %>% 
    slice(1)
  country.name = country.name$country_en
  
  #Visualize and save the grid
  fig_grid = ggplot() +
    geom_sf(data = st_geometry(bbox)) +
    geom_sf(data = st_geometry(gadm_prj)) +
    geom_sf(data = st_geometry(grid), alpha = 0) +
    labs(title = paste("Gridding of", country.name))
  fig_save = paste0(path_tmp, "/fig_grid_", iso, ".png")
  ggsave(fig_save,
         plot = fig_grid,
         device = "png",
         height = 6, width = 9)
  aws.s3::put_object(file = fig_save, 
                     bucket = paste("projet-afd-eva-ap", save_dir, iso, sep = "/"), 
                     region = "", 
                     show_progress = FALSE)
  
  #Append the log 
  cat("#Generating observation units\n-> OK\n", file = log, append = TRUE)
  
  #Return outputs
  list_output = list("ctry_shp_prj" = gadm_prj, 
                     "grid" = grid, 
                     "gridSize" = gridSize, 
                     "utm_code" = utm_code,
                     "is_ok" = TRUE)
  return(list_output)
  
    },
  
  error = function(e)
  {
    #Print the error and append the log
    print(e)
    #Append the log 
    cat(paste("#Generating observation units\n-> Error :\n", e, "\n"), file = log, append = TRUE)
    #Return string to inform user to skip
    return(list("is_ok" = FALSE))
  },
  
  warning = function(w)
  {
    #Print the warning and append the log
    print(w)
    #Append the log 
    cat(paste("#Generating observation units\n-> Warning :\n", w, "\n"), file = log, append = TRUE)
    #Return string to inform user to skip
    return(list("is_ok" = TRUE))
  }
  
  )
  
  return(output)
}

#Assign each pixel (observation unit) to a group : PA non-funded, funded and analyzed, funded and not analyzed, buffer, potential control. 
##INPUTS :
### iso : country ISO code
### path_tmp : temporary path to save figures
### utm_code : UTM of the country centroid
### buffer_m : buffer width, in meter
### data : a dataframe with the WDPAID of PAs funded by the AFD
### gadm_prj : country polygon, projected so that crs = UTM code
### grid : gridding of the country
### gridSize : resolution of the gridding
##OUTPUTS (depending on potential errors): 
### grid.param : a raster representing the gridding of the country with two layers. One for the group each pixel belongs to (funded PA, non-funded PA, potential control, buffer), the other for the WDPAID corresponding to each pixel (0 if not a PA)
### is_ok : a boolean indicating whether or not an error occured inside the function
##DATA SAVED :
### grid.param
### A plot of the country gridding with group of each pixel
### The share of PAs in the portfolio considered that are reported in the WDPA
### In the country considered, the share of PAs in the portfolio and analyzed, not analyzed or not in the portfolio
### Share of PAs reported in the WDPA and analyzed in the country considered
##NOTES :
### Errors can arise from the wdpa_clean() function, during "formatting attribute data" step. Can be settled playing with geometry_precision parameter
fn_pre_group = function(iso, wdpa_raw, status, yr_min, path_tmp, utm_code, buffer_m, data_pa, gadm_prj, grid, gridSize, log, save_dir)
{
  
  output = withCallingHandlers(
    
    {

  # The polygons of PAs are taken from WDPA, cleaned
  wdpa_prj = wdpa_raw %>%
    filter(ISO3 == iso) %>%
    #st_make_valid() %>%
    #celanign of PAs from the wdap_clean function :
    #Status filtering is performed manually juste after. 
    #The geometry precision is set to default. Used to be 1000 in Kemmeng code
    # Overlaps are not erased because we rasterize polygons
    #UNESCO Biosphere Reserves are not excluded so that our analysis of AFD portfolio is the most extensive
    wdpa_clean(retain_status = status, #NULL to remove proposed
               erase_overlaps = FALSE,
               exclude_unesco = FALSE,
               verbose = TRUE) %>% 
    # Remove the PAs that are only proposed, or have geometry type "point"
    #filter(STATUS != "Proposed") %>%  #24/08/2023 : "Proposed" status concerns only 6 PAs in the sample, including one implemented after 2000.
    filter(GEOMETRY_TYPE != "POINT") %>%
    # Project PA polygons to the previously determined UTM zone
    st_transform(crs = utm_code) 
  
  # Make Buffers around all protected areas
  buffer = st_buffer(wdpa_prj, dist = buffer_m) %>% 
    # Assign an ID "5" to the buffer group
    mutate(group=5,
           group_name = "Buffer")
  
  # Separate funded and non-funded protected areas
  ##PAs funded by AFD 
  ###... which can bu used in impact evaluation : in the country of interest, wdpaid known, area above 1km² (Wolf et al. 2021), implemented after yr_min defined by the user, non-marine (terrestrial or coastal, Wolf et al. 2021)
  pa_afd_ie = data_pa %>%
    filter(iso3 == iso & is.na(wdpaid) == FALSE & area_km2 > 1 & status_yr >= yr_min & marine %in% c(0,1))
  wdpaID_afd_ie = pa_afd_ie[pa_afd_ie$iso3 == iso,]$wdpaid
  wdpa_afd_ie = wdpa_prj %>% filter(WDPAID %in% wdpaID_afd_ie) %>%
    mutate(group=2,
           group_name = "Funded PA, analyzed") # Assign an ID "2" to the funded PA group
  ###...which cannot
  pa_afd_no_ie = data_pa %>%
    filter(iso3 == iso & (is.na(wdpaid) == TRUE | area_km2 <= 1 | is.na(area_km2) | status_yr < yr_min | marine == 2)) #PAs not in WDPA, of area less than 1km2 (Wolf et al 2020), not terrestrial/coastal or implemented after yr_min are not analyzed
  wdpaID_afd_no_ie = pa_afd_no_ie[pa_afd_no_ie$iso3 == iso,]$wdpaid 
  wdpa_afd_no_ie = wdpa_prj %>% filter(WDPAID %in% wdpaID_afd_no_ie) %>%
    mutate(group=3,
           group_name = "Funded PA, not analyzed") # Assign an ID "3" to the funded PA group which cannot be stuided in the impact evaluation
  ##PAs not funded by AFD
  wdpa_no_afd = wdpa_prj %>% filter(!WDPAID %in% c(wdpaID_afd_ie, wdpaID_afd_no_ie)) %>% 
    mutate(group=4,
           group_name = "Non-funded PA") # Assign an ID "4" to the non-funded PA group
  wdpaID_no_afd = wdpa_no_afd$WDPAID
  
  # Merge the dataframes of funded PAs, non-funded PAs and buffers
  # CAREFUL : the order of the arguments does matter. 
  ## During rasterization, in case a cell of the raster is on both funded analysed and non-funded, we want to cell to take the WDPAID of the funded analysed.
  ## Same funded, not analyzed. As the first layer is taken, wdpa_afd_ie needs to be first !
  wdpa_groups = rbind(wdpa_afd_ie, wdpa_afd_no_ie, wdpa_no_afd, buffer)
  # Subset to polygons that intersect with country boundary
  wdpa.sub = wdpa_groups %>% 
    st_intersects(gadm_prj, .) %>% 
    unlist()
  # Filter the PA+buffer to the subset
  wdpa_groups = wdpa_groups[sort(wdpa.sub),] %>%
    st_as_sf()
  
  # Initialize an empty raster to the spatial extent of the country
  r.ini = raster()
  extent(r.ini) = extent(gadm_prj)
  # Specify the raster resolution as same as the pre-defined 'gridSize'
  res(r.ini) = gridSize
  # Assign the raster pixels with "Group" values, 
  # Take the minimal value if a pixel is covered by overlapped polygons, so that PA Group ID has higher priority than Buffer ID.
  # Assign value "0" to the background pixels (control candidates group)
  # fun = "min" can lead to bad group assignment. This issue is developed and tackled below
  r.group = rasterize(wdpa_groups, r.ini, field="group", fun="min", background=0) %>%
    mask(., gadm_prj)
  # Rename Layer
  names(r.group) = "group"
  
  # Rasterize wdpaid
  ## CAREFUL : as stated above, the wdpa_groups raster is ordered so that the first layer is the one of funded, analyzed PA. Thus one needs to have fun = "first"
  r.wdpaid = rasterize(wdpa_groups, r.ini, field="WDPAID", fun="first", background=0) %>%
    mask(., gadm_prj)
  names(r.wdpaid) = "wdpaid"
  
  # Aggregate pixel values by taking the majority
  grid.group.ini = exact_extract(x=r.group, y=grid, fun='mode', append_cols="gridID") %>%
    rename(group = mode)
  grid.wdpaid = exact_extract(x=r.wdpaid, y=grid, fun="mode", append_cols="gridID") %>%
    rename(wdpaid = mode)

  # Randomly select background pixels as potential control pixels
  ##Take the list of background pixels, the  number of background and treatment pixels
  list_back_ID = grid.group.ini[grid.group.ini$group == 0 & is.na(grid.group.ini$group) == FALSE,]$gridID
  n_back_ID = length(list_back_ID)
  n_treat = length(grid.group.ini[grid.group.ini$group == 2 & is.na(grid.group.ini$group) == FALSE,]$gridID)
  ##The number of potential control units is five times the number of treatment units
  n_control = min(n_back_ID, n_treat*5)
  ##Select randomly the list of background pixels selected as controls
  ### Note that we control for the case n_back_ID = 1, which causes weird behavior using sample()
  set.seed(0) #To ensure reproductibility of the random sampling
  if(n_back_ID <= 1) list_control_ID = list_back_ID else list_control_ID = sample(x = list_back_ID, size = n_control, replace = FALSE)
  ## Finally, assign the background pixel chosen to the control group, characterized by group = 1
  grid.group = grid.group.ini %>%
    mutate(group = case_when(gridID %in% list_control_ID ~ 1,
                             TRUE ~ group))
  

  # Merge data frames
  grid.param.ini = grid.group %>%
    merge(., grid.wdpaid, by="gridID") %>%
    merge(., grid, by="gridID") %>%
    # drop rows having "NA" in column "group"
    drop_na(group) %>%
    st_as_sf() %>%
    # Grid is projected to WGS84 because mapme.biodiverty package merely works with this CRS
    st_transform(crs=4326) %>%
    #Add treatment year variable
    left_join(dplyr::select(data_pa, c(region_afd, region, sub_region, country_en, iso3, wdpaid, status_yr, year_funding_first, year_funding_all)), by = "wdpaid")
  
  # If two PAs in different groups overlap, then the rasterization with fun = "min" (as in r.group definition) can lead to bad assignment of pixels.
  # For instance, if a PA non-funded (group = 4) overlaps with a funded, analyzed one (group = 2), then the pixel will be assigned to the group 2
  # Same for group 3 (funded, not analyzed). Then, the following correction is applied.
  # Finally, each group is given a name for later plotting
  # grid.param = grid.param.ini %>%
  #   mutate(group = case_when(wdpaid %in% wdpaID_no_afd & group == 2 ~ 4,
  #                            wdpaid %in% wdpaID_afd_no_ie & group == 2 ~3,
  #                            TRUE ~ group)) %>%
  
  #/!\ For the moment, a pixel both non-funded and funded is considered funded !
  #But if funded not analyzed AND funded analyzed, then funded not analyzed.
  #Idea : the pixel could be treated out of the period considered, so not comparable to toher treatment pixels considered in funded, analyzed.
  # -> Check with Léa, Ingrid and PY if that seems OK
  grid.param = grid.param.ini %>%
    mutate(group = case_when(wdpaid %in% wdpaID_afd_no_ie & group == 2 ~ 3,
                             TRUE ~ group)) %>%
    #Add name for the group
    mutate(group_name = case_when(group == 0 ~ "Background",
                                  group == 1 ~ "Potential control",
                                  group == 2 ~ "Funded PA, analyzed (potential treatment)",
                                  group == 3 ~ "Funded PA, not analyzed",
                                  group == 4 ~ "Non-funded PA",
                                  group == 5 ~ "Buffer")) %>%
  #Add spatial resolution in m : useful to compute share of forest area in a given pixel and extrapolate to the PA for instance
  mutate(res_m = gridSize)
  
  
  #Save the grid
  s3write_using(grid.param,
                sf::write_sf,
                overwrite = TRUE,
                object = paste0(save_dir, "/", iso, "/", paste0("grid_param_", iso, ".gpkg")),
                bucket = "projet-afd-eva-ap",
                opts = list("region" = ""))
  
  # Visualize and save grouped grid cells
  
  ## Extract country name
  country.name = grid.param %>% 
    filter(group == 2) %>% 
    slice(1)
  country.name = country.name$country_en
  
  fig_grid_group = 
    ggplot(grid.param) +
    geom_sf(aes(fill = as.factor(group_name)), color = NA) +
    labs(title = paste("Gridding of", country.name)) +
    scale_fill_brewer(name = "Group", type = "qual", palette = "YlGnBu", direction = -1) +
    # scale_color_viridis_d(
    #   # legend title
    #   name="Group", 
    #   # legend label
    #   labels=c("control candidate", "treatment candidate", "non-funded PA", "buffer zone")) +
    theme_bw()
  fig_save = paste0(path_tmp, "/fig_grid_group_", iso, ".png")
  ggsave(fig_save,
         plot = fig_grid_group,
         device = "png",
         height = 6, width = 9)
  aws.s3::put_object(file = fig_save,
                     bucket = paste("projet-afd-eva-ap", save_dir, iso, sep = "/"),
                     region = "", 
                     show_progress = FALSE)
  
  
  # Pie plots
  df_pie_wdpa = data_pa %>%
    filter(iso3 == iso) %>%
    dplyr::select(c(iso3, wdpaid, name_pa, status_yr, area_km2)) %>%
    mutate(group_wdpa = case_when(is.na(wdpaid) == FALSE ~ "WDPA",
                             is.na(wdpaid) == TRUE ~ "Not WDPA")) %>%
    group_by(iso3, group_wdpa) %>%
    summarise(n = n()) %>%
    ungroup() %>%
    mutate(n_tot = sum(n),
           freq = round(n/n_tot*100, 1))
  
  df_pie_ie = wdpa_prj %>%
    st_drop_geometry() %>%
    dplyr::select(c(ISO3, WDPAID)) %>%
    mutate(group_ie = case_when(!WDPAID %in% c(wdpaID_afd_ie, wdpaID_afd_no_ie) ~ "Non-funded",
                                WDPAID %in% wdpaID_afd_ie ~ "Funded, analyzed",
                                WDPAID %in% wdpaID_afd_no_ie ~ "Funded, not analyzed")) %>%
    group_by(ISO3, group_ie) %>%
    summarise(n = n()) %>%
    ungroup() %>%
    mutate(n_tot = sum(n),
           freq = round(n/n_tot*100, 1))
  
  ## PAs funded : reported in the WDPAID or not
  pie_wdpa = ggplot(df_pie_wdpa, 
                        aes(x="", y= freq, fill = group_wdpa)) %>%
    + geom_bar(width = 0.5, stat = "identity", color="white") %>%
    + coord_polar("y", start=0) %>%
    + geom_label_repel(aes(x=1.1, label = paste0(round(freq, 1), "% (", n, ")")), 
                       color = "black", 
                       position = position_stack(vjust = 0.55), 
                       size=4, show.legend = FALSE) %>%
    # + geom_label(aes(x=1.4, label = paste0(freq_iucn, "%")), 
    #              color = "white", 
    #              position = position_stack(vjust = 0.7), size=2.5, 
    #              show.legend = FALSE) %>%
    + labs(x = "", y = "",
           title = "Share of PAs funded and reported in the WDPA",
           subtitle = paste("Sample :", sum(df_pie_wdpa$n), "funded protected areas in", country.name)) %>%
    + scale_fill_brewer(name = "", palette = "Greens") %>%
    + theme_void()
  
  ## PAs in the WDPA : analyzed or not
  pie_ie = ggplot(df_pie_ie, 
                    aes(x="", y= freq, fill = group_ie)) %>%
    + geom_bar(width = 0.5, stat = "identity", color="white") %>%
    + coord_polar("y", start=0) %>%
    + geom_label_repel(aes(x=1.1, label = paste0(round(freq, 1), "% (", n, ")")), 
                       color = "black", 
                       position = position_stack(vjust = 0.55), 
                       size=4, show.legend = FALSE) %>%
    # + geom_label(aes(x=1.4, label = paste0(freq_iucn, "%")), 
    #              color = "white", 
    #              position = position_stack(vjust = 0.7), size=2.5, 
    #              show.legend = FALSE) %>%
    + labs(x = "", y = "",
           title = "Share of PAs reported in the WDPA and analyzed",
           subtitle = paste("Sample :", sum(df_pie_ie$n), "funded protected areas in", country.name)) %>%
    + scale_fill_brewer(name = "", palette = "Greens") %>%
    + theme_void()
  
  ##Saving plots
  tmp = paste(tempdir(), "fig", sep = "/")
  ggsave(paste(tmp, paste0("pie_funded_wdpa_", iso, ".png"), sep = "/"),
         plot = pie_wdpa,
         device = "png",
         height = 6, width = 9)
  ggsave(paste(tmp, paste0("pie_wdpa_ie_", iso, ".png"), sep = "/"),
         plot = pie_ie,
         device = "png",
         height = 6, width = 9)
  
  files <- list.files(tmp, full.names = TRUE)
  ##Add each file in the bucket (same foler for every file in the temp)
  for(f in files) 
  {
    cat("Uploading file", paste0("'", f, "'"), "\n")
    aws.s3::put_object(file = f,
                       bucket = paste("projet-afd-eva-ap", save_dir, iso, sep = "/"),
                       region = "", show_progress = TRUE)
  }
  do.call(file.remove, list(list.files(tmp, full.names = TRUE)))
  
  #Append the log 
  cat("#Determining Group IDs and WDPA IDs\n-> OK\n", file = log, append = TRUE)
  
  #Return the output
  list_output = list("grid.param" = grid.param, "is_ok" = TRUE)
  return(list_output)
  
    },
  
  error = function(e)
  {
    #Print the error and append the log
    print(e)
    #Append the log 
    cat(paste("#Determining Group IDs and WDPA IDs\n-> Error :\n", e, "\n"), file = log, append = TRUE)
    #Return string to inform user to skip
    return(list("is_ok" = FALSE))
  },
  
  warning = function(w)
  {
    #Print the warning and append the log
    print(w)
    #Append the log 
    cat(paste("#Determining Group IDs and WDPA IDs\n-> Warning :\n", w, "\n"), file = log, append = TRUE)
    #Return string to inform user to skip
    return(list("is_ok" = TRUE))
  }
  
  )
  
  #Return outputs
  return(output)
  
}



#Building a matching dataframe for the country considered : for each pixel in treated and control groups, the data needed for the analysis are downloaded and the indicators computed. Eventually a dataset is obtained that is ready to enter a matching algorithm
##INPUTS :
### grid.param : a raster representing the gridding of the country with two layers. One for the group each pixel belongs to (funded PA, non-funded PA, potential control, buffer), the other for the WDPAID corresponding to each pixel (0 if not a PA)
### path_tmp : a temporary folder to store figures
### iso : ISO code of the country of interest
### name_output : the name of the matching frame to save
### ext_output : the file extension of the matching to save 
### yr_first : the first year of the period where the analysis takes place
### yr_last : the last year of the period where the analysis takes place
### log : a log file to track progress of the processing
### save_dir : saving directory
##OUTPUTS :
### is_ok : a boolean indicating whether or not an error occured inside the function
##DATA SAVED
### pivot.all : a dataframe with variables of interest (outcome, matching covariates) for all treated and potential control pixels

fn_pre_mf_parallel = function(grid.param, path_tmp, iso, name_output, ext_output, yr_first, yr_last, log, save_dir) 
{
  output = tryCatch(
    
    {
  tic = tic()
  
  print("----Initialize portfolio")
  # Take only potential control (group = 1) and treatment (group = 2) in the country gridding to lower the number of computations to perform
  grid.aoi = grid.param %>%
    filter(group %in% c(1,2))
  # Create a mapme.biodiversity portfolio for the area of interest (aoi). This specifies the period considered and the geospatial units where data are downloaded and indicators computed (here, the treated and control pixels in the country gridding)
  aoi = init_portfolio(grid.aoi,
                       years = yr_first:yr_last,
                       outdir = path_tmp,
                       add_resources = FALSE)
  
  #Extract a dataframe with pixels ID in the grid and the portfolio : useful for latter plotting of matched control and treated units. 
  df_gridID_assetID = aoi %>%
    st_drop_geometry() %>%
    as.data.frame() %>%
    dplyr::select(c(gridID, assetid))
  s3write_using(df_gridID_assetID,
                data.table::fwrite,
                object = paste0(save_dir, "/", iso, "/", "df_gridID_assetID_", iso, ".csv"),
                bucket = "projet-afd-eva-ap",
                opts = list("region" = ""))
  
  print("----Download data")
  # Download Data
  ## Version of Global Forest Cover data to consider
  list_version_gfc = mapme.biodiversity:::.available_gfw_versions() #all versions available
  version_gfc = list_version_gfc[length(list_version_gfc)] #last version considered
  ## Soil characteristics
  dl.soil = get_resources(aoi, 
                          resources = c("soilgrids"), 
                          layers = c("clay"), # resource specific argument
                          depths = c("0-5cm"), # resource specific argument
                          stats = c("mean"))
  ## Accessibility
  dl.travelT = get_resources(aoi, resources = "nelson_et_al",
                             range_traveltime = c("5k_110mio"))
  ## Tree cover evolution on the period
  dl.tree = get_resources(aoi, 
                          resources = c("gfw_treecover", "gfw_lossyear"),
                          vers_treecover = version_gfc,
                          vers_lossyear = version_gfc)
  ## Elevation
  dl.elevation = get_resources(aoi, "nasa_srtm")
  ## Terrain Ruggedness Index
  dl.tri = get_resources(aoi, "nasa_srtm")
  
  print("----Compute indicators")
  #Compute indicators
  
  #Begin multisession : use of parallel computing (computations performed in separate R sessions in background) to speed up the computations of indicators
  # gc : optimize memory management for the background sessions.
  # Multisession with workers = 6 as in mapme.biodiversity tutorial : https://mapme-initiative.github.io/mapme.biodiversity/articles/quickstart.html?q=parall#enabling-parallel-computing
  # Careful to the format of command to call parallel computations here : VALUE TO COMPUTE %<-% {EXPRESSION}.
  plan(multisession, workers = 6, gc = TRUE)
  with_progress({
    get.soil %<-% {calc_indicators(dl.soil,
                                indicators = "soilproperties",
                                stats_soil = c("mean"),
                                engine = "exactextract")} # the "exactextract" engine is chosen as it is the faster one for large rasters (https://tmieno2.github.io/R-as-GIS-for-Economists/extraction-speed-comparison.html)

    get.travelT  %<-% {calc_indicators(dl.travelT,
                                  indicators = "traveltime",
                                  stats_accessibility = c("mean"),  #Note KfW use "median" here, but for no specific reason a priori (mail to Kemmeng Liu, 28/09/2023). Mean is chosen coherently with the other covariates, though we could test in a second time whether this changes anything to the results.
                                  engine = "exactextract")}

    get.tree  %<-% {calc_indicators(dl.tree,
                               indicators = "treecover_area",
                               min_size=0.5, # FAO definition of forest :  Minimum treecover = 10%, minimum size =0.5 hectare (FAO 2020 Global Fores Resources Assessment, https://www.fao.org/3/I8661EN/i8661en.pdf)
                               min_cover=10)}
  
    get.elevation %<-% {calc_indicators(dl.elevation,
                      indicators = "elevation",
                      stats_elevation = c("mean"),
                      engine = "exactextract")}
    
    get.tri %<-% {calc_indicators(dl.tri,
                      indicators = "tri",
                      stats_tri = c("mean"),
                      engine = "exactextract")}
    
    })
  
  print("----Build indicators' datasets")
  #Build indicators' datasets
  ## Transform the output dataframe into a -ore convenient format
  data.soil = unnest(get.soil, soilproperties) %>%
    #mutate(across(c("mean"), \(x) round(x, 3))) %>% # Round numeric columns --> rounding before the matching algorithm is irrelevant to me
    pivot_wider(names_from = c("layer", "depth", "stat"), values_from = "mean") %>%
    rename("clay_0_5cm_mean" = "clay_0-5cm_mean") %>%
    mutate(clay_0_5cm_mean = case_when(is.nan(clay_0_5cm_mean) ~ NA,
                                       TRUE ~ clay_0_5cm_mean))
  
  data.travelT = unnest(get.travelT, traveltime) %>%
    pivot_wider(names_from = "distance", values_from = "minutes_median", names_prefix = "minutes_median_") %>%
    mutate(minutes_median_5k_110mio = case_when(is.nan(minutes_median_5k_110mio) ~ NA,
                                       TRUE ~ minutes_median_5k_110mio))
  
  data.tree = unnest(get.tree, treecover_area) %>%
    drop_na(treecover) %>% #get rid of units with NA values 
    #mutate(across(c("treecover"), \(x) round(x, 3))) %>% # Round numeric columns
    pivot_wider(names_from = "years", values_from = "treecover", names_prefix = "treecover_")
  
  data.tri = unnest(get.tri, tri) %>%
    mutate(tri_mean = case_when(is.nan(tri_mean) ~ NA,
                                TRUE ~ tri_mean))
  
  data.elevation = unnest(get.elevation, elevation) %>%
    mutate(elevation_mean = case_when(is.nan(elevation_mean) ~ NA,
                                TRUE ~ elevation_mean))
  
  ## End parallel plan : close parallel sessions, so must be done once indicators' datasets are built
  plan(sequential)
  
  # The calculation of tree loss area is performed at dataframe base
  # Get the column names of tree cover time series
  colnames_tree = names(data.tree)[startsWith(names(data.tree), "treecover")]
  # Drop the first year
  dropFirst = tail(colnames_tree, -1)
  # Drop the last year
  dropLast = head(colnames_tree, -1)
  # Set list of new column names for tree loss time series
  colnames_loss = dropFirst %>% str_split(., "_")
  
  # Add new columns: treeloss_tn = treecover_tn - treecover_t(n-1)  
  for (i in 1:length(dropFirst)) 
  {
    new_colname = paste0("treeloss_", colnames_loss[[i]][2]) 
    data.tree[[new_colname]] = data.tree[[dropFirst[i]]] - data.tree[[dropLast[i]]]
  }
  
  print("----Export Matching Frame")
  # Remove "geometry" column from dataframes
  df.tree = data.tree %>% mutate(x = NULL) %>% as.data.frame()
  df.travelT = data.travelT %>% mutate(x = NULL) %>% as.data.frame()
  df.soil = data.soil %>% mutate(x = NULL) %>% as.data.frame()
  df.elevation = data.elevation %>% mutate(x = NULL) %>% as.data.frame()
  df.tri = data.tri %>% mutate(x=NULL) %>% as.data.frame()
  
  # Make a dataframe containing only "assetid" and geometry
  # Use data.soil instead of data.tree, as some pixels are removed in data.tree (NA values from get.tree)
  df.geom = data.soil[, c("assetid", "x")] %>% as.data.frame() 
  
  # Merge all output dataframes 
  pivot.all = Reduce(dplyr::full_join, list(df.travelT, df.soil, df.tree, df.elevation, df.tri, df.geom)) %>%
    st_as_sf()

  # Make column Group ID and WDPA ID have data type "integer"
  pivot.all$group = as.integer(pivot.all$group)
  pivot.all$wdpaid = as.integer(pivot.all$wdpaid)

  # Save this matching dataframe
  name_save = paste0(name_output, "_", iso, ext_output)
  s3write_using(pivot.all,
                sf::st_write,
                object = paste0(save_dir, "/", iso, "/", name_save),
                bucket = "projet-afd-eva-ap",
                opts = list("region" = ""))
  
  #Removing files in the temporary folder
  do.call(file.remove, list(list.files(tmp_pre, include.dirs = F, full.names = T, recursive = T)))
  
  #End timer
  toc = toc()
  
  #Append the log
  cat(paste("#Calculating outcome and other covariates\n-> OK :", toc$callback_msg, "\n\n"), file = log, append = TRUE)

  #Return the output
  return(list("is_ok" = TRUE))
  
    },
  
  error = function(e)
  {
    #Print the error and append the log
    print(e)
    #Append the log 
    cat(paste("#Calculating outcome and other covariates\n-> Error :\n", e, "\n\n"), file = log, append = TRUE)
    #Return string to inform user to skip
    return(list("is_ok" = FALSE))
  }
  
  # warning = function(w)
  # {
  #   #Print the warning and append the log
  #   print(w)
  #   #Append the log 
  #   cat(paste("#Calculating outcome and other covariates\n-> Warning :\n", w, "\n"), file = log, append = TRUE)
  #   #Return string to inform user to skip
  #   return(list("is_ok" = TRUE))
  # }
  
  )
  
  return(output)
}


#####
###Post-processing
#####


#Load the matching dataframe obtained during pre-processing
##INPUTS :
### iso : the ISO code of the country considered
### name_input : name of the file to import
### ext_output : extension fo the file to import
### yr_min : the minimum treatment year to be considered in the analysis. As some matching covariates are defined with pre-treatment data (e.g average tree cover loss before treatment), this minimal year is greater than the first year in the period considered
### log : a log file to track progress of the processing
### save_dir : saving directory
##OUTPUTS :
### mf : matching dataframe. More precisely, it gives for each observation units in a country values of different covariates to perform matching.
### is_ok : a boolean indicating whether or not an error occured inside the function
##DATA SAVED
### The list of PAs in the matching frame, characterized by their WDPAID. Useful to loop over each PAs we want to analyze in a given country
fn_post_load_mf = function(iso, yr_min, name_input, ext_input, log, save_dir)
{
  output = tryCatch(
    
    {
      
  #Load the matching dataframe
  object = paste(save_dir, iso, paste0(name_input, "_", iso, ext_input), sep = "/")
  mf = s3read_using(sf::st_read,
                      bucket = "projet-afd-eva-ap",
                      object = object,
                      opts = list("region" = "")) 
  
  #Subset to control and treatment units with year of treatment >= yr_min
  mf = mf %>%
    filter(group==1 | group==2) %>%
    #Remove observations with NA values only for covariates :
    ## except for creation year, funding years, geographical location, country ISO and name, pixel resolution which are NA for control units
    drop_na(-c(status_yr, year_funding_first, year_funding_all, region_afd, region, sub_region, iso3, country_en, res_m)) #%>%
     #filter(status_yr >= yr_min | is.na(status_yr))
  
  #Write the list of PAs matched
  list_pa = mf %>%
    st_drop_geometry() %>%
    as.data.frame() %>%
    dplyr::select(c(region_afd, region, sub_region, country_en, iso3, wdpaid, status_yr, year_funding_first, year_funding_all)) %>%
    mutate(iso3 = iso, .before = "wdpaid") %>%
    filter(wdpaid != 0) %>%
    group_by(wdpaid) %>%
    slice(1) %>%
    ungroup()
  
  s3write_using(list_pa,
                data.table::fwrite,
                bucket = "projet-afd-eva-ap",
                object = paste(save_dir, iso, paste0("list_pa_matched_", iso, ".csv"), sep = "/"),
                opts = list("region" = ""))
  
  #Append the log
  cat("Loading the matching frame -> OK\n", file = log, append = TRUE)
  
  #Return output
  return(list("mf" = mf, "is_ok" = TRUE))
  
    },
  
  error = function(e)
  {
    print(e)
    cat(paste("Error in loading the matching frame :\n", e, "\n"), file = log, append = TRUE)
    return(list("is_ok" = FALSE))
  }
  
  # warning = function(w)
  # {
  #   #Print the warning and append the log
  #   print(w)
  #   #Append the log 
  #   cat(paste("Warining while loading the matching frame :\n", w, "\n"), file = log, append = TRUE)
  #   #Return string to inform user to skip
  #   return(list("is_ok" = TRUE))
  # }
  
  
  )
  
  return(output)
}


#Compute average forest loss before PA creation, and add it to the matching frame as a covariate
##INPUTS : 
### mf : the matching dataframe
### colname.flAvg : name of the average forest loss variable
### log : a log file to track progress of the processing
## OUTPUTS :
### mf : matching frame with the new covariate
### is_ok : a boolean indicating whether or not an error occured inside the function

fn_post_avgLoss_prefund = function(mf, colname.flAvg, log)
{
  
  output = tryCatch(
    
    {
      
  #Extract treatment year
  treatment.year = mf %>% 
    filter(group == 2) %>% 
    slice(1)
  treatment.year = treatment.year$status_yr
  
  #Extract first year treeloss is computed
  ##Select cols with "treeloss" in mf, drop geometry, replace "treeloss_" by "", convert to num and take min
  treeloss.ini.year = mf[grepl("treeloss", names(mf))] %>%
    st_drop_geometry() %>%
    names() %>%
    gsub(paste0("treeloss", "_"), "", .) %>%
    as.numeric() %>%
    min()
  
  #Define period to compute average loss
  ##If 5 pre-treatment periods are available at least, then average pre-treatment deforestation is computed on this 5 years range
  ## If less than 5 are available, compute on this restricted period
  ## Note that by construction, treatment.year >= treeloss.ini.year +1 (as yr_min = yr_first+2 in the parameters)
  if((treatment.year-treeloss.ini.year) >=5)
  {yr_start = (treatment.year)-5
  yr_end = (treatment.year)-1} else if((treatment.year-treeloss.ini.year <5) & (treatment.year-treeloss.ini.year >0))
  {yr_start = treeloss.ini.year
  yr_end = (treatment.year)-1} 
  #Transform it in variable suffix
  var_start = yr_start - 2000
  var_end = yr_end - 2000
  #Select only relevant variables
  df_fl = mf[grepl("treeloss", names(mf))][var_start:var_end] %>% 
    st_drop_geometry()
  #Compute average loss for each pixel and store it in mf. Also add the start and end years of pre-treatment period where average loss is computed.
  mf$avgLoss_pre_fund = round(rowMeans(df_fl), 2)
  mf$start_pre_fund = yr_start
  mf$end_pre_fund = yr_end
  #Remove NA values
  mf = mf %>% drop_na(avgLoss_pre_fund)
  
  #Append the log
  cat("#Add average pre-treatment treecover loss\n-> OK\n", file = log, append = TRUE)
  
  #Return output
  return(list("mf" = mf, "is_ok" = TRUE))
  
    },
  
  error = function(e)
  {
    print(e)
    cat(paste("#Add average pre-treatment treecover loss\n-> Error :\n", e, "\n"), file = log, append = TRUE)
    return(list("is_ok" = FALSE))
  }
  
  # warning = function(w)
  # {
  #   #Print the warning and append the log
  #   print(w)
  #   #Append the log 
  #   cat(paste("#Add average pre-treatment treecover loss\n-> Warning :\n", w, "\n"), file = log, append = TRUE)
  #   #Return string to inform user to skip
  #   return(list("is_ok" = TRUE))
  # }
  
  )
  
  return(output)
}


#Perform matching of treated and potential control units. 
##INPUTS :
### mf : the matching dataframe
### iso : the ISO code of the country considered
### dummy_int : should we consider the interaction of variables for matching ? Is recommended generally speaking (https://cran.r-project.org/web/packages/MatchIt/vignettes/assessing-balance.html). When using CEM matching, variables are binned then exact matching is performed on binned values. As a first approximation we can argue that if two units have the same binned values for two variables, then they likely have the same binned interaction value. It is not necessarily true though, as binned(A)*binned(B) can be different from binned(A*B).  
### match_method : the matching method to use. See https://cran.r-project.org/web/packages/MatchIt/vignettes/matching-methods.html for a list of matching methods we can use with the MatchIT package 
### cutoff_method : the method to use for automatic histogram binning of the variables. See Iacus, King and Porro 2011 (https://gking.harvard.edu/files/political_analysis-2011-iacus-pan_mpr013.pdf), 5.5.1, or MathIT documentation. "Sturges" tend to have the best outcomes (number of matched units) in our case (Antoine Vuillot, 28/09/2023)
### is_k2k : boolean. Should we use k2k matching ? If yes, each treated unit is eventually matched with a single control. For CEM matching, a treated unit is potentially associated with more than one control unit (exact matching on binned variables), and then the "closest' one is chosen with a metric defined in k2k_method
### k2k_method : metric to use to choose the closest control among the control units matched with a treated unit in CEM matching.
### th_mean :the maximum acceptable value for absolute standardized mean difference of covariates between matched treated and control units. Typically 0.1 (https://cran.r-project.org/web/packages/MatchIt/vignettes/assessing-balance.html) or 0.25 in conservation literature (e.g https://conbio.onlinelibrary.wiley.com/doi/abs/10.1111/cobi.13728) 
### th_var_min, th_var_max : the range of acceptable value for covariate variance ratio between matched treated and control units. Typicall 0.5 and 2, respectively (https://cran.r-project.org/web/packages/MatchIt/vignettes/assessing-balance.html)
### colname.travelTime, colname.clayContent, colname.elevation, colname.tri, colname.fcIni, colname.flAvg : name of the matching covariates
### log : a log file to track progress of the processing
##OUTPUTS : 
### out.cem : an object with all information on matching (parameters, results, etc.)
### df.cov.m : for each matching covariate, statistics to assess the quality of the match
### is_ok : a boolean indicating whether or not an error occured inside the function
##NOTES
### The matching method chosen is CEM though other exists. For a presentation of the different matching algorithms, see https://cran.r-project.org/web/packages/MatchIt/vignettes/matching-methods.html
fn_post_match_auto = function(mf,
                              iso,
                              dummy_int,
                              match_method,
                              cutoff_method,
                              is_k2k,
                              k2k_method,
                              th_mean, 
                              th_var_min, th_var_max,
                              colname.travelTime, colname.clayContent, colname.elevation, colname.tri, colname.fcIni, colname.flAvg,
                              log)
{

  #Append the log file : CEM step
  cat("#Run Coarsened Exact Matching\n", 
      file = log, append = TRUE)
  
  ## Matching handling errors due to absence of matching
  output = 
    tryCatch(
    {
      # Formula
      formula = eval(bquote(group ~ .(as.name(colname.travelTime)) 
                            + .(as.name(colname.clayContent))  
                            +  .(as.name(colname.fcIni)) 
                            + .(as.name(colname.flAvg))
                            + .(as.name(colname.tri))
                            + .(as.name(colname.elevation))))
      
      #Try to perform matching
      out.cem = matchit(formula,
                        data = mf,
                        method = match_method,
                        cutpoints = cutoff_method,
                        k2k = is_k2k,
                        k2k.method = k2k_method)
      
      # Then the performance of the matching is assessed, based on https://cran.r-project.org/web/packages/MatchIt/vignettes/assessing-balance.html
      ## Covariate balance : standardized mean difference and variance ratio
      ## For both tests and the joint one, a dummy variable is defined, with value TRUE is the test is passed
      df.cov.m = summary(out.cem, interactions = dummy_int)$sum.matched %>%
        as.data.frame() %>%
        clean_names() %>%
        mutate(is_var_ok = var_ratio < th_var_max & var_ratio > th_var_min, #Check variance ratio between treated and controls
               is_mean_ok = abs(std_mean_diff) < th_mean, #Check absolute standardized mean difference
               is_bal_ok = as.logical(is_var_ok*is_mean_ok), #Binary : TRUE if both variance and mean difference check pass, 0 if at least one does not
               .after = "std_mean_diff")
      
      #Add a warning if covariate balance tests are not passed
      if(sum(df.cov.m$is_bal_ok) < nrow(df.cov.m) | is.na(sum(df.cov.m$is_bal_ok)) == TRUE)
      {
        message("Matched control and treated units are not balanced enough. Increase sample size, turn to less restrictive tests or visually check balance.")
        cat("-> Careful : matched control and treated units are not balanced enough. Increase sample size, turn to less restrictive tests or visually check balance.\n", 
            file = log, append = TRUE)
      }
      
      #Append the log : note the step has already been appended at the beginning of the function
      cat("-> OK\n", file = log, append = TRUE)
      
      return(list("out.cem" = out.cem, "df.cov.m" = df.cov.m, "is_ok" = TRUE))
      
    },
    
    error=function(e)
    {
      print(e)
      cat(paste("-> Error :\n", e, "\n"), file = log, append = TRUE)
      return(list("is_ok" = FALSE))
    },
    
    warning = function(w)
    {
      #Print the warning and append the log
      #Append the log 
      cat(paste("-> Warning :\n", w, "\n"),
          file = log, append = TRUE)
      return(list("is_ok" = FALSE)) #Here warning comes from an absence of matching : thus must skip to next country
    }
    
  )
  
  return(output)
  
}
    


#Plot covariates balance (plots and summary table)
## INPUTS :
### out.cem : list of results from the CEM matching
### mf : the matching dataframe
### colname.travelTime, colname.clayContent, colname.elevation, colname.tri, colname.fcIni, colname.flAvg : name of the matching covariates
### iso : ISO code of the country considered
### path_tmp : temporary folder to store figures
### wdpaid : the WDPA ID of the protected area considered
### log : a log file to track progress of the processing
### save_dir : saving directory
##OUTPUTS :
### is_ok : a boolean indicating whether or not an error occured inside the function
## DATA SAVED :
### A covariate love plot
### A table with number of treated and control units, before and after matching
### A table with statistics on matched control and treated units 
### A table with statistics on unmatched control and treated units,
fn_post_covbal = function(out.cem, mf, 
                          colname.travelTime, colname.clayContent, colname.fcIni, colname.flAvg, colname.tri, colname.elevation, 
                          iso, path_tmp, wdpaid, log,
                          save_dir)
{
  
  output = tryCatch(
    
    {
      
  #Save summary table from matching
  smry_cem = summary(out.cem)
  tbl_cem_nn = smry_cem$nn
  tbl_cem_m = smry_cem$sum.matched
  tbl_cem_all = smry_cem$sum.all
  
  #Extract country name
  country.name = mf %>% 
    filter(group == 2) %>% 
    slice(1)
  country.name = country.name$country_en
  
  #Extract start and end years of pre-treatment period where average loss is computed
  year.start.prefund = mf %>%
    filter(group == 2) %>% 
    slice(1)
  year.start.prefund = year.start.prefund$start_pre_fund
  
  year.end.prefund = mf %>%
    filter(group == 2) %>% 
    slice(1)
  year.end.prefund = year.end.prefund$end_pre_fund
  
  #Plot covariate balance
  colname.flAvg.new = paste0("Avg. Annual Forest \n Loss ",  year.start.prefund, "-", year.end.prefund)
  c_name = data.frame(old = c(colname.travelTime, colname.clayContent, colname.tri, colname.elevation,
                              colname.fcIni, colname.flAvg),
                      new = c("Accessibility", "Clay Content", "Terrain Ruggedness Index (TRI)", "Elevation (m)", "Forest Cover in 2000",
                              colname.flAvg.new))
  
  # Refer to cobalt::love.plot()
  # https://cloud.r-project.org/web/packages/cobalt/vignettes/cobalt.html#love.plot
  fig_covbal = love.plot(out.cem, 
                       binary = "std", 
                       abs = TRUE,
                       #thresholds = c(m = .1),
                       var.order = "unadjusted",
                       var.names = c_name,
                       title = paste0("Covariate balance for WDPA ID ", wdpaid, " in ", country.name),
                       sample.names = c("Discarded", "Selected"),
                       wrap = 25 # at how many characters does axis label break to new line
  )
  # Finetune Layouts using ggplot
  fig_covbal + 
    geom_vline(aes(xintercept=0.1, linetype="Acceptable \n Balance \n (x=0.1)"), color=c("#2ecc71"), linewidth=0.35) +
    theme_bw() +
    theme(
      plot.title = element_text(family="Arial Black", size=16, hjust=0.5),
      
      legend.title = element_blank(),
      legend.text=element_text(size=14),
      legend.spacing.x = unit(0.5, 'cm'),
      legend.spacing.y = unit(0.75, 'cm'),
      
      axis.text.x = element_text(angle = 20, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=12),
      axis.title=element_text(size=14),
      axis.title.y = element_text(margin = margin(unit = 'cm', r = 0.5)),
      axis.title.x = element_text(margin = margin(unit = 'cm', t = 0.5)),
      
      panel.grid.major.x = element_line(color = 'grey', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey', linewidth = 0.3, linetype = 2)
    ) + guides(linetype = guide_legend(override.aes = list(color = "#2ecc71"))) # Add legend for geom_vline
  

  #Saving files
  
  ggsave(paste0(path_tmp, "/CovBal/fig_covbal", "_", iso, "_", wdpaid, ".png"),
         plot = fig_covbal,
         device = "png",
         height = 6, width = 9)
  print(xtable(tbl_cem_nn, type = "latex"),
        file = paste0(path_tmp, "/CovBal/tbl_cem_nn", "_", iso, "_", wdpaid, ".tex"))
  print(xtable(tbl_cem_m, type = "latex"),
        file = paste0(path_tmp, "/CovBal/tbl_cem_m", "_", iso, "_", wdpaid, ".tex"))
  print(xtable(tbl_cem_all, type = "latex"),
        file = paste0(path_tmp, "/CovBal/tbl_cem_all", "_", iso, "_", wdpaid, ".tex"))
  
  #Export to S3 storage
  ##List of files to save in the temp folder
  files <- list.files(paste(path_tmp, "CovBal", sep = "/"), full.names = TRUE)
  ##Add each file in the bucket (same foler for every file in the temp)
  for(f in files) 
  {
    cat("Uploading file", paste0("'", f, "'"), "\n")
    aws.s3::put_object(file = f, 
                       bucket = paste("projet-afd-eva-ap", save_dir, iso, wdpaid, sep = "/"),
                       region = "", show_progress = TRUE)
  }
  do.call(file.remove, list(list.files(paste(path_tmp, "CovBal", sep = "/"), full.names = TRUE)))
  
  #Append the log
  cat("#Plot covariates balance\n->OK\n", file = log, append = TRUE)
  
  return(list("is_ok" = TRUE))
  
    },
  
  error=function(e)
  {
    print(e)
    cat(paste("#Plot covariates balance\n-> Error :\n", e, "\n"), file = log, append = TRUE)
    return(list("is_ok" = FALSE))
  }
  
  # warning = function(w)
  # {
  #   #Print the warning and append the log
  #   print(w)
  #   #Append the log 
  #   cat(paste("#Plot covariates balance\n-> Warning :\n", w, "\n"), file = log, append = TRUE)
  #   #Return string to inform user to skip
  #   return(list("is_ok" = TRUE))
  # }
  
  )
  
  return(output)

}


#Density plots of covariates for control and treatment units, before and after matching
## INPUTS :
### out.cem : list of results from the CEM matching
### mf : the matching dataframe
### colname.travelTime, colname.clayContent, colname.elevation, colname.tri, colname.fcIni, colname.flAvg : name of the matching covariates
### iso : ISO code of the country considered
### path_tmp : temporary folder to store figures
### wdpaid : the WDPA ID of the protected area considered
### log : a log file to track progress of the processing
### save_dir : saving directory
## OUTPUTS :
### is_ok : a boolean indicating whether or not an error occured inside the function
## DATA SAVED :
### Density plots of the matching covariates considered, for matched treated and control units
fn_post_plot_density = function(out.cem, mf, 
                                colname.travelTime, colname.clayContent, colname.fcIni, colname.flAvg, colname.tri, colname.elevation, 
                                iso, path_tmp, wdpaid, log, save_dir)
{
  output = tryCatch(
    
    {
      
  # Define Facet Labels
  fnl = c(`Unadjusted Sample` = "Before Matching",
          `Adjusted Sample` = "After Matching")
  
  #Extract country name
  country.name = mf %>% 
    filter(group == 2) %>% 
    slice(1)
  country.name = country.name$country_en
  
  #Define plots
  ## Density plot for Travel Time
  fig_travel = bal.plot(out.cem, 
                      var.name = colname.travelTime,
                      #sample.names = c("Control", "Treatment"),
                      which = "both") +
    facet_wrap(.~which, labeller = as_labeller(fnl)) +
    #scale_fill_viridis(discrete = T) +
    scale_fill_manual(labels = c("Control", "Treatment"), values = c("#f5b041","#5dade2")) +
    labs(title = "Distributional balance for accessibility",
         subtitle = paste0("Protected area in ", country.name, ", WDPAID ", wdpaid),
         x = "Accessibility (min)",
         fill = "Group") +
    theme_bw() +
    theme(
      plot.title = element_text(family="Arial Black", size=16, hjust = 0),
      
      legend.title = element_blank(),
      legend.text=element_text(size=14),
      legend.spacing.x = unit(0.5, 'cm'),
      legend.spacing.y = unit(0.75, 'cm'),
      
      axis.text=element_text(size=12),
      axis.title=element_text(size=14),
      axis.title.y = element_text(margin = margin(unit = 'cm', r = 0.5)),
      axis.title.x = element_text(margin = margin(unit = 'cm', t = 0.5)),
      
      strip.text.x = element_text(size = 12) # Facet Label
    )
  
  ## Density plot for Clay Content
  fig_clay = bal.plot(out.cem, 
                    var.name = colname.clayContent,
                    which = "both") +
    facet_wrap(.~which, labeller = as_labeller(fnl)) +
    #scale_fill_viridis(discrete = T) +
    scale_fill_manual(labels = c("Control", "Treatment"), values = c("#f5b041","#5dade2")) +
    labs(title = "Distributional balance for clay content",
         subtitle = paste0("Protected area in ", country.name, ", WDPAID ", wdpaid),
         x = "Clay content at 0~20cm soil depth (%)",
         fill = "Group") +
    theme_bw() +
    theme(
      plot.title = element_text(family="Arial Black", size=16, hjust=0),
      
      legend.title = element_blank(),
      legend.text=element_text(size=14),
      legend.spacing.x = unit(0.5, 'cm'),
      legend.spacing.y = unit(0.75, 'cm'),
      
      axis.text=element_text(size=12),
      axis.title=element_text(size=14),
      axis.title.y = element_text(margin = margin(unit = 'cm', r = 0.5)),
      axis.title.x = element_text(margin = margin(unit = 'cm', t = 0.5)),
      
      strip.text.x = element_text(size = 12) # Facet Label
    )
  
  ## Density plot for Elevation
  fig_elevation = bal.plot(out.cem,
                      var.name = colname.elevation,
                      which = "both") +
      facet_wrap(.~which, labeller = as_labeller(fnl)) +
      #scale_fill_viridis(discrete = T) +
      scale_fill_manual(labels = c("Control", "Treatment"), values = c("#f5b041","#5dade2")) +
      labs(title = "Distributional balance for elevation",
           subtitle = paste0("Protected area in ", country.name, ", WDPAID ", wdpaid),
           x = "Elevation (m)",
           fill = "Group") +
      theme_bw() +
      theme(
          plot.title = element_text(family="Arial Black", size=16, hjust=0),
          legend.title = element_blank(),
          legend.text=element_text(size=14),
          legend.spacing.x = unit(0.5, 'cm'),
          legend.spacing.y = unit(0.75, 'cm'),

          axis.text=element_text(size=12),
          axis.title=element_text(size=14),
          axis.title.y = element_text(margin = margin(unit = 'cm', r = 0.5)),
          axis.title.x = element_text(margin = margin(unit = 'cm', t = 0.5)),

          strip.text.x = element_text(size = 12) # Facet Label
      )
  
  ## Density plot for TRI
  fig_tri = bal.plot(out.cem,
                      var.name = colname.tri,
                      which = "both") +
      facet_wrap(.~which, labeller = as_labeller(fnl)) +
      #scale_fill_viridis(discrete = T) +
      scale_fill_manual(labels = c("Control", "Treatment"), values = c("#f5b041","#5dade2")) +
      labs(title = "Distributional balance for Terrain Ruggedness Index (TRI)",
          subtitle = paste0("Protected area in ", country.name, ", WDPAID ", wdpaid),
           x = "TRI",
           fill = "Group") +
      theme_bw() +
      theme(
          plot.title = element_text(family="Arial Black", size=16, hjust=0),

          legend.title = element_blank(),
          legend.text=element_text(size=14),
          legend.spacing.x = unit(0.5, 'cm'),
          legend.spacing.y = unit(0.75, 'cm'),

          axis.text=element_text(size=12),
          axis.title=element_text(size=14),
          axis.title.y = element_text(margin = margin(unit = 'cm', r = 0.5)),
          axis.title.x = element_text(margin = margin(unit = 'cm', t = 0.5)),

          strip.text.x = element_text(size = 12) # Facet Label
      )
  
  ## Density plot for covariate "forest cover 2000"
  fig_fc = bal.plot(out.cem, 
                  var.name = colname.fcIni,
                  which = "both") +
    facet_wrap(.~which, labeller = as_labeller(fnl)) +
    scale_fill_manual(labels = c("Control", "Treatment"), values = c("#f5b041","#5dade2")) +
    # scale_x_continuous(trans = "log10") +
    labs(title = "Distributional balance for forest cover in 2000",
         subtitle = paste0("Protected area in ", country.name, ", WDPAID ", wdpaid),
         x = "Forest cover (ha)",
         fill = "Group") +
    theme_bw() +
    theme(
      plot.title = element_text(family="Arial Black", size=16, hjust=0),
      
      legend.title = element_blank(),
      legend.text=element_text(size=14),
      legend.spacing.x = unit(0.5, 'cm'),
      legend.spacing.y = unit(0.75, 'cm'),
      
      axis.text=element_text(size=12),
      axis.title=element_text(size=14),
      axis.title.y = element_text(margin = margin(unit = 'cm', r = 0.5)),
      axis.title.x = element_text(margin = margin(unit = 'cm', t = 0.5)),
      
      strip.text.x = element_text(size = 12) # Facet Label
    )
  
  ## Density plot for covariate "avg. annual forest loss prior funding"
  fig_fl = bal.plot(out.cem, 
                  var.name = colname.flAvg,
                  which = "both") +
    facet_wrap(.~which, labeller = as_labeller(fnl)) +
    #scale_fill_viridis(discrete = T) +
    scale_fill_manual(labels = c("Control", "Treatment"), values = c("#f5b041","#5dade2")) +
    labs(title = "Distributional balance for average pre-treatment forest loss",
         subtitle = paste0("Protected area in ", country.name, ", WDPAID ", wdpaid),
         x = "Forest loss (%)",
         fill = "Group") +
    theme_bw() +
    theme(
      plot.title = element_text(family="Arial Black", size=16, hjust=0),
      legend.title = element_blank(),
      legend.text=element_text(size=14),
      legend.spacing.x = unit(0.5, 'cm'),
      legend.spacing.y = unit(0.75, 'cm'),
      
      axis.text=element_text(size=12),
      axis.title=element_text(size=14),
      axis.title.y = element_text(margin = margin(unit = 'cm', r = 0.5)),
      axis.title.x = element_text(margin = margin(unit = 'cm', t = 0.5)),
      
      strip.text.x = element_text(size = 12) # Facet Label
    )
  
  #Saving plots
  
  tmp = paste(tempdir(), "fig", sep = "/")
  ggsave(paste(tmp, paste0("fig_travel_dplot_", iso, "_", wdpaid, ".png"), sep = "/"),
         plot = fig_travel,
         device = "png",
         height = 6, width = 9)
  ggsave(paste(tmp, paste0("fig_clay_dplot_", iso, "_", wdpaid, ".png"), sep = "/"),
         plot = fig_clay,
         device = "png",
         height = 6, width = 9)
  ggsave(paste(tmp, paste0("fig_elevation_dplot_", iso, "_", wdpaid, ".png"), sep = "/"),
         plot = fig_elevation,
         device = "png",
         height = 6, width = 9)
  ggsave(paste(tmp, paste0("fig_tri_dplot_", iso, "_", wdpaid, ".png"), sep = "/"),
         plot = fig_tri,
         device = "png",
         height = 6, width = 9)
  ggsave(paste(tmp, paste0("fig_fc_dplot_", iso, "_", wdpaid, ".png"), sep = "/"),
         plot = fig_fc,
         device = "png",
         height = 6, width = 9)
  ggsave(paste(tmp, paste0("fig_fl_dplot_", iso, "_", wdpaid, ".png"), sep = "/"),
         plot = fig_fl,
         device = "png",
         height = 6, width = 9)
  
  files <- list.files(tmp, full.names = TRUE)
  ##Add each file in the bucket (same foler for every file in the temp)
  for(f in files) 
  {
    cat("Uploading file", paste0("'", f, "'"), "\n")
    aws.s3::put_object(file = f, 
                       bucket = paste("projet-afd-eva-ap", save_dir, iso, wdpaid, sep = "/"),
                       region = "", show_progress = TRUE)
  }
  do.call(file.remove, list(list.files(tmp, full.names = TRUE)))
  
  #Append the log
  cat("#Plot covariates density\n->OK\n", file = log, append = TRUE)
  
  return(list("is_ok" = TRUE))
  
    },
  
  error=function(e)
  {
    print(e)
    cat(paste("#Plot covariates density\n-> Error :\n", e, "\n"), file = log, append = TRUE)
    return(list("is_ok" = FALSE))
  }
  
  # warning = function(w)
  # {
  #   #Print the warning and append the log
  #   print(w)
  #   #Append the log 
  #   cat(paste("#Plot covariates density\n-> Warning :\n", w, "\n"), file = log, append = TRUE)
  #   #Return string to inform user to skip
  #   return(list("is_ok" = TRUE))
  # }
  
  )
  
  return(output)
  
}


#Define panel datasets (long, wide format) for control and treatment observation units, before and after matching.
## INPUTS :
### out.cem : list of results from the CEM matching
### mf : the matching dataframe
### ext_output : extension fo the file to import
### iso : ISO code of the country considered
### wdpaid : the WDPA ID of the protected area considered
### log : a log file to track progress of the processing
### save_dir : saving directory
## OUTPUTS :
### a list of dataframes : (un)matched.wide/long. They contain covariates and outcomes for treatment and control units, before and after matching, in a wide or long format
### is_ok : a boolean indicating whether or not an error occured inside the function
## DATA SAVED
### (un)matched.wide/long dataframes. They contain covariates and outcomes for treatment and control units, before and after matching, in a wide or long format

fn_post_panel = function(out.cem, mf, ext_output, wdpaid, iso, log, save_dir)
{
  
  output = tryCatch(
    
    {
      
  # Convert dataframe of matched objects to pivot wide form
  matched.wide = match.data(object=out.cem, data=mf)
  
  # Pivot Wide ==> Pivot Long
  matched.long = matched.wide %>%
    dplyr::select(c(region_afd, region, sub_region, country_en, iso3, group, wdpaid, status_yr, year_funding_first, year_funding_all, assetid, weights, starts_with("treecover"), res_m)) %>%
    pivot_longer(cols = c(starts_with("treecover")),
                 names_to = c("var", "year"),
                 names_sep = "_",
                 values_to = "fc_ha")
  
  # Pivot wide Dataframe of un-matched objects
  unmatched.wide = mf
  
  # Pivot Wide ==> Pivot Long
  unmatched.long = unmatched.wide %>%
    dplyr::select(c(region_afd, region, sub_region, iso3, country_en, group, wdpaid, status_yr, year_funding_first, year_funding_all, assetid, starts_with(treecover), res_m)) %>%
    pivot_longer(cols = c(starts_with(treecover)),
                 names_to = c("var", "year"),
                 names_sep = "_",
                 values_to = "fc_ha")
  
  #Save the dataframes
  s3write_using(matched.wide,
                sf::st_write,
                object = paste0(save_dir, "/", iso, "/", wdpaid, "/", paste0("matched_wide", "_", iso, "_", wdpaid, ext_output)),
                bucket = "projet-afd-eva-ap",
                opts = list("region" = ""))
  s3write_using(unmatched.wide,
                sf::st_write,
                object = paste0(save_dir, "/", iso, "/", wdpaid, "/", paste0("unmatched_wide", "_", iso, "_", wdpaid, ext_output)),
                bucket = "projet-afd-eva-ap",
                opts = list("region" = ""))
  s3write_using(matched.long,
                sf::st_write,
                object = paste0(save_dir, "/", iso, "/", wdpaid, "/", paste0("matched_long", "_", iso, "_", wdpaid, ext_output)),
                bucket = "projet-afd-eva-ap",
                opts = list("region" = ""))
  s3write_using(unmatched.long,
                sf::st_write,
                object = paste0(save_dir, "/", iso, "/", wdpaid, "/", paste0("unmatched_long", "_", iso, "_", wdpaid, ext_output)),
                bucket = "projet-afd-eva-ap",
                opts = list("region" = ""))
  
  #Append the log
  cat("#Panelize dataframe\n-> OK\n", file = log, append = TRUE)
  
  #Return outputs
  list_output = list("matched.wide" = matched.wide, "matched.long" = matched.long,
                     "unmatched.wide" = unmatched.wide, "unmatched.long" = unmatched.long,
                     "is_ok" = TRUE)
  return(list_output)
  
    },
  
  error=function(e)
{
  print(e)
  cat(paste("#Panelize dataframe\n-> Error :\n", e, "\n"), file = log, append = TRUE)
  return(list("is_ok" = FALSE))
}

# warning = function(w)
# {
#   #Print the warning and append the log
#   print(w)
#   #Append the log 
#   cat(paste("#Panelize dataframe\n-> Warning :\n", w, "\n"), file = log, append = TRUE)
#   #Return string to inform user to skip
#   return(list("is_ok" = TRUE))
# }

  )
  
  return(output)
  
}   

#Plot the average trend of control and treated units in a given country, before and after the matching
## INPUTS :
### (un)matched.long : dataframe with covariates and outcomes for each treatment and control unit, before and after matching, in a long format (one row : pixel+year)
### mf : the matching dataframe
### data_pa : dataframe with information on each PA considered in the analysis
### iso : ISO code of the country considered
### wdpaid : the WDPA ID of the protected area considered
### log : a log file to track progress of the processing
### save_dir : saving directory
## OUTPUTS : 
### is_ok : a boolean indicating whether or not an error occured inside the function
## DATA SAVED :
### Evolution of forest cover in a treated and control pixel on average, before and after matching
### Same for total forest cover (pixel*# of pixels in the PA)
### Cumulated deforestation relative to 2000 forest cover, in treated and control pixels, before and after matching
fn_post_plot_trend = function(matched.long, unmatched.long, mf, data_pa, iso, wdpaid, log, save_dir) 
{
    
  output = tryCatch(
    
    {
      #First extract some relevant information
      #Extract spatial resolution of pixels res_m and define pixel area in ha
      res_m = unique(mf$res_m)
      res_ha = res_m^2*1e-4
        
      #Extract treatment year
      treatment.year = mf %>% 
        filter(group == 2) %>% 
        slice(1)
      treatment.year = treatment.year$status_yr
      
      #Extract funding years
      funding.years = mf %>% 
        filter(group == 2) %>% 
        slice(1)
      funding.years = funding.years$year_funding_first
      #funding.years = as.numeric(unlist(strsplit(funding.years$year_funding_all, split = ",")))
      
      #Extract country name
      country.name = mf %>% 
        filter(group == 2) %>% 
        slice(1)
      country.name = country.name$country_en
      
      ##Area of the PA
      wdpa_id = wdpaid #Need to give a name to wdpaid (function argument) different from the varaible in the dataset (wdpaid)
      area_ha = data_pa[data_pa$wdpaid == wdpa_id,]$area_km2*100
      
      #Extract number of pixels in the PA
      #n_pix_pa = length(unique(filter(unmatched.long, group == 2)$assetid))
      n_pix_pa = area_ha/res_ha #This measure is imperfect for extrapolation of total deforestation avoided, as part of a PA can be coastal. Indeed, this extrapolation assumes implicitly that all the PA is covered by forest potentially deforested in absence of the conservation 
      
     #Open a multisession for dataframe computations
      #Note the computations on unmatched units are the slowest here due to the number of observations relatively higher than for matched units
      plan(multisession, gc = TRUE, workers = 6)
      with_progress({
 
  # Make dataframe for plotting trend
  ## Matched units
  df.matched.trend  %<-% {matched.long %>%
    #First, compute deforestation relative to 2000 for each pixel (deforestation as computed in Wolf et al. 2021)
    group_by(assetid) %>%
    mutate(FL_2000_cum = (fc_ha-fc_ha[year == 2000])/fc_ha[year == 2000]*100) %>%
    ungroup() %>%
    #Then compute the average forest cover and deforestation in each year, for treated and control groups
    #Standard deviation and 95% confidence interval is also computed for each variable
    group_by(group, year) %>%
    summarise(n = n(),
              avgFC = mean(fc_ha, na.rm=TRUE), #Compute average forest cover in a pixel, its sd and ci
              sdFC = sd(fc_ha, na.rm = TRUE),
              ciFC_low = avgFC - qt(0.975,df=n-1)*sdFC/sqrt(n),
              ciFC_up = avgFC + qt(0.975,df=n-1)*sdFC/sqrt(n),
              avgFC_tot = n_pix_pa*mean(fc_ha, na.rm=TRUE), #Compute total average forest cover, sd and CI
              sdFC_tot = n_pix_pa*sdFC,
              ciFC_tot_low = avgFC_tot - qt(0.975,df=n-1)*sdFC_tot/sqrt(n),
              ciFC_tot_up = avgFC_tot + qt(0.975,df=n-1)*sdFC_tot/sqrt(n),
              avgFL_2000_cum = mean(FL_2000_cum, na.rm = TRUE), #Compute average forest loss relative to 2000 (Wolf et al 2021), sd and CI
              sdFL_2000_cum = sd(FL_2000_cum, na.rm = TRUE),
              ciFL_low = avgFL_2000_cum - qt(0.975,df=n-1)*sdFL_2000_cum/sqrt(n),
              ciFL_up = avgFL_2000_cum + qt(0.975,df=n-1)*sdFL_2000_cum/sqrt(n),
              matched = TRUE) %>%
    ungroup() %>%
    st_drop_geometry() }
  
  ##Unmatched
  df.unmatched.trend  %<-% {unmatched.long %>%
      #First, compute deforestation relative to 2000 for each pixel (deforestation as computed in Wolf et al. 2021); compute percentage of forest cover in the pixel in 2000
      group_by(assetid) %>%
      mutate(FL_2000_cum = (fc_ha-fc_ha[year == 2000])/fc_ha[year == 2000]*100) %>%
      ungroup() %>% #Compute average percentage of FC in a pixel in 2000, for each group. Compute also standard deviation
    #Then compute the average forest cover, average forest cover percentage, and deforestation in each year, for treated and control groups
    #Standard deviation and 95% confidence interval is also computed for each variable
    group_by(group, year) %>%
    summarise(n = n(),
              avgFC = mean(fc_ha, na.rm=TRUE), #Compute average forest cover in a pixel, its sd and ci
              sdFC = sd(fc_ha, na.rm = TRUE),
              ciFC_low = avgFC - qt(0.975,df=n-1)*sdFC/sqrt(n),
              ciFC_up = avgFC + qt(0.975,df=n-1)*sdFC/sqrt(n),
              avgFC_tot = n_pix_pa*mean(fc_ha, na.rm=TRUE), #Compute total average forest cover, sd and CI
              sdFC_tot = n_pix_pa*sdFC,
              ciFC_tot_low = avgFC_tot - qt(0.975,df=n-1)*sdFC_tot/sqrt(n),
              ciFC_tot_up = avgFC_tot + qt(0.975,df=n-1)*sdFC_tot/sqrt(n),
              avgFL_2000_cum = mean(FL_2000_cum, na.rm = TRUE), #Compute average forest loss relative to 2000 (Wolf et al 2021), sd and CI
              sdFL_2000_cum = sd(FL_2000_cum, na.rm = TRUE),
              ciFL_low = avgFL_2000_cum - qt(0.975,df=n-1)*sdFL_2000_cum/sqrt(n),
              ciFL_up = avgFL_2000_cum + qt(0.975,df=n-1)*sdFL_2000_cum/sqrt(n),
              matched = FALSE) %>%
      #Compute total forest cover loss, knowing area of the PA and average forest cover in 2000 in treated pixels
    ungroup() %>%
    st_drop_geometry() }
  
      })
  
  df.trend = rbind(df.matched.trend, df.unmatched.trend)
     
  
  #Close multisession
  plan(sequential)
  
  #Plot
  ## Change Facet Labels
  fct.labs <- c("Before Matching", "After Matching")
  names(fct.labs) <- c(FALSE, TRUE)
  
  ## Trend Plot for unmatched data
  ### Average forest cover in a pixel
  fig_trend_unm_fc_pix = ggplot(data = df.trend, aes(x = year, y = avgFC)) +
    geom_line(aes(group = group, color = as.character(group))) +
    geom_point(aes(color = as.character(group))) +
    geom_ribbon(aes(ymin = ciFC_low, ymax = ciFC_up, group = group, fill = as.character(group)), alpha = .1, show.legend = FALSE) +
    geom_vline(aes(xintercept=as.character(treatment.year), size="Treatment year"), linetype=1, linewidth=0.5, color="orange") +
    geom_vline(aes(xintercept=as.character(funding.years), size="Funding year"), linetype=2, linewidth=0.5, color="grey30") +
    scale_x_discrete(breaks=seq(2000,2020,5), labels=paste(seq(2000,2020,5))) + 
    scale_color_hue(labels = c("Control", "Treatment")) +
    facet_wrap(matched~., ncol = 2, #scales = 'free_x',
               labeller = labeller(matched = fct.labs)) +
    labs(title = "Evolution of forest cover in a pixel on average (unmatched units)",
         subtitle = paste0("Protected area in ", country.name, ", WDPAID ", wdpaid),
         caption = paste("Ribbons represent 95% confidence intervals.\nThe protected area has a surface of", format(area_ha, big.mark  = ","), "ha and pixels have a resolution of", res_ha, "ha."),
         x = "Year", y = "Forest cover (ha)", color = "Group") +
    theme_bw() +
    theme(
      axis.text.x = element_text(angle = -20, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11),
      axis.title=element_text(size=14),
      
      plot.caption = element_text(hjust = 0),
      
      #legend.position = "bottom",
      legend.title = element_blank(),
      legend.text=element_text(size=14),
      #legend.spacing.x = unit(1.0, 'cm'),
      legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      
      panel.grid.major.x = element_line(color = 'grey', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey', linewidth = 0.2, linetype = 2),
      
      strip.text.x = element_text(size = 12) # Facet Label
      ) +
    guides(size = guide_legend(override.aes = list(color = c("grey30", "orange")))) # Add legend for geom_vline
  
  ### Total forest cover
  fig_trend_unm_fc_tot = ggplot(data = df.trend, aes(x = year, y = avgFC_tot)) +
    geom_line(aes(group = group, color = as.character(group))) +
    geom_point(aes(color = as.character(group))) +
    geom_ribbon(aes(ymin = ciFC_tot_low, ymax = ciFC_tot_up, group = group, fill = as.character(group)), alpha = .1, show.legend = FALSE) +
    geom_vline(aes(xintercept=as.character(treatment.year), size="Treatment year"), linetype=1, linewidth=0.5, color="orange") +
    geom_vline(aes(xintercept=as.character(funding.years), size="Funding year"), linetype=2, linewidth=0.5, color="grey30") +
    scale_x_discrete(breaks=seq(2000,2020,5), labels=paste(seq(2000,2020,5))) +
    scale_color_hue(labels = c("Control", "Treatment")) +
    facet_wrap(matched~., ncol = 2, #scales = 'free_x',
               labeller = labeller(matched = fct.labs)) +
    labs(title = "Evolution of total forest cover (unmatched units)",
         subtitle = paste0("Protected area in ", country.name, ", WDPAID ", wdpaid),
         caption = paste("Ribbons represent 95% confidence intervals. The protected area has a surface of", format(area_ha, big.mark = ","), "ha.\nTotal forest cover is extrapolated from average pixel forest cover, multiplied by the number of pixel in the protected area."),
         x = "Year", y = "Forest cover (ha)", color = "Group") +
    theme_bw() +
    theme(
      axis.text.x = element_text(angle = -20, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11),
      axis.title=element_text(size=14),
      
      plot.caption = element_text(hjust = 0),
      
      #legend.position = "bottom",
      legend.title = element_blank(),
      legend.text=element_text(size=14),
      #legend.spacing.x = unit(1.0, 'cm'),
      legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      
      panel.grid.major.x = element_line(color = 'grey', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey', linewidth = 0.2, linetype = 2),
      
      strip.text.x = element_text(size = 12) # Facet Label
    ) +
    guides(size = guide_legend(override.aes = list(color = c("grey30", "orange")))) # Add legend for geom_vline
  
  ### Cumulative deforestation relative to 2000
  fig_trend_unm_defo = ggplot(data = df.trend, aes(x = year, y = avgFL_2000_cum)) +
    geom_line(aes(group = group, color = as.character(group))) +
    geom_point(aes(color = as.character(group))) +
    geom_ribbon(aes(ymin = ciFL_low, ymax = ciFL_up, group = group, fill = as.character(group)), alpha = .1, show.legend = FALSE) +
    geom_vline(aes(xintercept=as.character(treatment.year), size="Treatment year"), linetype=1, linewidth=0.5, color="orange") +
    geom_vline(aes(xintercept=as.character(funding.years), size="Funding year"), linetype=2, linewidth=0.5, color="grey30") +
    scale_x_discrete(breaks=seq(2000,2020,5), labels=paste(seq(2000,2020,5))) +
    scale_color_hue(labels = c("Control", "Treatment")) +
    facet_wrap(matched~., ncol = 2, #scales = 'free_x',
               labeller = labeller(matched = fct.labs)) +
    labs(title = "Cumulated deforestation relative to 2000 (unmatched units)",
         subtitle = paste0("Protected area in ", country.name, ", WDPAID ", wdpaid),
         caption = paste("Ribbons represent 95% confidence intervals. The protected area has a surface of", format(area_ha, big.mark = ","), "ha."),
         x = "Year", y = "Forest loss relative to 2000 (%)", color = "Group") +
    theme_bw() +
    theme(
      axis.text.x = element_text(angle = -20, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11),
      axis.title=element_text(size=14),
      
      plot.caption = element_text(hjust = 0),
      
      #legend.position = "bottom",
      legend.title = element_blank(),
      legend.text=element_text(size=14),
      #legend.spacing.x = unit(1.0, 'cm'),
      legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      
      panel.grid.major.x = element_line(color = 'grey', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey', linewidth = 0.2, linetype = 2),
      
      strip.text.x = element_text(size = 12) # Facet Label
    ) +
    guides(size = guide_legend(override.aes = list(color = c("grey30", "orange")))) # Add legend for geom_vline
  
  
  # Trend Plot for matched data
  ### Average forest cover in a pixel
  fig_trend_m_fc_pix = ggplot(data = df.matched.trend, aes(x = year, y = avgFC)) +
    geom_line(aes(group = group, color = as.character(group))) +
    geom_point(aes(color = as.character(group))) +
    geom_ribbon(aes(ymin = ciFC_low, ymax = ciFC_up, group = group, fill = as.character(group)), alpha = .1, show.legend = FALSE) +
    geom_vline(aes(xintercept=as.character(treatment.year), size="Treatment year"), linetype=1, linewidth=0.5, color="orange") +
    geom_vline(aes(xintercept=as.character(funding.years), size="Funding year"), linetype=2, linewidth=0.5, color="grey30") +
    scale_x_discrete(breaks=seq(2000,2020,5), labels=paste(seq(2000,2020,5))) + 
    scale_color_hue(labels = c("Control", "Treatment")) +
    labs(title = "Evolution of forest cover in a pixel on average (matched units)",
         subtitle = paste0("Protected area in ", country.name, ", WDPAID ", wdpaid),
         caption = paste("Ribbons represent 95% confidence intervals.\nThe protected area has a surface of", format(area_ha, big.mark  = ","), "ha and pixels have a resolution of", res_ha, "ha."),
         x = "Year", y = "Forest cover (ha)", color = "Group") +
    theme_bw() +
    theme(
      axis.text.x = element_text(angle = -20, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11),
      axis.title=element_text(size=14),
      
      plot.caption = element_text(hjust = 0),
      
      #legend.position = "bottom",
      legend.title = element_blank(),
      legend.text=element_text(size=14),
      #legend.spacing.x = unit(1.0, 'cm'),
      legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      
      panel.grid.major.x = element_line(color = 'grey', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey', linewidth = 0.2, linetype = 2),
      
      strip.text.x = element_text(size = 12) # Facet Label
    ) +
    guides(size = guide_legend(override.aes = list(color = c("grey30", "orange")))) # Add legend for geom_vline
  
  ### Total forest cover
  fig_trend_m_fc_tot = ggplot(data = df.matched.trend, aes(x = year, y = avgFC_tot)) +
    geom_line(aes(group = group, color = as.character(group))) +
    geom_point(aes(color = as.character(group))) +
    geom_ribbon(aes(ymin = ciFC_tot_low, ymax = ciFC_tot_up, group = group, fill = as.character(group)), alpha = .1, show.legend = FALSE) +
    geom_vline(aes(xintercept=as.character(treatment.year), size="Treatment year"), linetype=1, linewidth=0.5, color="orange") +
    geom_vline(aes(xintercept=as.character(funding.years), size="Funding year"), linetype=2, linewidth=0.5, color="grey30") +
    scale_x_discrete(breaks=seq(2000,2020,5), labels=paste(seq(2000,2020,5))) +
    scale_color_hue(labels = c("Control", "Treatment")) +
    labs(title = "Evolution of total forest cover (matched units)",
         subtitle = paste0("Protected area in ", country.name, ", WDPAID ", wdpaid),
         caption = paste("Ribbons represent 95% confidence intervals. The protected area has a surface of", format(area_ha, big.mark = ","), "ha.\nTotal forest cover is extrapolated from average pixel forest cover, multiplied by the number of pixel in the protected area."),
         x = "Year", y = "Total forest cover (ha)", color = "Group") +
    theme_bw() +
    theme(
      axis.text.x = element_text(angle = -20, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11),
      axis.title=element_text(size=14),
      
      plot.caption = element_text(hjust = 0),
      
      #legend.position = "bottom",
      legend.title = element_blank(),
      legend.text=element_text(size=14),
      #legend.spacing.x = unit(1.0, 'cm'),
      legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      
      panel.grid.major.x = element_line(color = 'grey', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey', linewidth = 0.2, linetype = 2),
      
      strip.text.x = element_text(size = 12) # Facet Label
    ) +
    guides(size = guide_legend(override.aes = list(color = c("grey30", "orange")))) # Add legend for geom_vline
  
  ### Cumulative deforestation relative to 2000
  fig_trend_m_defo = ggplot(data = df.matched.trend, aes(x = year, y = avgFL_2000_cum)) +
    geom_line(aes(group = group, color = as.character(group))) +
    geom_point(aes(color = as.character(group))) +
    geom_ribbon(aes(ymin = ciFL_low, ymax = ciFL_up, group = group, fill = as.character(group)), alpha = .1, show.legend = FALSE) +
    geom_vline(aes(xintercept=as.character(treatment.year), size="Treatment year"), linetype=1, linewidth=0.5, color="orange") +
    geom_vline(aes(xintercept=as.character(funding.years), size="Funding year"), linetype=2, linewidth=0.5, color="grey30") +
    scale_x_discrete(breaks=seq(2000,2020,5), labels=paste(seq(2000,2020,5))) +
    scale_color_hue(labels = c("Control", "Treatment")) +
    labs(title = "Cumulated deforestation relative to 2000 (matched units)",
         subtitle = paste0("Protected area in ", country.name, ", WDPAID ", wdpaid),
         caption = paste("Ribbons represent 95% confidence intervals. The protected area has a surface of", format(area_ha, big.mark = ","), "ha."),
         x = "Year", y = "Forest loss relative to 2000 (%)", color = "Group") +
    theme_bw() +
    theme(
      axis.text.x = element_text(angle = -20, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11),
      axis.title=element_text(size=14),
      
      plot.caption = element_text(hjust = 0),
      
      #legend.position = "bottom",
      legend.title = element_blank(),
      legend.text=element_text(size=14),
      #legend.spacing.x = unit(1.0, 'cm'),
      legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      
      panel.grid.major.x = element_line(color = 'grey', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey', linewidth = 0.2, linetype = 2),
      
      strip.text.x = element_text(size = 12) # Facet Label
    ) +
    guides(size = guide_legend(override.aes = list(color = c("grey30", "orange")))) # Add legend for geom_vline
  
  ##Saving plots
  tmp = paste(tempdir(), "fig", sep = "/")
  
  ggsave(paste(tmp, paste0("fig_trend_unmatched_avgFC_", iso, "_", wdpaid, ".png"), sep = "/"),
         plot = fig_trend_unm_fc_pix,
         device = "png",
         height = 6, width = 9)
  ggsave(paste(tmp, paste0("fig_trend_matched_avgFC_", iso, "_", wdpaid, ".png"), sep = "/"),
         plot = fig_trend_m_fc_pix,
         device = "png",
         height = 6, width = 9)
  
  ggsave(paste(tmp, paste0("fig_trend_unmatched_avgFC_tot_", iso, "_", wdpaid, ".png"), sep = "/"),
         plot = fig_trend_unm_fc_tot,
         device = "png",
         height = 6, width = 9)
  ggsave(paste(tmp, paste0("fig_trend_matched_avgFC_tot_", iso, "_", wdpaid, ".png"), sep = "/"),
         plot = fig_trend_m_fc_tot,
         device = "png",
         height = 6, width = 9)
  
  ggsave(paste(tmp, paste0("fig_trend_unmatched_avgFL_cum_2000_", iso, "_", wdpaid, ".png"), sep = "/"),
         plot = fig_trend_unm_defo,
         device = "png",
         height = 6, width = 9)
  ggsave(paste(tmp, paste0("fig_trend_matched_avgFL_cum_2000_", iso, "_", wdpaid, ".png"), sep = "/"),
         plot = fig_trend_m_defo,
         device = "png",
         height = 6, width = 9)
  
  files <- list.files(tmp, full.names = TRUE)
  ##Add each file in the bucket (same foler for every file in the temp)
  for(f in files) 
  {
    cat("Uploading file", paste0("'", f, "'"), "\n")
    aws.s3::put_object(file = f, 
                       bucket = paste("projet-afd-eva-ap", save_dir, iso, wdpaid, sep = "/"),
                       region = "", show_progress = TRUE)
  }
  do.call(file.remove, list(list.files(tmp, full.names = TRUE)))
  
  #Append the log
  cat("#Plot matched and unmatched trends\n-> OK\n\n", file = log, append = TRUE)
  
  return(list("is_ok" = TRUE))
  
    },
  
  error=function(e)
  {
    print(e)
    cat(paste("#Plot matched and unmatched trends\n-> Error :\n", e, "\n\n"), file = log, append = TRUE)
    return(list("is_ok" = FALSE))
  }
  
  # warning = function(w)
  # {
  #   #Print the warning and append the log
  #   print(w)
  #   #Append the log 
  #   cat(paste("#Plot matched and unmatched trends\n-> Warning :\n", w, "\n"), file = log, append = TRUE)
  #   #Return string to inform user to skip
  #   return(list("is_ok" = TRUE))
  # }
  
  )
  return(output)
  
}


# Plot the country grid with matched control and treated, for a given protected area (PA) or all protected areas in a country
##INPUTS
### iso : the ISO3 code of the country considered
### wdpaid : the WDPA ID of the PA considered
### is_pa : logical, whether the plotted grid is for a unique PA or all the PAs in the country considered
### df_pix_matched : dataframe with ID of matched pixels (ID from mapme.biodiversity portfolio)
### path_tmp : temporary folder to store figures
### log : a log file to track progress of the processing
### save_dir : saving directory
##OUTPUTS
### is_ok : a boolean indicating whether or not an error occured inside the function
##DATA SAVED
### Country grid with matched control and treated, for a given protected area (PA) or all protected areas in a country
fn_post_plot_grid = function(iso, wdpaid, is_pa, df_pix_matched, path_tmp, log, save_dir)
{
  
  output = tryCatch(
    
    {
      
  #Import dataframe where each pixel in the grid has both its grid ID and asset ID from the portfolio creation
  df_gridID_assetID = s3read_using(data.table::fread,
                                   object = paste0(save_dir, "/", iso, "/", paste0("df_gridID_assetID_", iso, ".csv")),
                                   bucket = "projet-afd-eva-ap",
                                   opts = list("region" = ""))
  
  #Importing the gridding of the country (funded and analyzed PAs, funded not analyzed PAs, non-funded PAs, buffer, control)
  #Merge with a dataframe so that each pixel in the grid has both its grid ID and asset ID from the portfolio creation
  #Merge with matched pixels dataframe
  grid =  s3read_using(sf::read_sf,
                       object = paste0(save_dir, "/", iso, "/", paste0("grid_param_", iso, ".gpkg")),
                       bucket = "projet-afd-eva-ap",
                       opts = list("region" = "")) %>%
    left_join(df_gridID_assetID, by = "gridID") %>%
    left_join(df_pix_matched, by = "assetid") %>%
    mutate(group_plot = case_when(group_matched == 1 ~ "Control (matched)",
                                  group_matched == 2 ~ "Treatment (matched)",
                                  TRUE ~ group_name))
  
  #Extract country name
  country.name = grid %>% 
    filter(group == 2) %>% 
    slice(1)
  country.name = country.name$country_en
  
  # Visualize and save grouped grid cells
  fig_grid = 
    ggplot(grid) +
    #The original gridding as a first layer
    geom_sf(aes(fill = as.factor(group_plot)), color = NA) +
    scale_fill_brewer(name = "Group", type = "qual", palette = "BrBG", direction = 1) +
    labs(title = paste("Gridding of", country.name, ": matched units"),
         subtitle = ifelse(is_pa == TRUE,
                            yes = paste("Focus on WDPAID", wdpaid),
                            no = "All protected areas analyzed")) +
    theme_bw()
  
  fig_save = ifelse(is_pa == TRUE,
                    yes = paste0(path_tmp, "/fig_grid_group_", iso, "_matched_", wdpaid, ".png"),
                    no = paste0(path_tmp, "/fig_grid_group_", iso, "_matched_all", ".png"))
  ggsave(fig_save,
         plot = fig_grid,
         device = "png",
         height = 6, width = 9)
  aws.s3::put_object(file = fig_save, 
                     bucket = ifelse(is_pa == TRUE,
                                     yes = paste("projet-afd-eva-ap", save_dir, iso, wdpaid, sep = "/"),
                                     no = paste("projet-afd-eva-ap", save_dir, iso, sep = "/")),
                     region = "", 
                     show_progress = FALSE)
  
  #Append the log
  if(is_pa == TRUE)
  {
    cat("#Plot the grid with matched control and treated for the PA \n-> OK\n", file = log, append = TRUE)
  } else cat("#Plot the grid with matched control and treated for all PAs in the country \n-> OK\n", file = log, append = TRUE)

  
  return(list("is_ok" = TRUE))
  
    },
  
  error = function(e)
  {
    print(e)
    if(is_pa == TRUE)
    {
      cat(paste("#Plot the grid with matched control and treated for the PA \n-> Error :\n", e, "\n"), file = log, append = TRUE)
    } else cat(paste("#Plot the grid with matched control and treated for all the PAs in the country \n-> Error :\n", e, "\n"), file = log, append = TRUE)
    return(list("is_ok" = FALSE))
  }
  
  # warning = function(w)
  # {
  #   #Print the warning and append the log
  #   print(w)
  #   if(is_pa == TRUE)
  #   {
  #     cat(paste("#Plot the grid with matched control and treated for the PA \n-> Warning :\n", w, "\n"), file = log, append = TRUE)
  #   } else cat(paste("#Plot the grid with matched control and treated for all the PAs in the country \n-> Warning :\n", w, "\n"), file = log, append = TRUE)
  #   return(list("is_ok" = TRUE))
  # }
  
  )
  
  return(output)
}
```

# Functions for difference-in-difference computations


```r
#####
#Functions to perform difference-in-difference and plot results
#####

#For each function, the aim of the function, inputs, outputs, data saved and notes are detailed. This takes the following form :
#Aim of the function
##INPUTS : the arguments needed in the function
###INPUT 1 to N
##OUTPUTS : the information returned by the function (data frames, numeric, characters, etc.) and necessary to pursue to processing
### OUTPUT 1 to N
##DATA SAVED : information put in the storage but not necessarily need to pursue the processing (figures, tables, data frames, etc.)
### ...
##NOTES : any useful remark
### ...

#Remarks :
##most functions are adapted for errors handling using base::withCallingHandlers(). Basically, the computation steps are declared in a block of withCallingHandlers function, while two other blocks specify what to do in case the first block face a warning or error. In our case, errors led to return a boolean indicating an error has occured and append the log with the error message. Warnings return a boolean but do not block the iteration. They also edit the log with the warning message.
##PA is used for "protected area(s)".
##To save plots and tables : save on temporary folder in the R session then put the saved object in the storage. Indeed print() and ggplot::ggsave() cannot write directly on s3 storage
###


#Load the list of PA matched during the matchign process
##INPUTS :
### iso : the ISO code of the country considered
##OUTPUTS :
### list_pa : a dataframe with the PA matched
### is_ok : a boolean indicating whether or not an error occured inside the function
fn_did_list_pa = function(iso, load_dir)
{
  output = tryCatch(
    
    {
      
  list_pa = s3read_using(data.table::fread,
                         bucket = "projet-afd-eva-ap",
                         object = paste(load_dir, iso, paste0("list_pa_matched_", iso, ".csv"), sep = "/"),
                         opts = list("region" = ""))
  list_pa = unique(list_pa$wdpaid)
  
  return(list("list_pa" = list_pa, "is_ok" = TRUE))
    },
  
  error = function(e)
  {
    print(e)
    #cat(paste("Error in loading the list of protected areas :\n", e, "\n"), file = log, append = TRUE)
    print(paste("Error in loading the list of protected areas :\n", e, "\n"))
    return(list("is_ok" = FALSE))
  }
  
  )
  
  return(output)
}


#For a protected area, compute annual deforestation rates à la Wolf et al. 2021, before and after treatment
## INPUTS 
### iso : the iso3 code for the country considered
### wdpaid : the WDPAID of the PA considered
### alpha : the margin of error to define confidence interval
### load_dir : a path to load matching frame
### ext_output : the output extension
## OUTPUTS
### df_fl_annual_wolf : a dataframe with statistics on annual deforestation in matched treated and control units, computed à la Wolf et al. 2021
### is_ok : a boolean indicating whether or not an error occured inside the function 
fn_fl_wolf = function(iso, wdpaid, alpha, load_dir, ext_input)
{
  output = tryCatch(
    
    {
      
  #Import matched units
  df_long = s3read_using(data.table::fread,
                           object = paste0(load_dir, "/", iso, "/", wdpaid, "/", paste0("matched_long", "_", iso, "_", wdpaid, ext_input)),
                           bucket = "projet-afd-eva-ap",
                           opts = list("region" = "")) %>%
    dplyr::select(c(region, iso3, wdpaid, group, assetid, status_yr, year_funding_first, year_funding_all, res_m, year, var, fc_ha))
  #select(c(region, country_en, iso3, wdpaid, group, status_yr, year_funding_first, year_funding_all, year, var, fc_ha))
  
  ##Extract country iso
  country.iso = df_long %>% 
    filter(group == 2) %>% 
    slice(1)
  country.iso = country.iso$iso3
  
  ##Extract region name
  region.name = df_long %>% 
    filter(group == 2) %>% 
    slice(1)
  region.name = region.name$region
  
  #Compute annual deforestation rates à la Wolf et al. 2021 before and after treatment for treated, and for all the period for controls. This is averaged across pixels.
  df_fl_annual_wolf = df_long %>%
    mutate(treatment_year = case_when(group == 1 ~0,
                                      group == 2 ~status_yr), #Set treatment year to 0 for control units (required by did::att_gt)
           time = ifelse(group == 2, yes = year-treatment_year, no = NA),
           .after = status_yr) %>%
    group_by(assetid) %>%
    # mutate(FL_2000_cum = (fc_ha-fc_ha[year == 2000])/fc_ha[year == 2000]*100,
    #        fc_2000 = fc_ha[year == 2000]) %>%
    mutate(FL_annual_wolf_pre = ifelse(group == 2, yes = ((fc_ha[time == -1]/fc_ha[year == 2000])^(1/(year[time == -1] - 2000))-1)*100, no = NA),
           FL_annual_wolf_post = ifelse(group == 2, yes = ((fc_ha[time == max(time)]/fc_ha[time == 0])^(1/max(time))-1)*100, no = NA),
           FL_annual_wolf_tot = ((fc_ha[year == 2021]/fc_ha[year == 2000])^(1/(2021-2000))-1)*100) %>%
    slice(1) %>%
    ungroup() %>%
    group_by(group) %>%
    summarize(avgFL_annual_wolf_pre = mean(FL_annual_wolf_pre, na.rm = TRUE),
              avgFL_annual_wolf_post = mean(FL_annual_wolf_post, na.rm = TRUE),
              avgFL_annual_wolf_tot = mean(FL_annual_wolf_tot, na.rm = TRUE),
              medFL_annual_wolf_pre = median(FL_annual_wolf_pre, na.rm = TRUE),
              medFL_annual_wolf_post = median(FL_annual_wolf_post, na.rm = TRUE),
              medFL_annual_wolf_tot = median(FL_annual_wolf_tot, na.rm = TRUE),
              sdFL_annual_wolf_pre = sd(FL_annual_wolf_pre, na.rm = TRUE),
              sdFL_annual_wolf_post = sd(FL_annual_wolf_post, na.rm = TRUE),
              sdFL_annual_wolf_tot = sd(FL_annual_wolf_tot, na.rm = TRUE)) %>%
    ungroup() %>%
    mutate(region = region.name, iso3 = country.iso, wdpaid = wdpaid, .before = "group") %>%
    mutate(group = case_when(group == 1 ~ "Control",
                             group == 2 ~ "Treated"))
  
  return(list("df_fl_annual_wolf" = df_fl_annual_wolf, "is_ok" = TRUE))
    },
  
  error = function(e)
  {
    print(e)
    #cat(paste("Error while computing annual deforestation à la Wolf et al. 2021 :\n", e, "\n"), file = log, append = TRUE)
    print(paste("Error while annual deforestation à la Wolf et al. 2021 :\n", e, "\n"))
    return(list("is_ok" = FALSE))
  }
  
  )
}

#Compute the treatment effect for a given protected area that is supported by the AFD. This function specifically includes information related to funding we obtain from AFD internal services.
## INPUTS : 
### iso : the iso3 code for the country considered
### wdpaid : the WDPAID of the PA considered
### data_pa : dataset with information on protected areas, and especially their surfaces
### data_fund : information on funding from AFD internal datasets, on AFD funded projects related to protected areas.
### data_report : list of projects related to protected areas in AFD, reported by technical departments
### alpha : the threshold for confidence interval
### is_m : boolean stating whether we compute treatment effects from matched (TRUE) or unmatched treated and control units (FALSE)
### save_dir : the saving directory in the remote storage
### load_dir : the loading directory in the remote storage
### ext_input : the extension of input dataframe
## OUTPUTS :
### df_fc_attgt : treatment effect computed for the protected area considered, expressed in avoided deforestation (hectare)
### df_fl_attgt : treatment effect computed for the protected area considered, expressed in change of deforestation rate
### is_ok : a boolean indicating whether or not an error occured inside the function 
## DATA SAVED :
### Dynamic treatment effects : avoided deforestation in an average pixel (in ha), avoided deforestation relative to 2000 forest cover, avoided deforestation extrapolated to the entire protected area (in ha), change in deforestation rate (in percentage points)
fn_did_att_afd = function(iso, wdpaid, data_pa, data_fund, data_report, alpha, is_m, load_dir, ext_input, save_dir)
{
  
  output = tryCatch(
    
    {
      
  #Loading matched and unmatched datasets
  df_long_m = s3read_using(data.table::fread,
                           object = paste0(load_dir, "/", iso, "/", wdpaid, "/", paste0("matched_long", "_", iso, "_", wdpaid, ext_input)),
                           bucket = "projet-afd-eva-ap",
                           opts = list("region" = "")) %>%
    dplyr::select(c(region, iso3, wdpaid, group, assetid, status_yr, year_funding_first, year_funding_all, res_m, year, var, fc_ha))
  #dplyr::select(c(region, country_en, iso3, wdpaid, group, status_yr, year_funding_first, year_funding_all, year, var, fc_ha))
  
  df_long_unm = s3read_using(data.table::fread,
                             object = paste0(load_dir, "/", iso, "/", wdpaid, "/", paste0("unmatched_long", "_", iso, "_", wdpaid, ext_input)),
                             bucket = "projet-afd-eva-ap",
                             opts = list("region" = "")) %>%
    dplyr::select(c(region, iso3, wdpaid, group, assetid, status_yr, year_funding_first, year_funding_all, year, res_m, var, fc_ha))
  #dplyr::select(c(region, country_en, iso3, wdpaid, group, status_yr, year_funding_first, year_funding_all, year, var, fc_ha))
  
  # Define the working datasets depending on the is_m value
  if(is_m == TRUE)
  {
    df_long = df_long_m
  } else{df_long = df_long_unm 
  }
  
  #Extract some relevant variables for later plots and treatment effect computations
  ##Extract spatial resolution of pixels res_m and define pixel area in ha
  res_m = unique(df_long$res_m)
  res_ha = res_m^2*1e-4
  
  ##Extract treatment year
  treatment.year = df_long %>% 
    filter(group == 2) %>% 
    slice(1)
  treatment.year = treatment.year$status_yr
  
  ##Extract funding years
  df_fund_yr = df_long %>% 
    filter(group == 2) %>% 
    slice(1)
  funding.years = df_fund_yr$year_funding_first
  list.funding.years = df_fund_yr$year_funding_all
  
  ##Extract country name
  # country.name = df_long %>% 
  #   filter(group == 2) %>% 
  #   slice(1)
  # country.name = country.name$country_en
  
  ##Extract country iso
  country.iso = df_long %>% 
    filter(group == 2) %>% 
    slice(1)
  country.iso = country.iso$iso3
  
  ##Extract region name
  region.name = df_long %>% 
    filter(group == 2) %>% 
    slice(1)
  region.name = region.name$region
  
  ##Extract more information not in the matched dataframe
  ### Area
  wdpa_id = wdpaid #Need to give a name to wdpaid (function argument) different from the varaible in the dataset (wdpaid)
  area_ha = data_pa[data_pa$wdpaid == wdpa_id,]$area_km2*100
  ### Name of the PA
  pa.name = data_pa %>% 
    filter(wdpaid == wdpa_id) %>% 
    slice(1)
  pa.name = pa.name$name_pa
  ### Country name
  country.name = data_pa %>% 
    filter(wdpaid == wdpa_id) %>% 
    slice(1)
  country.name = country.name$country_en
  ### AFD project ID
  # id.project = data_pa %>% 
  #   filter(wdpaid == wdpa_id) %>% 
  #   slice(1)
  # id.project = id.project$id_projet
  ### WDPA status
  status.wdpa = data_pa %>% 
    filter(wdpaid == wdpa_id) %>% 
    slice(1)
  status.wdpa = status.wdpa$status
  ### IUCN category and description
  iucn.wdpa = data_pa %>% 
    filter(wdpaid == wdpa_id) %>% 
    slice(1)
  iucn.cat = iucn.wdpa$iucn_cat
  iucn.des = iucn.wdpa$iucn_des_en
  ### Ecosystem
  eco.wdpa = data_pa %>% 
    filter(wdpaid == wdpa_id) %>% 
    slice(1)
  eco.wdpa = eco.wdpa$marine
  ### Governance
  gov.wdpa = data_pa %>% 
    filter(wdpaid == wdpa_id) %>% 
    slice(1)
  gov.wdpa = gov.wdpa$gov_type
  ### Owner
  own.wdpa = data_pa %>% 
    filter(wdpaid == wdpa_id) %>% 
    slice(1)
  own.wdpa = own.wdpa$own_type
  
  ## Extract information on funding
  ### Type of funding
  fund.type = data_fund %>%
    filter(id_projet == id.project) %>%
    slice(1)
  fund.type = fund.type$libelle_produit
  ### Cofunders
  cofund = data_fund %>%
    filter(id_projet == id.project) %>%
    slice(1)
  cofund = cofund$cofinanciers
  ### KfW ?
  kfw = data_fund %>%
    filter(id_projet == id.project) %>%
    slice(1)
  kfw = kfw$kfw
  ### FFEM ?
  ffem = data_fund %>%
    filter(id_projet == id.project) %>%
    slice(1)
  ffem = ffem$ffem

  ## Extract reporting department
  reporter = data_report %>%
    filter(wdpaid == wdpa_id & id_projet == id.project & nom_ap == pa.name) %>%
    slice(1)
  reporter = reporter$auteur_entree
  
  #Extract number of pixels in the PA
  #n_pix_pa = length(unique(filter(df_long_unm, group == 2)$assetid))
  n_pix_pa = area_ha/res_ha
  
  #Average forest cover in a treated pixel in 2000
  ## For matched 
  avgFC_2000_m = df_long_m %>% 
    filter(group == 2 & year == 2000) 
  avgFC_2000_m = mean(avgFC_2000_m$fc_ha, na.rm = TRUE)
  ## For unmatched
  avgFC_2000_unm = df_long_unm %>% 
    filter(group == 2 & year == 2000) 
  avgFC_2000_unm = mean(avgFC_2000_unm$fc_ha, na.rm = TRUE)
  
  #Then modify the dataframe before difference-in-difference computations
  ## Set treatment year = 0 for controls (necessary for did package to consider "never treated" units)
  ## Compute cumulative deforestation relative to 2000 forest cover (outcome where TE is computed)
  df_did = df_long %>%
    mutate(treatment_year = case_when(group == 1 ~0,
                                      group == 2 ~status_yr), #Set treatment year to 0 for control units (required by did::att_gt)
           time = ifelse(group == 2, yes = year-treatment_year, no = NA),
           .after = status_yr) %>%
    group_by(assetid) %>%
    # mutate(FL_2000_cum = (fc_ha-fc_ha[year == 2000])/fc_ha[year == 2000]*100,
    #        fc_2000 = fc_ha[year == 2000]) %>%
    mutate(FL_2000_cum = case_when(fc_ha[year == 2000] > 0 ~ (fc_ha-fc_ha[year == 2000])/fc_ha[year == 2000]*100, 
                                   TRUE ~ NA)) %>%
    ungroup()
  

  ##Average forest cover in 2000 in a pixel, and average share of forest cover in a pixel
  # fc_2000_avg = mean(df_did[df_did$group == 2,]$fc_2000, na.rm = TRUE)
  # per_fc_2000_avg = min(fc_2000_avg/res_ha, 1) #Take the min as in some cases, reported forest cover is higher than pixel area
  
  #Compute dynamic treatment effect with did package. 
  ## Control are "never treated" units, no covariate is added in the regression estimated with doubly-robust method
  ## standard errors are computed with bootstrap, and confidence intervals computed from it.
  ## No clustering is performed as it does not seem relevant in our case (https://blogs.worldbank.org/impactevaluations/when-should-you-cluster-standard-errors-new-wisdom-econometrics-oracle)
  ## Pseudo treatment effects are computed for each pre-treatment year (varying base period)
  
  ##For forest cover (ha and %)
  ### treatment effect computation
  fc_attgt = did::att_gt(yname = "fc_ha",
                         gname = "treatment_year",
                         idname = "assetid",
                         tname = "year",
                         control_group = "nevertreated", #Thsi corresponds to control pixels as defined in the matching , with treatment year set to 0
                         xformla = ~1,
                         alp = alpha, #For 95% confidence interval
                         allow_unbalanced_panel = TRUE, #Ensure no unit is dropped, though every pixel should have data for all years in the period
                         bstrap=TRUE, #Compute bootstrap CI
                         biters = 1000, #The number of bootstrap iteration, 1000 is default
                         cband = TRUE, #Compute CI
                         clustervars = NULL, #No clustering seems relevant to me 
                         base_period = "varying",
                         data = df_did,
                         print_details = F)
  ##For change in deforestation rate (percentage points)
  ### treatment effect computation
  fl_attgt = did::att_gt(yname = "FL_2000_cum",
                         gname = "treatment_year",
                         idname = "assetid",
                         tname = "year",
                         control_group = "nevertreated", #Thsi corresponds to control pixels as defined in the matching , with treatment year set to 0
                         xformla = ~1,
                         alp = alpha, #For 95% confidence interval
                         allow_unbalanced_panel = TRUE, #Ensure no unit is dropped, though every pixel should have data for all years in the period
                         bstrap=TRUE, #Compute bootstrap CI
                         biters = 1000, #The number of bootstrap iteration, 1000 is default
                         cband = TRUE, #Compute CI
                         clustervars = NULL, #No clustering seems relevant to me
                         base_period = "varying",
                         data = df_did,
                         print_details = F)
  
  
  ### Report results in a dataframe
  ### The computed is at pixel level
  ### This treatment effect is aggregated to protected area by multiplying treatment effect by the number of pixel in the PA. It is also expressed in percentage of pixel area (avoided deforestation in share of pixel area)
  ### confidence intervals (at pixel level) are computed from bootstrap standard errors after a coefficient is applied.
  ### This computation takes the one from did:::summary.MP function, line 15 and 16. 
  ### They are multiplied by the number of pixels to compute confidence intervals for treatment effect at protected area level 
  ### They are divided by the pixel area to compute CI for treatment effect in percentage of pixel area
  df_fc_attgt = data.frame("treatment_year" = fc_attgt$group,
                           "year" = fc_attgt$t,
                           "att_pix" = fc_attgt$att,
                           "c" = fc_attgt$c,
                           "se" = fc_attgt$se,  
                           "n" = fc_attgt$n) %>%
    #Compute treatment effect at PA level and in share of pixel area
    ## att_pa : the total avoided deforestation is the avoided deforestation in ha in a given pixel, multiplied by the number of pixel in the PA.
    ## att_per : avoided deforestation in a pixel, as a share of average forest cover in 2000 in matched treated. Can be extrapolated to full PA in principle (avoided deforestation in share of 2000 forest cover)
    mutate(att_pa = att_pix*n_pix_pa,
           att_per = att_pix/avgFC_2000_m*100) %>% 
    #Compute time relative to treatment year
    mutate(time = year - treatment_year,
           .before = year) %>%
    #Compute confidence intervals
    mutate(cband_lower_pix = round(att_pix-c*se, 4),
           cband_upper_pix = round(att_pix+c*se, 4),
           cband_lower_pa = cband_lower_pix*n_pix_pa,
           cband_upper_pa = cband_upper_pix*n_pix_pa,
           cband_lower_per = cband_lower_pix/avgFC_2000_m*100,
           cband_upper_per = cband_upper_pix/avgFC_2000_m*100,
           sig = sign(cband_lower_pix) == sign(cband_upper_pix),
           sig_5 = ifelse(max(time) >=5, yes = sig[time == 5] == TRUE, no = NA),
           sig_10 = ifelse(max(time) >= 10, yes = sig[time == 10] == TRUE, no = NA),
           sig_end = sig[time == max(time)] == TRUE,
           alpha = alpha) %>%
    #Add relevant information
    mutate(region = region.name,
           country_en = country.name,
           iso3 = country.iso,
           name_pa = pa.name,
           wdpaid = wdpaid,
           res_ha = res_ha,
           id_projet = id.project,
           status_wdpa = status.wdpa,
           iucn_cat = iucn.cat,
           iucn_des_en = iucn.des,
           gov_type = gov.wdpa,
           own_type = own.wdpa,
           marine = eco.wdpa,
           cofund = cofund,
           kfw = kfw,
           ffem = ffem,
           fund_type = fund.type,
           dept_report = reporter,
           funding_year = funding.years,
           funding_year_list = list.funding.years,
           .before = "treatment_year")
  
  # Same for change in deforestation rate
  df_fl_attgt = data.frame("treatment_year" = fl_attgt$group,
                           "year" = fl_attgt$t,
                           "att" = fl_attgt$att,
                           "c" = fl_attgt$c,
                           "se" = fl_attgt$se,
                           "n" = fl_attgt$n) %>%
    #Compute time relative to treatment year
    mutate(time = year - treatment_year,
           .before = year) %>%
    mutate(cband_lower = round(att-c*se, 4),
           cband_upper = round(att+c*se, 4),
           sig = sign(cband_lower) == sign(cband_upper),
           sig_5 = ifelse(max(time) >=5, yes = sig[time == 5] == TRUE, no = NA),
           sig_10 = ifelse(max(time) >= 10, yes = sig[time == 10] == TRUE, no = NA),
           sig_end = sig[time == max(time)] == TRUE,
           alpha = alpha) %>%
    #Compute time relative to treatment year
    mutate(time = year - treatment_year,
           .before = year) %>%
    #Add relevant information
    mutate(region = region.name,
           country_en = country.name,
           iso3 = country.iso,
           name_pa = pa.name,
           wdpaid = wdpaid,
           res_ha = res_ha,
           id_projet = id.project,
           status_wdpa = status.wdpa,
           iucn_cat = iucn.cat,
           iucn_des_en = iucn.des,
           gov_type = gov.wdpa,
           own_type = own.wdpa,
           marine = eco.wdpa,
           cofund = cofund,
           kfw = kfw,
           ffem = ffem,
           fund_type = fund.type,
           dept_report = reporter,
           funding_year = funding.years,
           funding_year_list = list.funding.years,
           .before = "treatment_year")
  
  ###Plot results
  ## treatment effect : avoided deforestation at pixel level (in ha)
  fig_att_pix = ggplot(data = df_fc_attgt,
                       aes(x = time, y = att_pix)) %>%
    + geom_line(color = "#08519C") %>%
    + geom_point(color = "#08519C") %>%
    + geom_ribbon(aes(ymin = cband_lower_pix, ymax = cband_upper_pix),
                  alpha=0.1, fill = "#FB6A4A", color = "black", linetype = "dotted") %>%
    + labs(title = ifelse(is_m == TRUE, 
                          yes = "Deforestation avoided in a pixel,on average (matched)",
                          no = "Deforestation avoided in a pixel,on average (unmatched)"),
           subtitle = paste0(pa.name, ", ", country.name, ", implemented in ", treatment.year),
           caption = paste("WDPA ID :", wdpa_id, "|", format(area_ha, big.mark = ","), "ha |", "Pixel resolution :", res_ha, "ha", "\nRibbon represents", (1-alpha)*100, "% confidence interval.\nTreatment effect is interpreted as the deforestation avoided at pixel level in hectare, due to the conservation program.\nA negative effect means the conservation program has caused higher deforestation."),
           y = "Area (ha)",
           x = "Year relative to treatment (t = 0)") %>%
    + scale_x_continuous(breaks=seq(min(df_fc_attgt$time),max(df_fc_attgt$time),by=1)) %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_blank(),
      
      #legend.position = "bottom",
      legend.title = element_blank(),
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  
  # treatment effect : avoided deforestation in terms of 2000 forest cover
  fig_att_per = ggplot(data = df_fc_attgt,
                       aes(x = time, y = att_per)) %>%
    + geom_line(color = "#08519C") %>%
    + geom_point(color = "#08519C") %>%
    + geom_ribbon(aes(ymin = cband_lower_per, ymax = cband_upper_per),
                  alpha=0.1, fill = "#FB6A4A", color = "black", linetype = "dotted") %>%
    + labs(title = ifelse(is_m == TRUE, 
                          yes = "Average deforestation avoided relative to 2000 forest cover (matched)",
                          no = "Average deforestation avoided relative to 2000 forest cover (unmatched)"),
           subtitle = paste0(pa.name, ", ", country.name, ", implemented in ", treatment.year),
           caption = paste("WDPA ID :", wdpa_id, "|", format(area_ha, big.mark = ","), "ha |", "Pixel resolution :", res_ha, "ha", "\nRibbon represents", (1-alpha)*100, "% confidence interval.\nTreatment effect is interpreted as the deforestation avoided in percentage of 2000 forest cover.\nA negative effect means the conservation program has caused higher deforestation."),
           y = "%",
           x = "Year relative to treatment (t = 0)") %>%
    + scale_x_continuous(breaks=seq(min(df_fc_attgt$time),max(df_fc_attgt$time),by=1)) %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_blank(),
      
      #legend.position = "bottom",
      legend.title = element_blank(),
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  
  # treatment effect : avoided deforestation in the PA
  fig_att_pa = ggplot(data = df_fc_attgt,
                      aes(x = time, y = att_pa)) %>%
    + geom_line(color = "#08519C") %>%
    + geom_point(color = "#08519C") %>%
    + geom_ribbon(aes(ymin = cband_lower_pa, ymax = cband_upper_pa),
                  alpha=0.1, fill = "#FB6A4A", color = "black", linetype = "dotted") %>%
    + labs(title = ifelse(is_m == TRUE, 
                          yes = "Total deforestation avoided (matched)",
                          no = "Total deforestation avoided (unmatched)"),
           subtitle = paste0(pa.name, ", ", country.name, ", implemented in ", treatment.year),
           caption = paste("WDPA ID :", wdpa_id, "|", format(area_ha, big.mark = ","), "ha |", "Pixel resolution :", res_ha, "ha",  "\nRibbon represents", (1-alpha)*100, "% confidence interval.\nTreatment effect is interpreted as the total deforestation avoided in the protected areas, in hectare (ha).\nThis measure is an extrapolation to the full protected area of average avoided deforestation at pixel level.\nA negative effect means the conservation program has caused higher deforestation."),
           y = "Forest area (ha)",
           x = "Year relative to treatment (t = 0)") %>%
    + scale_x_continuous(breaks=seq(min(df_fc_attgt$time),max(df_fc_attgt$time),by=1)) %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_blank(),
      
      #legend.position = "bottom",
      legend.title = element_blank(),
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  

  # treatment effect : change in deforestation rate
  fig_fl_att = ggplot(data = df_fl_attgt,
                      aes(x = time, y = att)) %>%
    + geom_line(color = "#08519C") %>%
    + geom_point(color = "#08519C") %>%
    + geom_ribbon(aes(ymin = cband_lower, ymax = cband_upper),
                  alpha=0.1, fill = "#FB6A4A", color = "black", linetype = "dotted") %>%
    + labs(title = "Effect of the conservation on the deforestation rate, relative to 2000",
           subtitle = paste0(pa.name, ", ", country.name, ", implemented in ", treatment.year),
           caption = paste("WDPA ID :", wdpa_id, "|", format(area_ha, big.mark = ","), "ha |", "Pixel resolution :", res_ha, "ha", "\nRibbon represents ", (1-alpha)*100, " % confidence interval.\nTreatment effect is interpreted as the reduction of cumulated deforestation rate (relative to 2000 forest cover) in percentage points (pp).\nA negative effect means the conservation program has caused higher deforestation."),
           y = "Reduction of deforestation (p.p)",
           x = "Year relative to treatment (t = 0)") %>%
    + scale_x_continuous(breaks=seq(min(df_fc_attgt$time),max(df_fc_attgt$time),by=1)) %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_blank(),
      
      #legend.position = "bottom",
      legend.title = element_blank(),
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  
  
  
  ##Saving plots
  tmp = paste(tempdir(), "fig", sep = "/")
  
  ggsave(ifelse(is_m == TRUE,
                yes = paste(tmp, paste0("fig_att_pix_", iso, "_", wdpaid, "_m", ".png"), sep = "/"),
                no = paste(tmp, paste0("fig_att_pix_", iso, "_", wdpaid, "_unm", ".png"), sep = "/")),
         plot = fig_att_pix,
         device = "png",
         height = 6, width = 9)
  ggsave(ifelse(is_m == TRUE,
                yes = paste(tmp, paste0("fig_att_pa_", iso, "_", wdpaid, "_m", ".png"), sep = "/"),
                no = paste(tmp, paste0("fig_att_pa_", iso, "_", wdpaid, "_unm", ".png"), sep = "/")),
         plot = fig_att_pa,
         device = "png",
         height = 6, width = 9)
  ggsave(ifelse(is_m == TRUE,
                yes = paste(tmp, paste0("fig_att_per_", iso, "_", wdpaid, "_m", ".png"), sep = "/"),
                no = paste(tmp, paste0("fig_att_per_", iso, "_", wdpaid, "_unm", ".png"), sep = "/")),
         plot = fig_att_per,
         device = "png",
         height = 6, width = 9)
  ggsave(paste(tmp, paste0("fig_fl_att_", iso, "_", wdpaid, ".png"), sep = "/"),
         plot = fig_fl_att,
         device = "png",
         height = 6, width = 9)
  
  files <- list.files(tmp, full.names = TRUE)
  ##Add each file in the bucket (same foler for every file in the temp)
  for(f in files) 
  {
    cat("Uploading file", paste0("'", f, "'"), "\n")
    aws.s3::put_object(file = f, 
                       bucket = paste("projet-afd-eva-ap", save_dir, iso, wdpaid, sep = "/"),
                       region = "", show_progress = TRUE)
  }
  do.call(file.remove, list(list.files(tmp, full.names = TRUE)))
  
  
  
  #Return outputs
  return(list("df_fc_att" = df_fc_attgt, "df_fl_att"  = df_fl_attgt, "is_ok" = TRUE))
  
    },
  
  error = function(e)
  {
    print(e)
    #cat(paste("Error while computing/plotting DiD :\n", e, "\n"), file = log, append = TRUE)
    print(paste("Error while computing/plotting DiD :\n", e, "\n"))
    return(list("is_ok" = FALSE))
  }
  
  )
  
  return(output)
  
  #TEST : is treatment effect computed by the did package coherent with manual computations ? 
  # --> YES :D
  # test = df_did %>% 
  #   group_by(group, year) %>%
  #   summarize(avgFL_2000_cum = mean(FL_2000_cum, na.rm = TRUE),
  #             avgFC_ha = mean(fc_ha, na.rm = TRUE)) %>%
  #   ungroup() %>%
  #   mutate(fc_te2 = (avgFC_ha[year == 2009 & group == 2] - avgFC_ha[year == 2006 & group == 2]) - (avgFC_ha[year == 2009 & group == 1] - avgFC_ha[year == 2006 & group == 1]),
  #          fl_te2 = (avgFL_2000_cum[year == 2009 & group == 2] - avgFL_2000_cum[year == 2006 & group == 2]) - (avgFL_2000_cum[year == 2009 & group == 1] - avgFL_2000_cum[year == 2006 & group == 1]))
  # 
  
  
}


#Compute the treatment effect for a given protected area. This function can be used on any protected area for which, though no funding information will be displayed contrary to fn_did_att_afd.
## INPUTS 
### iso : the iso3 code for the country considered
### wdpaid : the WDPAID of the PA considered
### data_pa : dataset with information on protected areas, and especially their surfaces
### alpha : the threshold for confidence interval
### is_m : boolean stating whether we compute treatment effects from matched (TRUE) or unmatched treated and control units (FALSE)
### save_dir : the saving directory in the remote storage
### load_dir : the loading directory in the remote storage
### ext_input : the extension of input dataframe
## OUTPUTS
### df_fc_attgt : treatment effect computed for the protected area considered, expressed in avoided deforestation (hectare)
### df_fl_attgt : treatment effect computed for the protected area considered, expressed in change of deforestation rate
### is_ok : a boolean indicating whether or not an error occured inside the function 
## DATA SAVED :
### Dynamic treatment effects : avoided deforestation in an average pixel (in ha), avoided deforestation relative to 2000 forest cover, avoided deforestation extrapolated to the entire protected area (in ha), change in deforestation rate (in percentage points)
fn_did_att_general = function(iso, wdpaid, data_pa, alpha, is_m, load_dir, ext_input, save_dir)
{
  
  output = tryCatch(
    
    {
      
      #Loading matched and unmatched datasets
      df_long_m = s3read_using(data.table::fread,
                               object = paste0(load_dir, "/", iso, "/", wdpaid, "/", paste0("matched_long", "_", iso, "_", wdpaid, ext_input)),
                               bucket = "projet-afd-eva-ap",
                               opts = list("region" = "")) %>%
        dplyr::select(c(region, iso3, wdpaid, group, assetid, status_yr, year_funding_first, year_funding_all, res_m, year, var, fc_ha))
      #dplyr::select(c(region, country_en, iso3, wdpaid, group, status_yr, year_funding_first, year_funding_all, year, var, fc_ha))
      
      df_long_unm = s3read_using(data.table::fread,
                                 object = paste0(load_dir, "/", iso, "/", wdpaid, "/", paste0("unmatched_long", "_", iso, "_", wdpaid, ext_input)),
                                 bucket = "projet-afd-eva-ap",
                                 opts = list("region" = "")) %>%
        dplyr::select(c(region, iso3, wdpaid, group, assetid, status_yr, year_funding_first, year_funding_all, year, res_m, var, fc_ha))
      #dplyr::select(c(region, country_en, iso3, wdpaid, group, status_yr, year_funding_first, year_funding_all, year, var, fc_ha))
      
      # Define the working datasets depending on the is_m value
      if(is_m == TRUE)
      {
        df_long = df_long_m
      } else{df_long = df_long_unm 
      }
      
      #Extract some relevant variables
      ##Extract spatial resolution of pixels res_m and define pixel area in ha
      res_m = unique(df_long$res_m)
      res_ha = res_m^2*1e-4
      
      ##Extract treatment year
      treatment.year = df_long %>% 
        filter(group == 2) %>% 
        slice(1)
      treatment.year = treatment.year$status_yr
      
      ##Extract funding years
      df_fund_yr = df_long %>% 
        filter(group == 2) %>% 
        slice(1)
      funding.years = df_fund_yr$year_funding_first
      list.funding.years = df_fund_yr$year_funding_all
      
      ##Extract country name
      # country.name = df_long %>% 
      #   filter(group == 2) %>% 
      #   slice(1)
      # country.name = country.name$country_en
      
      ##Extract country iso
      country.iso = df_long %>% 
        filter(group == 2) %>% 
        slice(1)
      country.iso = country.iso$iso3
      
      ##Extract region name
      region.name = df_long %>% 
        filter(group == 2) %>% 
        slice(1)
      region.name = region.name$region
      
      ##Extract more information not in the matched dataframe
      ### Area
      wdpa_id = wdpaid #Need to give a name to wdpaid (function argument) different from the varaible in the dataset (wdpaid)
      area_ha = data_pa[data_pa$wdpaid == wdpa_id,]$area_km2*100
      ### Name of the PA
      pa.name = data_pa %>% 
        filter(wdpaid == wdpa_id) %>% 
        slice(1)
      pa.name = pa.name$name_pa
      ### Country name
      country.name = data_pa %>% 
        filter(wdpaid == wdpa_id) %>% 
        slice(1)
      country.name = country.name$country_en
      ### WDPA status
      status.wdpa = data_pa %>% 
        filter(wdpaid == wdpa_id) %>% 
        slice(1)
      status.wdpa = status.wdpa$status
      ### IUCN category and description
      iucn.wdpa = data_pa %>% 
        filter(wdpaid == wdpa_id) %>% 
        slice(1)
      iucn.cat = iucn.wdpa$iucn_cat
      iucn.des = iucn.wdpa$iucn_des_en
      ### Ecosystem
      eco.wdpa = data_pa %>% 
        filter(wdpaid == wdpa_id) %>% 
        slice(1)
      eco.wdpa = eco.wdpa$marine
      ### Governance
      gov.wdpa = data_pa %>% 
        filter(wdpaid == wdpa_id) %>% 
        slice(1)
      gov.wdpa = gov.wdpa$gov_type
      ### Owner
      own.wdpa = data_pa %>% 
        filter(wdpaid == wdpa_id) %>% 
        slice(1)
      own.wdpa = own.wdpa$own_type
      
      #Extract number of pixels in the PA
      #n_pix_pa = length(unique(filter(df_long_unm, group == 2)$assetid))
      n_pix_pa = area_ha/res_ha
      
      #Average forest cover in a treated pixel in 2000
      ## For matched 
      avgFC_2000_m = df_long_m %>% 
        filter(group == 2 & year == 2000) 
      avgFC_2000_m = mean(avgFC_2000_m$fc_ha, na.rm = TRUE)
      ## For unmatched
      avgFC_2000_unm = df_long_unm %>% 
        filter(group == 2 & year == 2000) 
      avgFC_2000_unm = mean(avgFC_2000_unm$fc_ha, na.rm = TRUE)
      
      #Then modify the dataframe before DiD computations
      ## Set treatment year = 0 for controls (necessary for did package to consider "never treated" units)
      ## Compute cumulative deforestation relative to 2000 forest cover (outcome where TE is computed)
      df_did = df_long %>%
        mutate(treatment_year = case_when(group == 1 ~0,
                                          group == 2 ~status_yr), #Set treatment year to 0 for control units (required by did::att_gt)
               time = ifelse(group == 2, yes = year-treatment_year, no = NA),
               .after = status_yr) %>%
        group_by(assetid) %>%
        # mutate(FL_2000_cum = (fc_ha-fc_ha[year == 2000])/fc_ha[year == 2000]*100,
        #        fc_2000 = fc_ha[year == 2000]) %>%
        mutate(FL_2000_cum = case_when(fc_ha[year == 2000] > 0 ~ (fc_ha-fc_ha[year == 2000])/fc_ha[year == 2000]*100, 
                                       TRUE ~ NA)) %>%
        ungroup()
      
      
      ##Average forest cover in 2000 in a pixel, and average share of forest cover in a pixel
      # fc_2000_avg = mean(df_did[df_did$group == 2,]$fc_2000, na.rm = TRUE)
      # per_fc_2000_avg = min(fc_2000_avg/res_ha, 1) #Take the min as in some cases, reported forest cover is higher than pixel area
      
      #Compute dynamic TE with did package. 
      ## Control are "never treated" units, no covariate is added in the regression estimated with doubly-robust method
      ## standard errors are computed with bootstrap, and confidence intervals computed from it.
      ## No clustering is performed as it does not seem relevant in our case (https://blogs.worldbank.org/impactevaluations/when-should-you-cluster-standard-errors-new-wisdom-econometrics-oracle)
      ## Pseudo treatment effect are computed for each pre-treatment year (varying base period)
      
      ##For forest cover (ha and %)
      ### treatment effect computation
      fc_attgt = did::att_gt(yname = "fc_ha",
                             gname = "treatment_year",
                             idname = "assetid",
                             tname = "year",
                             control_group = "nevertreated", #Thsi corresponds to control pixels as defined in the matching , with treatment year set to 0
                             xformla = ~1,
                             alp = alpha, #For 95% confidence interval
                             allow_unbalanced_panel = TRUE, #Ensure no unit is dropped, though every pixel should have data for all years in the period
                             bstrap=TRUE, #Compute bootstrap CI
                             biters = 1000, #The number of bootstrap iteration, 1000 is default
                             cband = TRUE, #Compute CI
                             clustervars = NULL, #No clustering seems relevant to me 
                             base_period = "varying",
                             data = df_did,
                             print_details = F)
      ##For change in deforestation rate (percentage points)
      fl_attgt = did::att_gt(yname = "FL_2000_cum",
                             gname = "treatment_year",
                             idname = "assetid",
                             tname = "year",
                             control_group = "nevertreated", #Thsi corresponds to control pixels as defined in the matching , with treatment year set to 0
                             xformla = ~1,
                             alp = alpha, #For 95% confidence interval
                             allow_unbalanced_panel = TRUE, #Ensure no unit is dropped, though every pixel should have data for all years in the period
                             bstrap=TRUE, #Compute bootstrap CI
                             biters = 1000, #The number of bootstrap iteration, 1000 is default
                             cband = TRUE, #Compute CI
                             clustervars = NULL, #No clustering seems relevant to me
                             base_period = "varying",
                             data = df_did,
                             print_details = F)
      
      
      ### Report results in a dataframe
      ### The treatment effect computed is at pixel level (avoided deforestation in a pixel, in ha)
      ### This treatment effect is aggregated to PA by multiplying treatment effect by the number of pixel in the PA. It is also expressed in percentage of pixel area (avoided deforestation in share of pixel area)
      ### confidence intervals (at pixel level) are computed from bootstrap standard errors after a coefficient is applied.
      ### This computation takes the one from did:::summary.MP function, line 15 and 16. 
      ### They are multiplied by the number of pixels to compute CI for treatment effect at PA level 
      ### They are divided by the pixel area to compute CI for treatment effect in percentage of pixel area
      df_fc_attgt = data.frame("treatment_year" = fc_attgt$group,
                               "year" = fc_attgt$t,
                               "att_pix" = fc_attgt$att,
                               "c" = fc_attgt$c,
                               "se" = fc_attgt$se,  
                               "n" = fc_attgt$n) %>%
        #Compute treatment effect at PA level and in share of pixel area
        ## att_pa : the total avoided deforestation is the avoided deforestation in ha in a given pixel, multiplied by the number of pixel in the PA.
        ## att_per : avoided deforestation in a pixel, as a share of average forest cover in 2000 in matched treated. Can be extrapolated to full PA in principle (avoided deforestation in share of 2000 forest cover)
        mutate(att_pa = att_pix*n_pix_pa,
               att_per = att_pix/avgFC_2000_m*100) %>% 
        #Compute time relative to treatment year
        mutate(time = year - treatment_year,
               .before = year) %>%
        #Compute confidence intervals
        mutate(cband_lower_pix = round(att_pix-c*se, 4),
               cband_upper_pix = round(att_pix+c*se, 4),
               cband_lower_pa = cband_lower_pix*n_pix_pa,
               cband_upper_pa = cband_upper_pix*n_pix_pa,
               cband_lower_per = cband_lower_pix/avgFC_2000_m*100,
               cband_upper_per = cband_upper_pix/avgFC_2000_m*100,
               sig = sign(cband_lower_pix) == sign(cband_upper_pix),
               sig_5 = ifelse(max(time) >=5, yes = sig[time == 5] == TRUE, no = NA),
               sig_10 = ifelse(max(time) >= 10, yes = sig[time == 10] == TRUE, no = NA),
               sig_end = sig[time == max(time)] == TRUE,
               alpha = alpha) %>%
        #Add relevant information
        mutate(region = region.name,
               country_en = country.name,
               iso3 = country.iso,
               name_pa = pa.name,
               wdpaid = wdpaid,
               res_ha = res_ha,
               status_wdpa = status.wdpa,
               iucn_cat = iucn.cat,
               iucn_des_en = iucn.des,
               gov_type = gov.wdpa,
               own_type = own.wdpa,
               marine = eco.wdpa,
               funding_year = funding.years,
               funding_year_list = list.funding.years,
               .before = "treatment_year")
      # Same for treatment effect expressed as a change in deforestation rate
      df_fl_attgt = data.frame("treatment_year" = fl_attgt$group,
                               "year" = fl_attgt$t,
                               "att" = fl_attgt$att,
                               "c" = fl_attgt$c,
                               "se" = fl_attgt$se,
                               "n" = fl_attgt$n) %>%
        #Compute time relative to treatment year
        mutate(time = year - treatment_year,
               .before = year) %>%
        mutate(cband_lower = round(att-c*se, 4),
               cband_upper = round(att+c*se, 4),
               sig = sign(cband_lower) == sign(cband_upper),
               sig_5 = ifelse(max(time) >=5, yes = sig[time == 5] == TRUE, no = NA),
               sig_10 = ifelse(max(time) >= 10, yes = sig[time == 10] == TRUE, no = NA),
               sig_end = sig[time == max(time)] == TRUE,
               alpha = alpha) %>%
        #Compute time relative to treatment year
        mutate(time = year - treatment_year,
               .before = year) %>%
        #Add relevant information
        mutate(region = region.name,
               country_en = country.name,
               iso3 = country.iso,
               name_pa = pa.name,
               wdpaid = wdpaid,
               res_ha = res_ha,
               status_wdpa = status.wdpa,
               iucn_cat = iucn.cat,
               iucn_des_en = iucn.des,
               gov_type = gov.wdpa,
               own_type = own.wdpa,
               marine = eco.wdpa,
               funding_year = funding.years,
               funding_year_list = list.funding.years,
               .before = "treatment_year")
      
      ###Plot results
      ## treatment effect : avoided deforestation at pixel level (in ha)
      fig_att_pix = ggplot(data = df_fc_attgt,
                           aes(x = time, y = att_pix)) %>%
        + geom_line(color = "#08519C") %>%
        + geom_point(color = "#08519C") %>%
        + geom_ribbon(aes(ymin = cband_lower_pix, ymax = cband_upper_pix),
                      alpha=0.1, fill = "#FB6A4A", color = "black", linetype = "dotted") %>%
        + labs(title = ifelse(is_m == TRUE, 
                              yes = "Deforestation avoided in a pixel,on average (matched)",
                              no = "Deforestation avoided in a pixel,on average (unmatched)"),
               subtitle = paste0(pa.name, ", ", country.name, ", implemented in ", treatment.year),
               caption = paste("WDPA ID :", wdpa_id, "|", format(area_ha, big.mark = ","), "ha |", "Pixel resolution :", res_ha, "ha", "\nRibbon represents", (1-alpha)*100, "% confidence interval.\nTreatment effect is interpreted as the deforestation avoided at pixel level in hectare, due to the conservation program.\nA negative effect means the conservation program has caused higher deforestation."),
               y = "Area (ha)",
               x = "Year relative to treatment (t = 0)") %>%
        + scale_x_continuous(breaks=seq(min(df_fc_attgt$time),max(df_fc_attgt$time),by=1)) %>%
        + theme_minimal() %>%
        + theme(
          axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
          axis.text=element_text(size=11, color = "black"),
          axis.title=element_text(size=14, color = "black", face = "plain"),
          
          plot.caption = element_text(hjust = 0),
          plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
          plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
          
          strip.text = element_blank(),
          
          #legend.position = "bottom",
          legend.title = element_blank(),
          legend.text=element_text(size=10),
          #legend.spacing.x = unit(1.0, 'cm'),
          legend.spacing.y = unit(0.75, 'cm'),
          legend.key.size = unit(2, 'line'),
          
          panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
          panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
          panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
          panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
        )
      
      # treatment effect : avoided deforestation in terms of 2000 forest cover
      fig_att_per = ggplot(data = df_fc_attgt,
                           aes(x = time, y = att_per)) %>%
        + geom_line(color = "#08519C") %>%
        + geom_point(color = "#08519C") %>%
        + geom_ribbon(aes(ymin = cband_lower_per, ymax = cband_upper_per),
                      alpha=0.1, fill = "#FB6A4A", color = "black", linetype = "dotted") %>%
        + labs(title = ifelse(is_m == TRUE, 
                              yes = "Average deforestation avoided relative to 2000 forest cover (matched)",
                              no = "Average deforestation avoided relative to 2000 forest cover (unmatched)"),
               subtitle = paste0(pa.name, ", ", country.name, ", implemented in ", treatment.year),
               caption = paste("WDPA ID :", wdpa_id, "|", format(area_ha, big.mark = ","), "ha |", "Pixel resolution :", res_ha, "ha", "\nRibbon represents", (1-alpha)*100, "% confidence interval.\nTreatment effect is interpreted as the deforestation avoided in percentage of 2000 forest cover.\nA negative effect means the conservation program has caused higher deforestation."),
               y = "%",
               x = "Year relative to treatment (t = 0)") %>%
        + scale_x_continuous(breaks=seq(min(df_fc_attgt$time),max(df_fc_attgt$time),by=1)) %>%
        + theme_minimal() %>%
        + theme(
          axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
          axis.text=element_text(size=11, color = "black"),
          axis.title=element_text(size=14, color = "black", face = "plain"),
          
          plot.caption = element_text(hjust = 0),
          plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
          plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
          
          strip.text = element_blank(),
          
          #legend.position = "bottom",
          legend.title = element_blank(),
          legend.text=element_text(size=10),
          #legend.spacing.x = unit(1.0, 'cm'),
          legend.spacing.y = unit(0.75, 'cm'),
          legend.key.size = unit(2, 'line'),
          
          panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
          panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
          panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
          panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
        )
      
      # treatment effect : avoided deforestation in the PA
      fig_att_pa = ggplot(data = df_fc_attgt,
                          aes(x = time, y = att_pa)) %>%
        + geom_line(color = "#08519C") %>%
        + geom_point(color = "#08519C") %>%
        + geom_ribbon(aes(ymin = cband_lower_pa, ymax = cband_upper_pa),
                      alpha=0.1, fill = "#FB6A4A", color = "black", linetype = "dotted") %>%
        + labs(title = ifelse(is_m == TRUE, 
                              yes = "Total deforestation avoided (matched)",
                              no = "Total deforestation avoided (unmatched)"),
               subtitle = paste0(pa.name, ", ", country.name, ", implemented in ", treatment.year),
               caption = paste("WDPA ID :", wdpa_id, "|", format(area_ha, big.mark = ","), "ha |", "Pixel resolution :", res_ha, "ha",  "\nRibbon represents", (1-alpha)*100, "% confidence interval.\nTreatment effect is interpreted as the total deforestation avoided in the protected areas, in hectare (ha).\nThis measure is an extrapolation to the full protected area of average avoided deforestation at pixel level.\nA negative effect means the conservation program has caused higher deforestation."),
               y = "Forest area (ha)",
               x = "Year relative to treatment (t = 0)") %>%
        + scale_x_continuous(breaks=seq(min(df_fc_attgt$time),max(df_fc_attgt$time),by=1)) %>%
        + theme_minimal() %>%
        + theme(
          axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
          axis.text=element_text(size=11, color = "black"),
          axis.title=element_text(size=14, color = "black", face = "plain"),
          
          plot.caption = element_text(hjust = 0),
          plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
          plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
          
          strip.text = element_blank(),
          
          #legend.position = "bottom",
          legend.title = element_blank(),
          legend.text=element_text(size=10),
          #legend.spacing.x = unit(1.0, 'cm'),
          legend.spacing.y = unit(0.75, 'cm'),
          legend.key.size = unit(2, 'line'),
          
          panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
          panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
          panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
          panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
        )
      
      
      # treatment effect : change in deforestation rate
      fig_fl_att = ggplot(data = df_fl_attgt,
                          aes(x = time, y = att)) %>%
        + geom_line(color = "#08519C") %>%
        + geom_point(color = "#08519C") %>%
        + geom_ribbon(aes(ymin = cband_lower, ymax = cband_upper),
                      alpha=0.1, fill = "#FB6A4A", color = "black", linetype = "dotted") %>%
        + labs(title = "Effect of the conservation on the deforestation rate, relative to 2000",
               subtitle = paste0(pa.name, ", ", country.name, ", implemented in ", treatment.year),
               caption = paste("WDPA ID :", wdpa_id, "|", format(area_ha, big.mark = ","), "ha |", "Pixel resolution :", res_ha, "ha", "\nRibbon represents ", (1-alpha)*100, " % confidence interval.\nTreatment effect is interpreted as the reduction of cumulated deforestation rate (relative to 2000 forest cover) in percentage points (pp).\nA negative effect means the conservation program has caused higher deforestation."),
               y = "Reduction of deforestation (p.p)",
               x = "Year relative to treatment (t = 0)") %>%
        + scale_x_continuous(breaks=seq(min(df_fc_attgt$time),max(df_fc_attgt$time),by=1)) %>%
        + theme_minimal() %>%
        + theme(
          axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
          axis.text=element_text(size=11, color = "black"),
          axis.title=element_text(size=14, color = "black", face = "plain"),
          
          plot.caption = element_text(hjust = 0),
          plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
          plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
          
          strip.text = element_blank(),
          
          #legend.position = "bottom",
          legend.title = element_blank(),
          legend.text=element_text(size=10),
          #legend.spacing.x = unit(1.0, 'cm'),
          legend.spacing.y = unit(0.75, 'cm'),
          legend.key.size = unit(2, 'line'),
          
          panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
          panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
          panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
          panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
        )
      
      
      
      ##Saving plots
      tmp = paste(tempdir(), "fig", sep = "/")
      
      ggsave(ifelse(is_m == TRUE,
                    yes = paste(tmp, paste0("fig_att_pix_", iso, "_", wdpaid, "_m", ".png"), sep = "/"),
                    no = paste(tmp, paste0("fig_att_pix_", iso, "_", wdpaid, "_unm", ".png"), sep = "/")),
             plot = fig_att_pix,
             device = "png",
             height = 6, width = 9)
      ggsave(ifelse(is_m == TRUE,
                    yes = paste(tmp, paste0("fig_att_pa_", iso, "_", wdpaid, "_m", ".png"), sep = "/"),
                    no = paste(tmp, paste0("fig_att_pa_", iso, "_", wdpaid, "_unm", ".png"), sep = "/")),
             plot = fig_att_pa,
             device = "png",
             height = 6, width = 9)
      ggsave(ifelse(is_m == TRUE,
                    yes = paste(tmp, paste0("fig_att_per_", iso, "_", wdpaid, "_m", ".png"), sep = "/"),
                    no = paste(tmp, paste0("fig_att_per_", iso, "_", wdpaid, "_unm", ".png"), sep = "/")),
             plot = fig_att_per,
             device = "png",
             height = 6, width = 9)
      ggsave(paste(tmp, paste0("fig_fl_att_", iso, "_", wdpaid, ".png"), sep = "/"),
             plot = fig_fl_att,
             device = "png",
             height = 6, width = 9)
      
      files <- list.files(tmp, full.names = TRUE)
      ##Add each file in the bucket (same foler for every file in the temp)
      for(f in files) 
      {
        cat("Uploading file", paste0("'", f, "'"), "\n")
        aws.s3::put_object(file = f, 
                           bucket = paste("projet-afd-eva-ap", save_dir, iso, wdpaid, sep = "/"),
                           region = "", show_progress = TRUE)
      }
      do.call(file.remove, list(list.files(tmp, full.names = TRUE)))
      
      
      
      #Return outputs
      return(list("df_fc_att" = df_fc_attgt, "df_fl_att"  = df_fl_attgt, "is_ok" = TRUE))
      
    },
    
    error = function(e)
    {
      print(e)
      #cat(paste("Error while computing/plotting DiD :\n", e, "\n"), file = log, append = TRUE)
      print(paste("Error while computing/plotting DiD :\n", e, "\n"))
      return(list("is_ok" = FALSE))
    }
    
  )
  
  return(output)
  
  #TEST : is treatment effect computed by the did package coherent with manual computations ? 
  # --> YES :D
  # test = df_did %>% 
  #   group_by(group, year) %>%
  #   summarize(avgFL_2000_cum = mean(FL_2000_cum, na.rm = TRUE),
  #             avgFC_ha = mean(fc_ha, na.rm = TRUE)) %>%
  #   ungroup() %>%
  #   mutate(fc_te2 = (avgFC_ha[year == 2009 & group == 2] - avgFC_ha[year == 2006 & group == 2]) - (avgFC_ha[year == 2009 & group == 1] - avgFC_ha[year == 2006 & group == 1]),
  #          fl_te2 = (avgFL_2000_cum[year == 2009 & group == 2] - avgFL_2000_cum[year == 2006 & group == 2]) - (avgFL_2000_cum[year == 2009 & group == 1] - avgFL_2000_cum[year == 2006 & group == 1]))
  # 
  
  
}


#Plot the forest cover loss with 2000 as a base year in treated and control units, before and after matching
##INPUTS
### iso : the iso3 code for the country considered
### wdpaid : the WDPAID of the PA considered
### data_pa : dataset with information on protected areas, and especially their surfaces
### alpha : the threshold for confidence interval
### save_dir : the saving directory in the remote storage
### load_dir : the loading directory in the remote storage
### ext_input : the extension of input dataframe
## DATA SAVED
### Plot of forest cover loss with 2000 as a base year in treated and control units, before and after matching
fn_plot_forest_loss = function(iso, wdpaid, data_pa, alpha, load_dir, ext_input, save_dir)
{
  
  #Loading matched and unmatched data frames
  df_long_m_raw = s3read_using(data.table::fread,
                               object = paste0(load_dir, "/", iso, "/", wdpaid, "/", paste0("matched_long", "_", iso, "_", wdpaid, ext_input)),
                               bucket = "projet-afd-eva-ap",
                               opts = list("region" = "")) %>%
    dplyr::select(c(region, iso3, wdpaid, group, assetid, status_yr, year_funding_first, year_funding_all, res_m, year, var, fc_ha))
  
  df_long_unm_raw = s3read_using(data.table::fread,
                                 object = paste0(load_dir, "/", iso, "/", wdpaid, "/", paste0("unmatched_long", "_", iso, "_", wdpaid, ext_input)),
                                 bucket = "projet-afd-eva-ap",
                                 opts = list("region" = "")) %>%
    dplyr::select(c(region, iso3, wdpaid, group, assetid, status_yr, year_funding_first, year_funding_all, year, res_m, var, fc_ha))
  
  wdpa_id = wdpaid
  #Extract relevant information
  ##Spatial resolution of pixels res_m and define pixel area in ha
  res_m = unique(df_long_m_raw$res_m)
  res_ha = res_m^2*1e-4
  
  ##treatment year
  treatment.year = df_long_m_raw %>% 
    filter(group == 2) %>% 
    slice(1)
  treatment.year = treatment.year$status_yr
  
  ##funding years
  funding.years = df_long_m_raw %>% 
    filter(group == 2) %>% 
    slice(1)
  funding.years = funding.years$year_funding_first
  #funding.years = as.numeric(unlist(strsplit(funding.years$year_funding_all, split = ",")))
  
  ##country iso
  country.iso = df_long_m_raw %>% 
    filter(group == 2) %>% 
    slice(1)
  country.iso = country.iso$iso3
  
  ##region name
  region.name = df_long_m_raw %>% 
    filter(group == 2) %>% 
    slice(1)
  region.name = region.name$region
  
  ##Area of the PA and PA/country name
  area_ha = data_pa[data_pa$wdpaid == wdpa_id,]$area_km2*100
  country.name = data_pa %>% 
    filter(iso3 == iso) %>% 
    slice(1)
  country.name = country.name$country_en
  pa.name = data_pa %>% 
    filter(wdpaid == wdpa_id) %>% 
    slice(1)
  pa.name = pa.name$name_pa
  
  
  #Forest cover loss is computed for each pixel relative to 2000, then average forest cover evolution and loss is computed for treated and controls
  df_long_m = df_long_m_raw %>%
    #Compute forest loss relative to 2000 in ha for each pixel
    group_by(assetid) %>%
    mutate(fc_rel00_ha = fc_ha - fc_ha[year == 2000],
           .after = "fc_ha") %>%
    ungroup() %>%
    #Compute average forest cover and forest cover loss relative to 2000 for each group, year
    group_by(group, year) %>%
    summarise(n= n(),
              avgfc_ha = mean(fc_ha, na.rm = TRUE),
              sdfc_ha = sd(fc_ha, na.rm = TRUE),
              avgfc_rel00_ha = mean(fc_rel00_ha, na.rm = TRUE),
              sdfc_rel00_ha = sd(fc_rel00_ha, na.rm = TRUE),
              fc_ha_ci_upper = avgfc_ha + qt((1-alpha)/2,df=n-1)*sdfc_ha/sqrt(n),
              fc_ha_ci_lower = avgfc_ha - qt((1-alpha)/2,df=n-1)*sdfc_ha/sqrt(n),
              fc_rel00_ha_ci_upper = avgfc_rel00_ha + qt((1-alpha)/2,df=n-1)*sdfc_rel00_ha/sqrt(n),
              fc_rel00_ha_ci_lower = avgfc_rel00_ha - qt((1-alpha)/2,df=n-1)*sdfc_rel00_ha/sqrt(n),
              matched = T) %>%
    #Compute total forest cover and forest loss relative to 2000, knowing area of the PA and average forest share in a pixel in 2000
    #CI are computed at 95% confidence level
    ungroup() %>%
    mutate(#per_fc_2000_avg = min(fc_ha[year == 2000]/res_ha, 1),
      #fc_tot_ha = fc_ha*(area_ha*per_fc_2000_avg/res_ha),
      fc_tot_ha = avgfc_ha*(area_ha/res_ha),
      #fc_tot_rel00_ha = avgfc_rel00_ha*(area_ha*per_fc_2000_avg/res_ha),
      fc_tot_rel00_ha = avgfc_rel00_ha*(area_ha/res_ha),
      fc_tot_ha_ci_upper = fc_ha_ci_upper*(area_ha/res_ha),
      fc_tot_ha_ci_upper = fc_ha_ci_lower*(area_ha/res_ha),
      fc_tot_rel00_ha_ci_upper = fc_rel00_ha_ci_upper*(area_ha/res_ha),
      fc_tot_rel00_ha_ci_lower = fc_rel00_ha_ci_lower*(area_ha/res_ha),
      alpha = alpha)
  
  df_long_unm = df_long_unm_raw %>%
    #Compute forest loss relative to 2000 in ha for each pixel
    group_by(assetid) %>%
    mutate(fc_rel00_ha = fc_ha - fc_ha[year == 2000],
           .after = "fc_ha") %>%
    ungroup() %>%
    #Compute average forest cover and forest cover loss relative to 2000 for each group, year
    group_by(group, year) %>%
    summarise(n= n(),
              avgfc_ha = mean(fc_ha, na.rm = TRUE),
              sdfc_ha = sd(fc_ha, na.rm = TRUE),
              avgfc_rel00_ha = mean(fc_rel00_ha, na.rm = TRUE),
              sdfc_rel00_ha = sd(fc_rel00_ha, na.rm = TRUE),
              fc_ha_ci_upper = avgfc_ha + qt((1-alpha)/2,df=n-1)*sdfc_ha/sqrt(n),
              fc_ha_ci_lower = avgfc_ha - qt((1-alpha)/2,df=n-1)*sdfc_ha/sqrt(n),
              fc_rel00_ha_ci_upper = avgfc_rel00_ha + qt((1-alpha)/2,df=n-1)*sdfc_rel00_ha/sqrt(n),
              fc_rel00_ha_ci_lower = avgfc_rel00_ha - qt((1-alpha)/2,df=n-1)*sdfc_rel00_ha/sqrt(n),
              matched = F) %>%
    #Compute total forest cover and forest loss relative to 2000, knowing area of the PA and average forest share in a pixel in 2000
    #CI are computed at 95% confidence level
    ungroup() %>%
    mutate(#per_fc_2000_avg = min(fc_ha[year == 2000]/res_ha, 1),
      #fc_tot_ha = fc_ha*(area_ha*per_fc_2000_avg/res_ha),
      fc_tot_ha = avgfc_ha*(area_ha/res_ha),
      #fc_tot_rel00_ha = avgfc_rel00_ha*(area_ha*per_fc_2000_avg/res_ha),
      fc_tot_rel00_ha = avgfc_rel00_ha*(area_ha/res_ha),
      fc_tot_ha_ci_upper = fc_ha_ci_upper*(area_ha/res_ha),
      fc_tot_ha_ci_upper = fc_ha_ci_lower*(area_ha/res_ha),
      fc_tot_rel00_ha_ci_upper = fc_rel00_ha_ci_upper*(area_ha/res_ha),
      fc_tot_rel00_ha_ci_lower = fc_rel00_ha_ci_lower*(area_ha/res_ha),
      alpha = alpha)
  
  
  #Define plotting dataset
  df_plot = rbind(df_long_m, df_long_unm) %>%
    mutate(group = case_when(group == 1 ~"Control",
                             group == 2 ~"Treated"),
           region = region.name,
           country_en = country.name,
           iso3 = country.iso,
           wdpaid = wdpaid, 
           name_pa = pa.name,
           area_ha = area_ha)
  
  #The period where deforestation is plotted
  year.max = max(df_long_m$year)
  
  #Plot
  fct.labs <- c("Before Matching", "After Matching")
  names(fct.labs) <- c(FALSE, TRUE)
  
  fig = ggplot(data = filter(df_plot, year == year.max),
               aes(y = abs(fc_tot_rel00_ha), fill = as.factor(group), x = group)) %>%
    + geom_bar(position =  position_dodge(width = 0.8), stat = "identity", show.legend = FALSE) %>% 
    + geom_errorbar(aes(ymax=abs(fc_tot_rel00_ha_ci_upper), ymin=abs(fc_tot_rel00_ha_ci_lower)), width=0.4, colour="grey60", alpha=0.9, size=1.3) %>%
    + geom_label(aes(label = format(round(abs(fc_tot_rel00_ha), 0), big.mark = ","), y = abs(fc_tot_rel00_ha)), 
                 color = "black",
                 show.legend = FALSE) %>%
    + scale_fill_brewer(name = "Group", palette = "Blues") %>%
    + labs(x = "",
           y = "Forest cover loss (ha)",
           title = paste("Average area deforested between 2000 and", year.max),
           subtitle = paste("WDPA ID", wdpaid, "in", country.iso, ",implemented in", treatment.year, "and covering", format(area_ha, big.mark = ","), "ha"),
           caption = paste((1-alpha)*100, "% confidence intervals.")) %>%
    + facet_wrap(~matched,
                 labeller = labeller(matched = fct.labs))  %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_text(color = "black", size = 12),
      
      #legend.position = "bottom",
      #legend.title = element_blank(),
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  
  ##Saving plots
  tmp = paste(tempdir(), "fig", sep = "/")
  
  ggsave(paste(tmp, paste0("fig_fl_2000_", year.max, "_m_unm_", iso, "_", wdpaid, ".png"), sep = "/"),
         plot = fig,
         device = "png",
         height = 6, width = 9)
  
  files <- list.files(tmp, full.names = TRUE)
  ##Add each file in the bucket (same foler for every file in the temp)
  for(f in files) 
  {
    cat("Uploading file", paste0("'", f, "'"), "\n")
    aws.s3::put_object(file = f, 
                       bucket = paste("projet-afd-eva-ap", save_dir, iso, wdpaid, sep = "/"),
                       region = "", show_progress = TRUE)
  }
  do.call(file.remove, list(list.files(tmp, full.names = TRUE)))
  
  return(df_plot)
}


# Plotting the treatment effect of each protected area analyzed in the same graph. This function suits for AFD supported protected areas : it includes funding information to the table and figures
## INPUTS
### df_fc_att : a dataset with treatment effects for each protected area in the sample, expressed as avoided deforestation (hectare)
### df_fl_att : a dataset with treatment effects for each protected area in the sample, expressed as change in deforestation rate
### alpha : the threshold for confidence interval
### save_dir : the saving directory in the remote storage
## DATA SAVED
### Tables and figures : treatment effects computed for each protected area in the sample, expressed as avoided deforestaion (hectare and percentage of 2000 forest cover) and change in deforestation rate.
fn_plot_att_afd = function(df_fc_att, df_fl_att, alpha = alpha, save_dir)
{
  
  #list of PAs and two time periods
  list_ctry_plot = df_fc_att %>%
    dplyr::select(iso3, country_en, wdpaid, iucn_cat) %>%
    unique() %>%
    group_by(iso3, country_en, wdpaid) %>%
    summarize(time = c(5, 10),
              iucn_wolf = case_when(iucn_cat %in% c("I", "II", "III", "IV") ~ "Strict",
                                    iucn_cat %in% c("V", "VI") ~ "Non strict",
                                    grepl("not", iucn_cat, ignore.case = TRUE) ~ "Unknown")) %>%
    ungroup()
  
  #treatment effect for each wdpa (some have not on the two time periods)
  temp_fc = df_fc_att %>%
    dplyr::select(c(region, iso3, country_en, wdpaid, name_pa, iucn_cat, treatment_year, time, year, att_per, cband_lower_per, cband_upper_per, att_pa, cband_lower_pa, cband_upper_pa)) %>%
    mutate(sig_pa = sign(cband_lower_pa) == sign(cband_upper_pa),
           sig_per = sign(cband_lower_per) == sign(cband_upper_per)) %>%
    filter(time %in% c(5, 10)) 
  temp_fl = df_fl_att %>%
    dplyr::select(c(region, iso3, country_en, wdpaid, name_pa, iucn_cat, treatment_year, time, year, att, cband_lower, cband_upper)) %>%
    mutate(sig = sign(cband_lower) == sign(cband_upper)) %>%
    filter(time %in% c(5, 10)) 
  
  #Att for each WDPAID, for each period (NA if no value)
  df_plot_fc_att = left_join(list_ctry_plot, temp_fc, by = c("iso3", "country_en", "wdpaid", "time")) %>%
    group_by(time, country_en) %>%
    arrange(country_en) %>%
    mutate(country_en = paste0(country_en, " (", LETTERS[row_number()], ")")) %>%
    ungroup() 
  df_plot_fl_att = left_join(list_ctry_plot, temp_fl, by = c("iso3", "country_en", "wdpaid", "time"))%>%
    group_by(time, country_en) %>%
    arrange(country_en) %>%
    mutate(country_en = paste0(country_en, " (", LETTERS[row_number()], ")")) %>%
    ungroup()
  
  #Plots
  names = c(`5` = "5 years after treatment",
            `10` = "10 years after treatment",
            `Strict` = "Strict\nIUCN cat. I-IV",
            `Non strict` = "Non strict\nIUCN V-VI",
            `Unknown` = "Unknown")
  
  ## Att in share of 2000 forest cover
  fig_att_per = ggplot(df_plot_fc_att, 
                       aes(x = att_per, 
                           y = factor(country_en, levels = unique(rev(sort(country_en)))),
                           xmin = cband_lower_per, xmax = cband_upper_per)) %>%
    + geom_point(aes(color = sig_per)) %>%
    + geom_vline(xintercept = 0) %>%
    + geom_errorbarh(aes(color = sig_per)) %>% 
    + scale_color_discrete(name = paste0("Significance\n(", (1-alpha)*100, "% level)"),
                           na.translate = F) %>%
    # + scale_x_continuous(breaks=seq(min(df_plot_fc_att$att_per, na.rm = TRUE),max(df_plot_fc_att$att_per, na.rm = TRUE),by=1)) %>%
    + facet_grid(~time,scales="free", space="free",  labeller= as_labeller(names)) %>%
    + labs(title = "Deforestation avoided relative to 2000 forest cover",
           x = "%",
           y = "") %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_text(color = "black", size = 12),
      strip.clip = "off",
      panel.spacing = unit(2, "lines"),
      
      #legend.position = "bottom",
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      #legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  
  fig_att_per_iucn = ggplot(df_plot_fc_att, 
                       aes(x = att_per, 
                           y = factor(country_en, levels = unique(rev(sort(country_en)))),
                           xmin = cband_lower_per, xmax = cband_upper_per)) %>%
    + geom_point(aes(color = sig_per)) %>%
    + geom_vline(xintercept = 0) %>%
    + geom_errorbarh(aes(color = sig_per)) %>% 
    + scale_color_discrete(name = paste0("Significance\n(", (1-alpha)*100, "% level)"),
                           na.translate = F) %>%
    # + scale_x_continuous(breaks=seq(min(df_plot_fc_att$att_per, na.rm = TRUE),max(df_plot_fc_att$att_per, na.rm = TRUE),by=1)) %>%
    + facet_grid(iucn_wolf~time,scales="free", space="free",  labeller= as_labeller(names)) %>%
    + labs(title = "Deforestation avoided relative to 2000 forest cover",
           x = "%",
           y = "") %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_text(color = "black", size = 12),
      strip.clip = "off",
      panel.spacing = unit(2, "lines"),
      
      #legend.position = "bottom",
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      #legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )

    
  ##treatment effect : total deforestation avoided
  fig_att_pa = ggplot(df_plot_fc_att, 
                      aes(x = att_pa, 
                          y = factor(country_en, levels = unique(rev(sort(country_en)))),
                          xmin = cband_lower_pa, xmax = cband_upper_pa)) %>%
    + geom_point(aes(color = sig_pa)) %>%
    + geom_vline(xintercept = 0) %>%
    + geom_errorbarh(aes(color = sig_pa)) %>% 
    + scale_color_discrete(name = paste0("Significance\n(", (1-alpha)*100, "% level)"),
                           na.translate = F) %>%
    + facet_grid(~time,scales="free", space="free",  labeller= as_labeller(names)) %>%
    + labs(title = "Total deforestation avoided",
           x = "ha",
           y = "") %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_text(color = "black", size = 12),
      strip.clip = "off",
      panel.spacing = unit(2, "lines"),
      
      #legend.position = "bottom",
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      #legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  
  fig_att_pa_iucn = ggplot(df_plot_fc_att, 
                      aes(x = att_pa, 
                          y = factor(country_en, levels = unique(rev(sort(country_en)))),
                          xmin = cband_lower_pa, xmax = cband_upper_pa)) %>%
    + geom_point(aes(color = sig_pa)) %>%
    + geom_vline(xintercept = 0) %>%
    + geom_errorbarh(aes(color = sig_pa)) %>% 
    + scale_color_discrete(name = paste0("Significance\n(", (1-alpha)*100, "% level)"),
                           na.translate = F) %>%
    + facet_grid(iucn_wolf~time,scales="free", space="free",  labeller= as_labeller(names)) %>%
    + labs(title = "Total deforestation avoided",
           x = "ha",
           y = "") %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_text(color = "black", size = 12),
      strip.clip = "off",
      panel.spacing = unit(2, "lines"),
      
      #legend.position = "bottom",
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      #legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  
  ##treatment effect : avoided deforestation in percentage points
  fig_att_fl = ggplot(df_plot_fl_att, 
                      aes(x = att, 
                          y = factor(country_en, levels = unique(rev(sort(country_en)))),
                          xmin = cband_lower, xmax = cband_upper)) %>%
    + geom_point(aes(color = sig)) %>%
    + geom_vline(xintercept = 0) %>%
    + geom_errorbarh(aes(color = sig)) %>% 
    + scale_color_discrete(name = paste0("Significance\n(", (1-alpha)*100, "% level)"),
                           na.translate = F) %>%
    + facet_grid(~time,scales="free", space="free",  labeller= as_labeller(names)) %>%
    + labs(title = "Reduction of deforestation due to the conservation",
           x = "p.p.",
           y = "") %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_text(color = "black", size = 12),
      strip.clip = "off",
      panel.spacing = unit(2, "lines"),
      
      #legend.position = "bottom",
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      #legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  
  fig_att_fl_iucn = ggplot(df_plot_fl_att, 
                      aes(x = att, 
                          y = factor(country_en, levels = unique(rev(sort(country_en)))),
                          xmin = cband_lower, xmax = cband_upper)) %>%
    + geom_point(aes(color = sig)) %>%
    + geom_vline(xintercept = 0) %>%
    + geom_errorbarh(aes(color = sig)) %>% 
    + scale_color_discrete(name = paste0("Significance\n(", (1-alpha)*100, "% level)"),
                           na.translate = F) %>%
    + facet_grid(iucn_wolf~time, scales="free", space="free",  labeller= as_labeller(names)) %>%
    + labs(title = "Reduction of deforestation due to the conservation",
           x = "p.p.",
           y = "") %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_text(color = "black", size = 12),
      strip.clip = "off",
      panel.spacing = unit(2, "lines"),
      
      #legend.position = "bottom",
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      #legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  
  
  #Tables 
  ## treatment effect : percentage of deforestation avoided
  tbl_fc_att_per = df_fc_att %>%
    mutate(sig_per = case_when(sign(cband_lower_per) == sign(cband_upper_per) ~ "Yes",
                               sign(cband_lower_per) != sign(cband_upper_per) ~ "No"),
           iucn_wolf = case_when(iucn_cat %in% c("I", "II", "III", "IV") ~ "Strict",
                                 iucn_cat %in% c("V", "VI") ~ "Non strict",
                                 grepl("not", iucn_cat, ignore.case = TRUE) ~ "Unknown"),
           dept_report = case_when(dept_report == "Léa Poulin,Pierre-Yves Durand,Ingrid Dallmann" ~ "Unknown",
                                   TRUE ~ dept_report),
           kfw = case_when(kfw == TRUE ~ "Yes", kfw == FALSE ~ "No"),
           ffem = case_when(ffem == TRUE ~ "Yes", ffem == FALSE ~ "No"),
           funding_year_list = case_when(is.na(funding_year_list) == TRUE ~ "Unknown",
                                         TRUE ~ funding_year_list),
           name_pa = case_when(nchar(name_pa) <= 25 ~ stri_trans_general(name_pa, id = "Latin-ASCII"),
                               nchar(name_pa) > 25 ~ stri_trans_general(paste0(substr(name_pa, 1, 25), "..."),  id = "Latin-ASCII"))
    ) %>%
    dplyr::select(c(name_pa, id_projet, dept_report, country_en, treatment_year, funding_year_list, fund_type, kfw, ffem, iucn_wolf, gov_type, time, att_per, sig_per)) %>%
    filter(time %in% c(5, 10)) %>%
    pivot_wider(values_from = c("att_per", "sig_per"), names_from = c("time", "time")) %>%
    dplyr::select(c(name_pa, id_projet, dept_report, country_en, treatment_year, funding_year_list, kfw, ffem, iucn_wolf, att_per_5, sig_per_5, att_per_10, sig_per_10)) %>%
    #dplyr::select(c(name_pa, id_projet, dept_report, country_en, treatment_year, funding_year_list, fund_type, kfw, ffem, iucn_wolf, gov_type, att_per_5, sig_per_5, att_per_10, sig_per_10)) %>%
    mutate(across(.cols = starts_with(c("att", "sig")),
                  .fns = \(x) case_when(is.na(x) == TRUE ~ "/", TRUE ~ as.character(format(x, digit = 1))))) %>%
    rename("Effect (5 y., %)" = "att_per_5",
           "Signi. (5 y.)" = "sig_per_5",
           "Effect (10 y., %)" = "att_per_10",
           "Signi. (10 y.)" = "sig_per_10") 
  # names(tbl_fc_att_per) = c("Name", "Project ID", "Tech. div.", "Country", "Creation", "Funding year", "Type of funding", "KfW", "FFEM", "Protection", 
  #                           "Governance", "Effect (5 y., %)", "Significance (5 y.)","Effect (10 y., %)", "Significance (10 y.)")
  names(tbl_fc_att_per) = c("Name", "Project ID", "Tech. div.", "Country", "Creation", "Funding year", "KfW", "FFEM", "Protection", 
                            "Effect (5 y., %)", "Signi. (5 y.)","Effect (10 y., %)", "Signi. (10 y.)")
  
  # treatment effect : total deforestation avoided 
  tbl_fc_att_pa = df_fc_att %>%
    mutate(sig_pa = case_when(sign(cband_lower_pa) == sign(cband_upper_pa) ~ "Yes",
                              sign(cband_lower_pa) != sign(cband_upper_pa) ~ "No"),
           iucn_wolf = case_when(iucn_cat %in% c("I", "II", "III", "IV") ~ "Strict",
                                 iucn_cat %in% c("V", "VI") ~ "Non strict",
                                 grepl("not", iucn_cat, ignore.case = TRUE) ~ "Unknown"),
           dept_report = case_when(dept_report == "Léa Poulin,Pierre-Yves Durand,Ingrid Dallmann" ~ "Unknown",
                                   TRUE ~ dept_report),
           kfw = case_when(kfw == TRUE ~ "Yes", kfw == FALSE ~ "No"),
           ffem = case_when(ffem == TRUE ~ "Yes", ffem == FALSE ~ "No"),
           funding_year_list = case_when(is.na(funding_year_list) == TRUE ~ "Unknown",
                                         TRUE ~ funding_year_list),
           name_pa = case_when(nchar(name_pa) <= 25 ~ stri_trans_general(name_pa, id = "Latin-ASCII"),
                               nchar(name_pa) > 25 ~ stri_trans_general(paste0(substr(name_pa, 1, 25), "..."),  id = "Latin-ASCII"))
    ) %>%
    dplyr::select(c(name_pa, id_projet, dept_report, country_en, treatment_year, funding_year_list, fund_type, kfw, ffem, iucn_wolf, gov_type, time, att_pa, sig_pa)) %>%
    filter(time %in% c(5, 10)) %>%
    pivot_wider(values_from = c("att_pa", "sig_pa"), names_from = c("time", "time")) %>%
    # dplyr::select(c(name_pa, id_projet, dept_report, country_en, treatment_year, funding_year_list, fund_type, kfw, ffem, iucn_wolf, gov_type, att_pa_5, sig_pa_5, att_pa_10, sig_pa_10)) %>%
    dplyr::select(c(name_pa, id_projet, dept_report, country_en, treatment_year, funding_year_list, kfw, ffem, iucn_wolf, att_pa_5, sig_pa_5, att_pa_10, sig_pa_10)) %>%
    mutate(across(.cols = starts_with(c("att", "sig")),
                  .fns = \(x) case_when(is.na(x) == TRUE ~ "/", TRUE ~ as.character(format(x, digit = 1))))) %>%
    rename("Effect (5 y., %)" = "att_pa_5",
           "Signi. (5 y.)" = "sig_pa_5",
           "Effect (10 y., %)" = "att_pa_10",
           "Signi. (10 y.)" = "sig_pa_10") 
  # names(tbl_fc_att_pa) = c("Name", "Project ID", "Tech. div.", "Country", "Creation", "Funding year", "Type of funding", "KfW", "FFEM", "Protection", 
  #                           "Governance", "Effect (5 y., ha)", "Significance (5 y.)","Effect (10 y., ha)", "Significance (10 y.)")
  names(tbl_fc_att_pa) = c("Name", "Project ID", "Tech. div.", "Country", "Creation", "Funding year", "KfW", "FFEM", "Protection", 
                           "Effect (5 y., ha)", "Signi. (5 y.)","Effect (10 y., ha)", "Signi. (10 y.)")
  
  
  
  #Saving plots
  
  ##Saving plots
  tmp = paste(tempdir(), "fig", sep = "/")
  
  ggsave(paste(tmp, "fig_att_per.png", sep = "/"),
         plot = fig_att_per,
         device = "png",
         height = 6, width = 9)
  
  ggsave(paste(tmp, "fig_att_per_iucn.png", sep = "/"),
         plot = fig_att_per_iucn,
         device = "png",
         height = 6, width = 9)
  
  ggsave(paste(tmp, "fig_att_pa.png", sep = "/"),
         plot = fig_att_pa,
         device = "png",
         height = 6, width = 9)
  
  ggsave(paste(tmp, "fig_att_pa_iucn.png", sep = "/"),
         plot = fig_att_pa_iucn,
         device = "png",
         height = 6, width = 9)
  
  ggsave(paste(tmp, "fig_att_fl.png", sep = "/"),
         plot = fig_att_fl,
         device = "png",
         height = 6, width = 9)
  
  ggsave(paste(tmp, "fig_att_fl_iucn.png", sep = "/"),
         plot = fig_att_fl_iucn,
         device = "png",
         height = 6, width = 9)
  
  print(xtable(tbl_fc_att_pa, 
               type = "latex"),
        file = paste(tmp, "tbl_fc_att_pa.tex", sep = "/"))
  
  print(xtable(tbl_fc_att_per, type = "latex"),
        file = paste(tmp, "tbl_fc_att_per.tex", sep = "/"))
  
  files <- list.files(tmp, full.names = TRUE)
  ##Add each file in the bucket (same foler for every file in the temp)
  for(f in files) 
  {
    cat("Uploading file", paste0("'", f, "'"), "\n")
    aws.s3::put_object(file = f, 
                       bucket = paste("projet-afd-eva-ap", save_dir, sep = "/"),
                       region = "", show_progress = TRUE)
  }
  do.call(file.remove, list(list.files(tmp, full.names = TRUE)))
}


# Plotting the treatment effect of each protected area analyzed in the same graph. This function suits for all protected areas in general, and does not include any information on funding.
## INPUTS
### df_fc_att : a dataset with treatment effects for each protected area in the sample, expressed as avoided deforestation (hectare)
### df_fl_att : a dataset with treatment effects for each protected area in the sample, expressed as change in deforestation rate
### alpha : the threshold for confidence interval
### save_dir : the saving directory in the remote storage
## DATA SAVED
### Tables and figures : treatment effects computed for each protected area in the sample, expressed as avoided deforestaion (hectare and percentage of 2000 forest cover) and change in deforestation rate.
fn_plot_att_general = function(df_fc_att, df_fl_att, list_focus, alpha = alpha, save_dir)
{
  
  #list of PAs and two time periods
  list_ctry_plot = df_fc_att %>%
    dplyr::select(iso3, country_en, wdpaid, name_pa, iucn_cat, gov_type, own_type, treatment_year, status_wdpa) %>%
    unique() %>%
    group_by(iso3, country_en, wdpaid, name_pa) %>%
    summarize(time = c(5, 10),
              iucn_cat = iucn_cat,
              iucn_wolf = case_when(iucn_cat %in% c("I", "II", "III", "IV") ~ "Strict",
                                    iucn_cat %in% c("V", "VI") ~ "Non strict",
                                    grepl("not", iucn_cat, ignore.case = TRUE) ~ "Unknown"),
              treatment_year = treatment_year,
              gov_type = gov_type,
              own_type = own_type,
              status_wdpa = status_wdpa) %>%
    ungroup()
  
  #treatment effect for each wdpa (some have not on the two time periods)
  temp_fc = df_fc_att %>%
    dplyr::select(c(region, iso3, country_en, wdpaid, name_pa, time, year, att_per, cband_lower_per, cband_upper_per, att_pa, cband_lower_pa, cband_upper_pa)) %>%
    mutate(sig_pa = sign(cband_lower_pa) == sign(cband_upper_pa),
           sig_per = sign(cband_lower_per) == sign(cband_upper_per)) %>%
    filter(time %in% c(5, 10)) 
  temp_fl = df_fl_att %>%
    dplyr::select(c(region, iso3, country_en, wdpaid, name_pa, time, year, att, cband_lower, cband_upper)) %>%
    mutate(sig = sign(cband_lower) == sign(cband_upper)) %>%
    filter(time %in% c(5, 10)) 
  
  #Att for each WDPAID, for each period (NA if no value)
  ## For figures
  df_plot_fc_att = left_join(list_ctry_plot, temp_fc, by = c("iso3", "country_en", "wdpaid", "name_pa", "time")) %>%
    mutate(focus = case_when(wdpaid %in% list_focus ~ "focus",
                             !(wdpaid %in% list_focus) ~ "not focus")) %>%
    group_by(time, country_en) %>%
    arrange(country_en, focus) %>%
    mutate(country_en = paste0(country_en, " (", row_number(), ")"),
           n = row_number()) %>%
    ungroup()
  
  df_plot_fl_att = left_join(list_ctry_plot, temp_fl, by = c("iso3", "country_en", "wdpaid", "name_pa", "time"))%>%
    mutate(focus = case_when(wdpaid %in% list_focus ~ "focus",
                             !(wdpaid %in% list_focus) ~ "not focus")) %>%
    group_by(time, country_en) %>%
    arrange(country_en, focus) %>%
    mutate(country_en = paste0(country_en, " (", row_number(), ")"),
           n = row_number()) %>%
    ungroup()
  
  
  #Plots
  names = c(`5` = "5 years after treatment",
            `10` = "10 years after treatment",
            `Strict` = "Strict\nIUCN cat. I-IV",
            `Non strict` = "Non strict\nIUCN V-VI",
            `Unknown` = "Unknown",
            `focus` = "FAPBM funded",
            `not focus` = "Others")
  # df_colors = df_plot_fc_att %>% group_by(n) %>% slice(1)
  # colors = ifelse(df_colors$wdpaid %in% list_focus,"#3182BD","black")
  
  ## Att in share of 2000 forest cover
  fig_att_per = ggplot(df_plot_fc_att, 
                       aes(x = att_per, 
                           y = factor(name_pa, levels = unique(rev(sort(name_pa)))),
                           xmin = cband_lower_per, xmax = cband_upper_per)) %>%
    + geom_point(aes(color = sig_per)) %>%
    + geom_vline(xintercept = 0) %>%
    + geom_errorbarh(aes(color = sig_per)) %>% 
    + scale_color_discrete(name = paste0("Significance\n(", (1-alpha)*100, "% level)"),
                           na.translate = F) %>%
    # + scale_x_continuous(breaks=seq(min(df_plot_fc_att$att_per, na.rm = TRUE),max(df_plot_fc_att$att_per, na.rm = TRUE),by=1)) %>%
    + facet_grid(~time,scales="free", space="free",  labeller= as_labeller(names)) %>%
    + labs(title = "Deforestation avoided relative to 2000 forest cover",
           #caption = "FAPBM funded protected areas are in blue, others are in black.",
           x = "%",
           y = "") %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      #axis.text.y = element_text(color = rev(colors)),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_text(color = "black", size = 12),
      strip.clip = "off",
      panel.spacing = unit(2, "lines"),
      
      #legend.position = "bottom",
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      #legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )

  fig_att_per_focus_others = ggplot(df_plot_fc_att, 
                       aes(x = att_per, 
                           y = factor(name_pa, levels = unique(rev(sort(name_pa)))),
                           xmin = cband_lower_per, xmax = cband_upper_per)) %>%
    + geom_point(aes(color = sig_per)) %>%
    + geom_vline(xintercept = 0) %>%
    + geom_errorbarh(aes(color = sig_per)) %>% 
    + scale_color_discrete(name = paste0("Significance\n(", (1-alpha)*100, "% level)"),
                           na.translate = F) %>%
    # + scale_x_continuous(breaks=seq(min(df_plot_fc_att$att_per, na.rm = TRUE),max(df_plot_fc_att$att_per, na.rm = TRUE),by=1)) %>%
    + facet_grid(focus~time,scales="free", space="free",  labeller= as_labeller(names)) %>%
    + labs(title = "Deforestation avoided relative to 2000 forest cover",
           x = "%",
           y = "") %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_text(color = "black", size = 12),
      strip.clip = "off",
      panel.spacing = unit(2, "lines"),
      
      #legend.position = "bottom",
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      #legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  
  fig_att_per_focus = ggplot(filter(df_plot_fc_att, focus == "focus"),
                                    aes(x = att_per, 
                                        y = factor(name_pa, levels = unique(rev(sort(name_pa)))),
                                        xmin = cband_lower_per, xmax = cband_upper_per)) %>%
    + geom_point(aes(color = sig_per)) %>%
    + geom_vline(xintercept = 0) %>%
    + geom_errorbarh(aes(color = sig_per)) %>% 
    + scale_color_discrete(name = paste0("Significance\n(", (1-alpha)*100, "% level)"),
                           na.translate = F) %>%
    # + scale_x_continuous(breaks=seq(min(df_plot_fc_att$att_per, na.rm = TRUE),max(df_plot_fc_att$att_per, na.rm = TRUE),by=1)) %>%
    + facet_grid(~time,scales="free", space="free",  labeller= as_labeller(names)) %>%
    + labs(title = "Deforestation avoided relative to 2000 forest cover",
           subtitle = "Protected areas funded by the FAPBM only",
           x = "%",
           y = "") %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_text(color = "black", size = 12),
      strip.clip = "off",
      panel.spacing = unit(2, "lines"),
      
      #legend.position = "bottom",
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      #legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  
  
  fig_att_per_iucn = ggplot(df_plot_fc_att, 
                            aes(x = att_per, 
                                y = factor(name_pa, levels = unique(rev(sort(name_pa)))),
                                xmin = cband_lower_per, xmax = cband_upper_per)) %>%
    + geom_point(aes(color = sig_per)) %>%
    + geom_vline(xintercept = 0) %>%
    + geom_errorbarh(aes(color = sig_per)) %>% 
    + scale_color_discrete(name = paste0("Significance\n(", (1-alpha)*100, "% level)"),
                           na.translate = F) %>%
    # + scale_x_continuous(breaks=seq(min(df_plot_fc_att$att_per, na.rm = TRUE),max(df_plot_fc_att$att_per, na.rm = TRUE),by=1)) %>%
    + facet_grid(iucn_wolf~time,scales="free", space="free",  labeller= as_labeller(names)) %>%
    + labs(title = "Deforestation avoided relative to 2000 forest cover",
           x = "%",
           y = "") %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      #axis.text.y = element_text(color = rev(colors)),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_text(color = "black", size = 12),
      strip.clip = "off",
      panel.spacing = unit(2, "lines"),
      
      #legend.position = "bottom",
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      #legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  
  
  ##treatment effect : total deforestation avoided
  fig_att_pa = ggplot(df_plot_fc_att, 
                      aes(x = att_pa, 
                          y = factor(name_pa, levels = unique(rev(sort(name_pa)))),
                          xmin = cband_lower_pa, xmax = cband_upper_pa)) %>%
    + geom_point(aes(color = sig_pa)) %>%
    + geom_vline(xintercept = 0) %>%
    + geom_errorbarh(aes(color = sig_pa)) %>% 
    + scale_color_discrete(name = paste0("Significance\n(", (1-alpha)*100, "% level)"),
                           na.translate = F) %>%
    + facet_grid(~time,scales="free", space="free",  labeller= as_labeller(names)) %>%
    + labs(title = "Total deforestation avoided",
           x = "ha",
           y = "") %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_text(color = "black", size = 12),
      strip.clip = "off",
      panel.spacing = unit(2, "lines"),
      
      #legend.position = "bottom",
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      #legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  
  fig_att_pa_focus_others = ggplot(df_plot_fc_att, 
                      aes(x = att_pa, 
                          y = factor(name_pa, levels = unique(rev(sort(name_pa)))),
                          xmin = cband_lower_pa, xmax = cband_upper_pa)) %>%
    + geom_point(aes(color = sig_pa)) %>%
    + geom_vline(xintercept = 0) %>%
    + geom_errorbarh(aes(color = sig_pa)) %>% 
    + scale_color_discrete(name = paste0("Significance\n(", (1-alpha)*100, "% level)"),
                           na.translate = F) %>%
    + facet_grid(focus~time,scales="free", space="free",  labeller= as_labeller(names)) %>%
    + labs(title = "Total deforestation avoided",
           x = "ha",
           y = "") %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_text(color = "black", size = 12),
      strip.clip = "off",
      panel.spacing = unit(2, "lines"),
      
      #legend.position = "bottom",
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      #legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  
  fig_att_pa_focus = ggplot(filter(df_plot_fc_att, focus == "focus"), 
                                   aes(x = att_pa, 
                                       y = factor(name_pa, levels = unique(rev(sort(name_pa)))),
                                       xmin = cband_lower_pa, xmax = cband_upper_pa)) %>%
    + geom_point(aes(color = sig_pa)) %>%
    + geom_vline(xintercept = 0) %>%
    + geom_errorbarh(aes(color = sig_pa)) %>% 
    + scale_color_discrete(name = paste0("Significance\n(", (1-alpha)*100, "% level)"),
                           na.translate = F) %>%
    + facet_grid(focus~time,scales="free", space="free",  labeller= as_labeller(names)) %>%
    + labs(title = "Total deforestation avoided",
           subtitle = "Protected areas funded by the FAPBM only",
           x = "ha",
           y = "") %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_text(color = "black", size = 12),
      strip.clip = "off",
      panel.spacing = unit(2, "lines"),
      
      #legend.position = "bottom",
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      #legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  
  fig_att_pa_iucn = ggplot(df_plot_fc_att, 
                           aes(x = att_pa, 
                               y = factor(name_pa, levels = unique(rev(sort(name_pa)))),
                               xmin = cband_lower_pa, xmax = cband_upper_pa)) %>%
    + geom_point(aes(color = sig_pa)) %>%
    + geom_vline(xintercept = 0) %>%
    + geom_errorbarh(aes(color = sig_pa)) %>% 
    + scale_color_discrete(name = paste0("Significance\n(", (1-alpha)*100, "% level)"),
                           na.translate = F) %>%
    + facet_grid(iucn_wolf~time,scales="free", space="free",  labeller= as_labeller(names)) %>%
    + labs(title = "Total deforestation avoided",
           x = "ha",
           y = "") %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_text(color = "black", size = 12),
      strip.clip = "off",
      panel.spacing = unit(2, "lines"),
      
      #legend.position = "bottom",
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      #legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  
  ##treatment effect : avoided deforestation in percentage points
  fig_att_fl = ggplot(df_plot_fl_att, 
                      aes(x = att, 
                          y = factor(name_pa, levels = unique(rev(sort(name_pa)))),
                          xmin = cband_lower, xmax = cband_upper)) %>%
    + geom_point(aes(color = sig)) %>%
    + geom_vline(xintercept = 0) %>%
    + geom_errorbarh(aes(color = sig)) %>% 
    + scale_color_discrete(name = paste0("Significance\n(", (1-alpha)*100, "% level)"),
                           na.translate = F) %>%
    + facet_grid(~time,scales="free", space="free",  labeller= as_labeller(names)) %>%
    + labs(title = "Reduction of deforestation due to the conservation",
           x = "p.p.",
           y = "") %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_text(color = "black", size = 12),
      strip.clip = "off",
      panel.spacing = unit(2, "lines"),
      
      #legend.position = "bottom",
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      #legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  
  fig_att_fl_focus_others = ggplot(df_plot_fl_att, 
                      aes(x = att, 
                          y = factor(name_pa, levels = unique(rev(sort(name_pa)))),
                          xmin = cband_lower, xmax = cband_upper)) %>%
    + geom_point(aes(color = sig)) %>%
    + geom_vline(xintercept = 0) %>%
    + geom_errorbarh(aes(color = sig)) %>% 
    + scale_color_discrete(name = paste0("Significance\n(", (1-alpha)*100, "% level)"),
                           na.translate = F) %>%
    + facet_grid(focus~time,scales="free", space="free",  labeller= as_labeller(names)) %>%
    + labs(title = "Reduction of deforestation due to the conservation",
           x = "p.p.",
           y = "") %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_text(color = "black", size = 12),
      strip.clip = "off",
      panel.spacing = unit(2, "lines"),
      
      #legend.position = "bottom",
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      #legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  
  fig_att_fl_focus = ggplot(filter(df_plot_fl_att, focus == "focus"),
                                   aes(x = att, 
                                       y = factor(name_pa, levels = unique(rev(sort(name_pa)))),
                                       xmin = cband_lower, xmax = cband_upper)) %>%
    + geom_point(aes(color = sig)) %>%
    + geom_vline(xintercept = 0) %>%
    + geom_errorbarh(aes(color = sig)) %>% 
    + scale_color_discrete(name = paste0("Significance\n(", (1-alpha)*100, "% level)"),
                           na.translate = F) %>%
    + facet_grid(focus~time,scales="free", space="free",  labeller= as_labeller(names)) %>%
    + labs(title = "Reduction of deforestation due to the conservation",
           subtitle = "Protected areas funded by the FAPBM only",
           x = "p.p.",
           y = "") %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_text(color = "black", size = 12),
      strip.clip = "off",
      panel.spacing = unit(2, "lines"),
      
      #legend.position = "bottom",
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      #legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  
  fig_att_fl_iucn = ggplot(df_plot_fl_att, 
                           aes(x = att, 
                               y = factor(name_pa, levels = unique(rev(sort(name_pa)))),
                               xmin = cband_lower, xmax = cband_upper)) %>%
    + geom_point(aes(color = sig)) %>%
    + geom_vline(xintercept = 0) %>%
    + geom_errorbarh(aes(color = sig)) %>% 
    + scale_color_discrete(name = paste0("Significance\n(", (1-alpha)*100, "% level)"),
                           na.translate = F) %>%
    + facet_grid(iucn_wolf~time, scales="free", space="free",  labeller= as_labeller(names)) %>%
    + labs(title = "Reduction of deforestation due to the conservation",
           x = "p.p.",
           y = "") %>%
    + theme_minimal() %>%
    + theme(
      axis.text.x = element_text(angle = 0, hjust = 0.5, vjust = 0.5),
      axis.text=element_text(size=11, color = "black"),
      axis.title=element_text(size=14, color = "black", face = "plain"),
      
      plot.caption = element_text(hjust = 0),
      plot.title = element_text(size=16, color = "black", face = "plain", hjust = 0),
      plot.subtitle = element_text(size=12, color = "black", face = "plain", hjust = 0),
      
      strip.text = element_text(color = "black", size = 12),
      strip.clip = "off",
      panel.spacing = unit(2, "lines"),
      
      #legend.position = "bottom",
      legend.text=element_text(size=10),
      #legend.spacing.x = unit(1.0, 'cm'),
      #legend.spacing.y = unit(0.75, 'cm'),
      legend.key.size = unit(2, 'line'),
      
      panel.grid.major.x = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.x = element_line(color = 'grey80', linewidth = 0.2, linetype = 2),
      panel.grid.major.y = element_line(color = 'grey80', linewidth = 0.3, linetype = 1),
      panel.grid.minor.y = element_line(color = 'grey80', linewidth = 0.2, linetype = 2)
    )
  
  
  #Tables 
  ## treatment effect : percentage of deforestation avoided
  tbl_fc_att_per = df_plot_fc_att  %>%
    mutate(focus = case_when(wdpaid %in% list_focus ~ "Yes",
                             !(wdpaid %in% list_focus) ~ "No"),
           sig_per = case_when(sign(cband_lower_per) == sign(cband_upper_per) ~ "Yes",
                               sign(cband_lower_per) != sign(cband_upper_per) ~ "No"),
           iucn_wolf = case_when(iucn_cat %in% c("I", "II", "III", "IV") ~ "Strict",
                                 iucn_cat %in% c("V", "VI") ~ "Non strict",
                                 grepl("not", iucn_cat, ignore.case = TRUE) ~ "Unknown"),
           name_pa = case_when(nchar(name_pa) <= 25 ~ stri_trans_general(name_pa, id = "Latin-ASCII"),
                               nchar(name_pa) > 25 ~ stri_trans_general(paste0(substr(name_pa, 1, 25), "..."),  id = "Latin-ASCII"))
    ) %>%
    dplyr::select(c(name_pa, focus, treatment_year, iucn_wolf, gov_type, time, att_per, sig_per)) %>%
    pivot_wider(values_from = c("att_per", "sig_per"), names_from = c("time", "time")) %>%
    dplyr::select(c(name_pa, focus, treatment_year, iucn_wolf, att_per_5, sig_per_5, att_per_10, sig_per_10)) %>%
    #dplyr::select(c(name_pa, country_en, treatment_year, iucn_wolf, gov_type, att_per_5, sig_per_5, att_per_10, sig_per_10)) %>%
    mutate(across(.cols = starts_with(c("att")),
                  .fns = \(x) case_when(is.na(x) == TRUE ~ "/", TRUE ~ as.character(format(round(x, 2), scientific = FALSE))))) %>%
    mutate(across(.cols = starts_with(c("sig")),
                  .fns = \(x) case_when(is.na(x) == TRUE ~ "/", TRUE ~ x))) %>%
    rename("Effect (5 y., %)" = "att_per_5",
           "Signi. (5 y.)" = "sig_per_5",
           "Effect (10 y., %)" = "att_per_10",
           "Signi. (10 y.)" = "sig_per_10") %>%
    arrange(focus, name_pa)
  # names(tbl_fc_att_per) = c("Name", "FAPBM", "Creation",  "Protection", 
  #                           "Governance", "Effect (5 y., %)", "Significance (5 y.)","Effect (10 y., %)", "Significance (10 y.)")
  names(tbl_fc_att_per) = c("Name", "FAPBM", "Creation", "Protection", 
                            "Effect (5 y., %)", "Signi. (5 y.)","Effect (10 y., %)", "Signi. (10 y.)")
  
  # treatment effect : total deforestation avoided 
  tbl_fc_att_pa = df_plot_fc_att %>%
    mutate(focus = case_when(wdpaid %in% list_focus ~ "Yes",
                             !(wdpaid %in% list_focus) ~ "No"),
           sig_pa = case_when(sign(cband_lower_pa) == sign(cband_upper_pa) ~ "Yes",
                              sign(cband_lower_pa) != sign(cband_upper_pa) ~ "No"),
           iucn_wolf = case_when(iucn_cat %in% c("I", "II", "III", "IV") ~ "Strict",
                                 iucn_cat %in% c("V", "VI") ~ "Non strict",
                                 grepl("not", iucn_cat, ignore.case = TRUE) ~ "Unknown"),
           name_pa = case_when(nchar(name_pa) <= 25 ~ stri_trans_general(name_pa, id = "Latin-ASCII"),
                               nchar(name_pa) > 25 ~ stri_trans_general(paste0(substr(name_pa, 1, 25), "..."),  id = "Latin-ASCII"))
    ) %>%
    dplyr::select(c(name_pa, focus, country_en, treatment_year, iucn_wolf, gov_type, time, att_pa, sig_pa)) %>%
    pivot_wider(values_from = c("att_pa", "sig_pa"), names_from = c("time", "time")) %>%
    # dplyr::select(c(name_pa, country_en, treatment_year, iucn_wolf, gov_type, att_pa_5, sig_pa_5, att_pa_10, sig_pa_10)) %>%
    dplyr::select(c(name_pa, focus, treatment_year, iucn_wolf, att_pa_5, sig_pa_5, att_pa_10, sig_pa_10)) %>%
    mutate(across(.cols = starts_with(c("att")),
                  .fns = \(x) case_when(is.na(x) == TRUE ~ "/", TRUE ~ as.character(format(round(x, 2), scientific = FALSE))))) %>%
    mutate(across(.cols = starts_with(c("sig")),
                  .fns = \(x) case_when(is.na(x) == TRUE ~ "/", TRUE ~ x))) %>%
    rename("Effect (5 y., %)" = "att_pa_5",
           "Signi. (5 y.)" = "sig_pa_5",
           "Effect (10 y., %)" = "att_pa_10",
           "Signi. (10 y.)" = "sig_pa_10") %>%
  arrange(focus, name_pa)
  # names(tbl_fc_att_pa) = c("Name", "FAPBM", "Creation",  "Protection", 
  #                           "Governance", "Effect (5 y., ha)", "Significance (5 y.)","Effect (10 y., ha)", "Significance (10 y.)")
  names(tbl_fc_att_pa) = c("Name", "FAPBM", "Creation", "Protection", 
                            "Effect (5 y., ha)", "Signi. (5 y.)","Effect (10 y., ha)", "Signi. (10 y.)")
  
  # treatment effect : avoided deforestation, in terms of difference in cumultaed deforestation rate 
  tbl_fl_att = df_plot_fl_att %>%
    mutate(focus = case_when(wdpaid %in% list_focus ~ "Yes",
                             !(wdpaid %in% list_focus) ~ "No"),
           sig = case_when(sign(cband_lower) == sign(cband_upper) ~ "Yes",
                              sign(cband_lower) != sign(cband_upper) ~ "No"),
           iucn_wolf = case_when(iucn_cat %in% c("I", "II", "III", "IV") ~ "Strict",
                                 iucn_cat %in% c("V", "VI") ~ "Non strict",
                                 grepl("not", iucn_cat, ignore.case = TRUE) ~ "Unknown"),
           name_pa = case_when(nchar(name_pa) <= 25 ~ stri_trans_general(name_pa, id = "Latin-ASCII"),
                               nchar(name_pa) > 25 ~ stri_trans_general(paste0(substr(name_pa, 1, 25), "..."),  id = "Latin-ASCII"))
    ) %>%
    dplyr::select(c(name_pa, focus, country_en, treatment_year, iucn_wolf, gov_type, time, att, sig)) %>%
    pivot_wider(values_from = c("att", "sig"), names_from = c("time", "time")) %>%
    # dplyr::select(c(name_pa, country_en, treatment_year, iucn_wolf, gov_type, att_5, sig_5, att_10, sig_10)) %>%
    dplyr::select(c(name_pa, focus, treatment_year, iucn_wolf, att_5, sig_5, att_10, sig_10)) %>%
    mutate(across(.cols = starts_with(c("att")),
                  .fns = \(x) case_when(is.na(x) == TRUE ~ "/", TRUE ~ as.character(format(round(x, 2), scientific = FALSE))))) %>%
    mutate(across(.cols = starts_with(c("sig")),
                  .fns = \(x) case_when(is.na(x) == TRUE ~ "/", TRUE ~ x))) %>%
    rename("Effect (5 y., %)" = "att_5",
           "Signi. (5 y.)" = "sig_5",
           "Effect (10 y., %)" = "att_10",
           "Signi. (10 y.)" = "sig_10") %>%
    arrange(focus, name_pa)
  # names(tbl_fl_att) = c("Name", "FAPBM", "Creation",  "Protection", 
  #                           "Governance", "Effect (5 y., pp)", "Significance (5 y.)","Effect (10 y., pp)", "Significance (10 y.)")
  names(tbl_fl_att) = c("Name", "FAPBM", "Creation", "Protection", 
                           "Effect (5 y., pp)", "Signi. (5 y.)","Effect (10 y., pp)", "Signi. (10 y.)")
  
  #Saving plots
  
  ##Saving plots
  tmp = paste(tempdir(), "fig", sep = "/")
  
  ggsave(paste(tmp, "fig_att_per.png", sep = "/"),
         plot = fig_att_per,
         device = "png",
         height = 8, width = 12)

  ggsave(paste(tmp, "fig_att_per_focus.png", sep = "/"),
         plot = fig_att_per_focus,
         device = "png",
         height = 6, width =9)
  
  ggsave(paste(tmp, "fig_att_per_focus_others.png", sep = "/"),
         plot = fig_att_per_focus_others,
         device = "png",
         height = 8, width = 12)
  
  ggsave(paste(tmp, "fig_att_per_iucn.png", sep = "/"),
         plot = fig_att_per_iucn,
         device = "png",
         height = 8, width = 12)
  
  ggsave(paste(tmp, "fig_att_pa.png", sep = "/"),
         plot = fig_att_pa,
         device = "png",
         height = 8, width = 12)
  
  ggsave(paste(tmp, "fig_att_pa_focus.png", sep = "/"),
         plot = fig_att_pa_focus,
         device = "png",
         height = 6, width = 9)
  
  ggsave(paste(tmp, "fig_att_pa_focus_others.png", sep = "/"),
         plot = fig_att_pa_focus_others,
         device = "png",
         height = 8, width = 12)
  
  ggsave(paste(tmp, "fig_att_pa_iucn.png", sep = "/"),
         plot = fig_att_pa_iucn,
         device = "png",
         height = 8, width = 12)
  
  ggsave(paste(tmp, "fig_att_fl.png", sep = "/"),
         plot = fig_att_fl,
         device = "png",
         height = 8, width = 12)
  
  ggsave(paste(tmp, "fig_att_fl_focus.png", sep = "/"),
         plot = fig_att_fl_focus,
         device = "png",
         height = 6, width = 9)
  
  ggsave(paste(tmp, "fig_att_fl_focus_others.png", sep = "/"),
         plot = fig_att_fl_focus_others,
         device = "png",
         height = 8, width = 12)
  
  ggsave(paste(tmp, "fig_att_fl_iucn.png", sep = "/"),
         plot = fig_att_fl_iucn,
         device = "png",
         height = 8, width = 12)
  
  print(xtable(tbl_fc_att_pa, type = "latex", auto = T),
        file = paste(tmp, "tbl_fc_att_pa.tex", sep = "/"))
  
  print(xtable(tbl_fc_att_per, type = "latex", auto = T),
        file = paste(tmp, "tbl_fc_att_per.tex", sep = "/"))
  
  print(xtable(tbl_fl_att, type = "latex", auto = T),
        file = paste(tmp, "tbl_fl_att.tex", sep = "/"))
  
  files <- list.files(tmp, full.names = TRUE)
  ##Add each file in the bucket (same foler for every file in the temp)
  for(f in files) 
  {
    cat("Uploading file", paste0("'", f, "'"), "\n")
    aws.s3::put_object(file = f, 
                       bucket = paste("projet-afd-eva-ap", save_dir, sep = "/"),
                       region = "", show_progress = TRUE)
  }
  do.call(file.remove, list(list.files(tmp, full.names = TRUE)))
}
```

