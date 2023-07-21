# Building the datasets

## Importing relevant packages




```r
library(tidyverse)
library(stargazer)
library(dplyr)
library(data.table)
library(readxl)
library(splitstackshape) 
library(janitor)
library(stringi)
library(sf)
library(mapview)
```

## Datasets for descriptive statistics

The raw dataset has been built by Léa Poulin. It comes from the merging of SIOP extract, and PAs dataset from ARB covering 2000-2017 period. Majority of PAs had no corresponding WDPA ID, so it was found and reported manually by Léa Poulin and Ingrid Dallmann on WDPA website.

Different datasets are built. First a dataset (data_PA_tidy) enriched of information on co-investors and ISO code for the country hosting the PA.

From this dataset are extracted : (1) a dataset with only SIOP variables, to allow future working on PAs; (2) a dataset to perform most descriptive statistics, except confidential ones related to funding; (3) a confidential dataset with funding data.

Then a dataset of PAs polygons is imported from the WDPA, and confidential data removed.

Some statistics need size of PAs at country/region/world level (evolution of area covered by PAs at world level for instance). Datasets with total areas at country/region/world level are created. As many areas overlap (according to WDPA documentation), a sum of reported area would overestimate aggregated area. The computation method of WDPA is followed (<https://www.protectedplanet.net/en/resources/calculating-protected-area-coverage>).

### Cleaning the raw dataset


```r
#import a dataset with country names and corresponding ISO3 code
data_iso = fread("data_raw/liste-197-etats-2020.csv") %>%
  rename("iso3" = "CODE",
         "nom_alpha" = "NOM_ALPHA") %>%
  dplyr::select(c("nom_alpha", "iso3")) %>%
  #get rid of accentuations and change name by hand to correspond with BDD_joint
  mutate(nom_alpha2 = iconv(nom_alpha, to = "ASCII//TRANSLIT"),
         nom_alpha2 = case_when(
           nom_alpha2 == "Cook (Iles)" ~ "Cook",
           nom_alpha2 == "Cote d'Ivoire" ~ "Cote D Ivoire",
           nom_alpha2 == "Sao Tome-et-Principe" ~ "Sao Tome",
           nom_alpha2 == "Palaos" ~ "Palau",
           nom_alpha2 == "Guinee-Bissao" ~ "Guinee-Bissau",
           !(nom_alpha %in% c("Cook (Iles)", "Cote d'Ivoire", "Sao Tome-et-Principe",
                              "Palaos", "Guinée-Bissao")) ~ nom_alpha2
         ))

#Import the initial BDD_joint, change the encoding to keep accentuation 
data_PA_raw = read_excel("data_raw/BDD_joint.xlsx") %>%
  as.data.frame() %>%
  mutate(across(.cols = names(.), 
                .fns = ~stri_enc_toutf8(.x)))

#Modify errors in the dataset
##WDPAID 797 in Senegal
data_PA_raw[data_PA_raw$wdpaid == "797" & data_PA_raw$pays == "Senegal",]$nom_ap = "APAC de Kawawana"
data_PA_raw[data_PA_raw$wdpaid == "797" & data_PA_raw$pays == "Senegal",]$wdpaid = NA
##4223, 4224, 4226, 4228, 4229 : all in PS-N.Caledonie
data_PA_raw[data_PA_raw$wdpaid %in% c("4223", "4224", "4226", "4228", "4229") & data_PA_raw$pays == "Fidji",]$pays = "P-S N.Caléd"
##305082 : Vanuatu instead of Fidji
data_PA_raw[data_PA_raw$wdpaid %in% c("305082") & data_PA_raw$pays == "Fidji",]$pays = "Vanuatu"


#Create a clean dataset with relevant variables
data_PA_tidy = data_PA_raw %>%
  #Get rid of the iso code in the initial dataset, to avoid confusion when merging with data_iso
  dplyr::select(-iso3) %>%
  #Create dummy variables for main investors
  #AFD is always funder, so no need of a dummy
  mutate(
    # afd_bin = case_when((cofinancier_1 == "AFD" | cofinancier_2 == "AFD" | cofinancier_3 == "AFD" |
    #                       cofinancier_4 == "AFD" | cofinancier_5 == "AFD" | cofinancier_6 == "AFD") ~ TRUE,
    #                       (is.na(cofinancier_1) & is.na(cofinancier_2) & is.na(cofinancier_3) & is.na(cofinancier_4) 
    #                       & is.na(cofinancier_5) & is.na(cofinancier_6)) ~ TRUE,
    #                       TRUE ~ FALSE),
         kfw_bin = ifelse(cofinancier_1 == "KFW" | cofinancier_2 == "KFW" | cofinancier_3 == "KFW" |
                          cofinancier_4 == "KFW" | cofinancier_5 == "KFW" | cofinancier_6 == "KFW",
                          yes = TRUE, 
                          no = FALSE),
         kfw_bin = ifelse(is.na(kfw_bin), yes = FALSE, no = kfw_bin),
         ffem_bin = ifelse(cofinancier_1 == "FFEM" | cofinancier_2 == "FFEM" | cofinancier_3 == "FFEM" |
                          cofinancier_4 == "FFEM" | cofinancier_5 == "FFEM" | cofinancier_6 == "FFEM",
                          yes = TRUE, 
                          no = FALSE),
         ffem_bin = ifelse(is.na(ffem_bin), yes = FALSE, no = ffem_bin),
         .after = "cofinancier_6") %>%
  #Create a dummy for other co-investors
  # mutate(cofin_unkwn = ifelse(is.na(cofinancier_1) & is.na(cofinancier_1) & is.na(cofinancier_1) & is.na(cofinancier_1) & is.na(cofinancier_1) & is.na(cofinancier_1),
  #                         yes = TRUE, 
  #                         no = FALSE),
  #        cofin_unkwn = ifelse(is.na(cofin_unkwn), yes = FALSE, no = cofin_unkwn),
  #        .after = "ffem_bin") %>%
  #Create a dummy for investors not AFD, KFW or FFEM
  # mutate(cofin_other = afd_bin == FALSE & 
  #       kfw_bin == FALSE & ffem_bin == FALSE & cofin_unkwn == FALSE,
  #       .after = "cofin_unkwn") %>%
  #Add the ISO3 code corresponding to the country hosting the PA. Facilitate future mergings
  left_join(data_iso, by = c("pays" = "nom_alpha2")) %>%
  #Some entries in "pays" are French department, DROM-COM, "Ocean Indien" or "Multi-Pays".
  #French related : assigned to France. Ocen Indien let NA value, Muti-pays set to ZZ as in the SIOP 
  mutate(iso3 = case_when(
    pays %in% c("Mayotte", "P-N N.Caléd", "P-S N.Caléd", 
                "Polynesie Francaise", "Nlle Caledonie") ~ "FRA",
    pays %in% c("Multi-Pays", "Ocean Indien") ~ "ZZ",
    !(pays %in% c("Mayotte", "P-N N.Caléd", "P-S N.Caléd", 
                "Polynesie Francaise", "Nlle Caledonie", "Multi-Pays")) ~ iso3
  )) %>%
  #Add the description of IUCN from its category
    mutate(iucn_des = case_when(
  !is.na(wdpaid) & iucn_cat == "Ia" ~ "Réserve naturelle intégrale",
  !is.na(wdpaid) & iucn_cat == "Ib" ~ "Zone de nature sauvage",
  !is.na(wdpaid) & iucn_cat == "II" ~ "Parc national",
  !is.na(wdpaid) & iucn_cat == "III" ~ "Monument naturel",
  !is.na(wdpaid) & iucn_cat == "IV" ~ "Gest. des habitats/espèces",
  !is.na(wdpaid) & iucn_cat == "V" ~ "Paysage protégé",
  !is.na(wdpaid) & iucn_cat == "VI" ~ "Gest. de ress. protégées",
  !is.na(wdpaid) & iucn_cat == "Not Applicable" ~ "Non catégorisée",
  !is.na(wdpaid) & iucn_cat == "Not Reported" ~ "Non catégorisée",
  !is.na(wdpaid) & iucn_cat == "Not Assigned" ~ "Non catégorisée",
  TRUE ~ "Non référencée"), .after = iucn_cat) %>%
    #Modify class of some variables
  mutate(across(.cols = c("wdpaid", "superficie",
                          "montant_prevu_concours_euro_octroi",
                          "mt_global_projet_prevu_devise" ,
                          "engagements_nets_euro_octroi",
                          "montant_total_projet"),
                .fns = ~as.numeric(.x))) %>%
    mutate(across(.cols = -c("wdpaid", "superficie",
                                  "montant_prevu_concours_euro_octroi",
                          "mt_global_projet_prevu_devise" ,
                          "engagements_nets_euro_octroi",
                          "montant_total_projet"), 
                .fns = ~stri_enc_toutf8(.x))) 
#fwrite(data_PA_tidy, "data_tidy/BDD_joint_tidy.csv")
```

### Dataset with only SIOP variables


```r
#Select info corresponding to SIOP extract and AP 
list_var_siop_AP = c( "id_projet", "nom_du_projet", "nom_de_projet_pour_les_instances",
                      "id_concours", "libelle_du_concours", "description_du_projet",
                      "wdpaid", "wdpaid_2", "wdpaid_3",
                      "montant_prevu_concours_euro_octroi", "mt_global_projet_prevu_devise", 
                      "engagements_nets_euro_octroi", "montant_total_projet", "produit", 
                      "libelle_produit", "division_technique", "libelle_division_technique", "agence", 
                      "libelle_agence", "annee_octroi", "date_octroi",
                      "date_de_1er_versement_projet", "dernier_versement_en_date_projet", 
                      "date_signature_convention_cf", "duree_du_concours_annee",
                      "duree_du_concours_annee_et_mois", "id_pays_de_realisation", "pays", "iso3",
                      "direction_regionale", "autres_pays_de_realisation", 
                      "etat_du_projet", "libelle_etat_du_projet", "beneficiaire_primaire",
                      "responsable_equipe_projet", "directeur_trice_dagence",
                      "date_de_remise_du_rapport_final", "cofinancier_1", "cofinancier_2", 
                      "cofinancier_3","cofinancier_4", "cofinancier_5", "cofinancier_6",
                      "kfw_bin", "ffem_bin",
                      "maitrise_ouvrage", "superf_interne", "nb_ap_nombre_potentiel", "detail", "projet", "commentaires")

#Definining dataset for future analysis : siop and AP but not WDPA 
data_siop_AP = data_PA_tidy %>%
  dplyr::select(all_of(list_var_siop_AP), iso3)
#write_csv(data_siop_AP, "data_tidy/BDD_AP_SIOP_joint.xlsx")
```

### Dataset for non-confidential statistics

To perform descriptive statistics we want to keep one row per PA, characterized by WDPA ID or a name (if no ID reported). Some PAs can have several lines if they receive funds at different time or by different investors.


```r
#Listing relevant variables for descriptive statistics that are NOT confidential (i.e not concern funding)
list_var_PA_nofund = 
  c("id_projet", "nom_du_projet", "nom_ap", "id_concours", 
    "wdpaid", 
    "pays", "iso3", "direction_regionale",
    "annee_octroi", "nb_ap_nombre_potentiel", "detail",
    "iucn_cat", "iucn_des", "marine", "superficie",
    "status", "status_yr", "gov_type","own_type", "mang_auth")

#Defining dataset for descriptive statistics
data_stat_nofund = data_PA_tidy %>%
  select(all_of(list_var_PA_nofund), iso3)

#fwrite(data_stat_nofund, "data_tidy/BDD_DesStat_nofund.csv")

#Then to keep only one row per PA, we need to consider separately PAs having WDPA ID and PAs which do not.

data_stat_nofund_wdpa = data_stat_nofund %>%
  subset(is.na(wdpaid) == FALSE) %>%
  group_by(wdpaid) %>% 
  arrange(annee_octroi) %>%
  slice(1) %>%
  ungroup()

data_stat_nofund_na = data_stat_nofund %>%
  subset(is.na(wdpaid) == TRUE) %>%
  mutate(id1 = paste(id_projet, iso3, nom_ap, sep = "_"),
       .before = id_projet) %>%
  group_by(id1) %>%
  arrange(annee_octroi) %>%
  slice(1) %>%
  ungroup() %>%
  #remove id1 variable, to bind data_stat_na and data_stat_wdpa by row
  select(-id1)

data_stat_nofund_nodupl = rbind(data_stat_nofund_wdpa, data_stat_nofund_na) 

#fwrite(data_stat_nofund_nodupl, "data_tidy/BDD_DesStat_nofund_nodupl.csv")
```

### Dataset for confidential descriptive statistics


```r
#Listing relevant variables for descriptive statistics
list_var_PA_fund = c("id_projet", "nom_du_projet", "nom_ap", "id_concours", 
                "wdpaid",
                "pays", "iso3", "direction_regionale",
                "montant_prevu_concours_euro_octroi",
                "mt_global_projet_prevu_devise",
                "engagements_nets_euro_octroi",
                "montant_total_projet", "libelle_produit", "annee_octroi",
                "cofinancier_1", "cofinancier_2", "cofinancier_3", 
                "cofinancier_4", "cofinancier_5", "cofinancier_6",
                 "kfw_bin", "ffem_bin")

#Defining dataset for descriptive statistics
#Duplicates are not revmoved here, as we want the information on the different concours for instance. Peculiar datasets 
data_stat_fund = data_PA_tidy %>%
  select(all_of(list_var_PA_fund), iso3)

#fwrite(data_stat_fund, "data_tidy/BDD_DesStat_fund.csv")
```

### A polygon dataset without confidential information


```r
#The confidential information removed concern funding or nominative variables
pa_shp = read_sf("data_raw/WDPA_SHP/BDD_SHP_nodupl.shp") %>%
  clean_names() %>%
  select(-c(mntnt, mtglb, e, mnt, prodt, lbllp, dt_ct, d_1, d, d_c, durdcncrs_a, drdcncrs_an, atrsp, ett, lbllt, rspns, dr, mtrs, detal, projt, cmmnt, cmmnt_1))

# st_write(pa_shp,
#          dsn = "data_tidy/BDD_shp_pub.gpkg",
#          delete_dsn = TRUE)

rm(pa_shp)
```

## Computing total areas covered by PAs in the sample

Knowing the total area covered by PAs at different level of aggregation is interesting per se. It is also necessary to compute several statistics (e.g average funding by unit of area). According to the WDPA documentation, it is likely that some reported polygons overlap. Simply summing the areas would thus lead to a biased estimate of the total area at a given level of aggregation. We follow the procedure of the WDPA (<https://www.protectedplanet.net/en/resources/calculating-protected-area-coverage>). Our case is simpler as all of the PAs we consider are given a polygon.

1.  The layer is converted to Mollweide (an equal area projection) and the area of each polygon is calculated, in km2.

2.  Intersection of polygons and the corresponding area are computed.

3.  Then the intersection can be aggregated at country, region or world level. Then it is subtracted to the sum of areas at country, region or world level. Note that intersections between PAs whose polygon is unknown won't be taken into account.

Note that the following codes are about computing total area at country/region/world level, taking potential intersections into account. It is not about generating a new shape files for the impact analysis. Indeed the overlap should be taken into account in the impact evaluation analysis codes.

### Computations of polygons' area


```r
#Importing shapefiles
sf_use_s2(FALSE)
pa_shp = 
  read_sf("data_tidy/BDD_shp_raw.gpkg") %>%
  # aws.s3::s3read_using(
  # FUN = sf::read_sf,
  # # Mettre les options de FUN ici
  # object = "data_tidy/BDD_SHP_nodupl_pub.gpkg",
  # bucket = "projet-afd-eva-ap",
  # opts = list("region" = "")) %>%
  #Ensure all geometries are valid
  st_make_valid() %>%
  #From multipolygon to polygon
  sf::st_cast(to="POLYGON") %>%
  clean_names() %>%
  #Select relevant variables
  #Note ann_c = annee_octroi in the initial database data_PA_raw
  dplyr::select(c(wdpaid, sprfc, rep_a, gis_a, geom, iso3, drct, ann_c)) %>%
  #Variable sprfc corresponds to AFD internal reported size
  rename("sprfc_int" = "sprfc") %>%
  mutate(wdpaid = as.numeric(wdpaid),
         sprfc_int = as.numeric(sprfc_int)) 

#Spatial definition of wdpaid 555547988 overlaps CMR and CAF. Wdpaid 1245 corresponds to the CMR part. The overlap is removed and iso3 redefined so that 555547988 is CAF only. 
geom_555547988_1245 = st_difference(pa_shp[pa_shp$wdpaid == 555547988,]$geom, pa_shp[pa_shp$wdpaid == 1245,]$geom)
pa_shp[pa_shp$wdpaid == 555547988,]$geom = geom_555547988_1245
pa_shp[pa_shp$wdpaid == 555547988,]$iso3 = "CAF"

#Define a tidy version of the former dataset, with modifications on wdpaid 555547988
pa_shp_tidy = pa_shp %>%
  #Project to Mollweide to compute relevant areas in km2
  st_transform(crs = "+proj=moll +datum=WGS84") %>%
  #Compute areas in km2 from the geometry, in km2. It must be equal to gis_a by definition 
  #Then to take into account potential refinements of the geometries (as for wdpaid 55547988), a variable for relevant area is defined. It takes rep_a value except for modified geometries where area_sf_moll is taken
  mutate(area_sf_moll = as.numeric(st_area(geom)/1e6),
         sprfc_km2 = ifelse(wdpaid == 555547988, yes = area_sf_moll, no = rep_a))
```

### Computing the intersection at country, region, world level


```r
#Compute intersecting areas of polygons
pa_int = st_intersection(pa_shp_tidy, pa_shp_tidy) %>%
  #Remove intersection of polygons with themselves
  subset(wdpaid != wdpaid.1) %>%
  #If one of the two intersectin polygon have unknwon area, then it is not necessary to subtract the interesction area. Indeed there is no double-counting of the intersection in this case, when both polygon areas are summed.
  subset(is.na(sprfc_km2) == FALSE & is.na(sprfc_km2.1) == FALSE) %>%
  #Compute the intersecting areas (pa_shp already in Mollweide projection) in km2
  mutate(area_int = as.numeric(st_area(geom)/1e6)) %>%
  #Now duplicates need to be removed : intersection of X with Y AND intersection of Y with X are reported. We need only one.
  #An id_int to identify the intersection of a given pair
  mutate(id_int = paste0(wdpaid, "_", wdpaid.1), .before = wdpaid) %>%
  mutate(id_int_temp = paste0(wdpaid, "_", wdpaid.1), .before = wdpaid) %>%
  #create a mirror idX_idY --> idY_idX so that we identify the both member of a pair with the same id
  separate(id_int_temp, into = c("id_temp1", "id_temp2"), sep = "_") %>%
  mutate(id_int_rev = case_when(
    id_temp1 < id_temp2 ~ paste(id_temp1, id_temp2, sep = "_"),
    id_temp1 > id_temp2 ~ paste(id_temp2, id_temp1, sep = "_"),
    TRUE ~ paste(id_temp1, id_temp2, sep = "_")),
    .after = id_int) %>%
  #finally, get rid of the duplicates (have the same id_int_rev)
  group_by(id_int_rev) %>%
  slice(1) %>%
  ungroup() %>%
  #select relevant variables only
  select(wdpaid, iso3, drct, ann_c, wdpaid.1, iso3.1, drct.1, ann_c.1, geom, area_int)

#Computing the total area of intersections
#At country level ...
pa_int_ctry = pa_int %>%
  #Only overlapping PAs in the same country are considered
  subset(iso3 ==  iso3.1) %>%
  group_by(iso3) %>%
  summarize(tot_area_int = sum(area_int)) %>%
  st_drop_geometry()

#At region level
pa_int_dr = pa_int %>%
  #Only overlapping PAs in the same DR are considered
  subset(drct == drct.1) %>%
  group_by(drct) %>%
  summarize(tot_area_int = sum(area_int)) %>%
  st_drop_geometry()

##At world level : all overlap are considered
pa_int_wld = sum(pa_int$area_int) 

#Compute the total intersection for each year
pa_int_yr = pa_int %>%
  #Define intersection year : the date ann_c of the later PA in the pair
  rowwise() %>%
  mutate(annee_int = max(ann_c, ann_c.1)) %>% 
  group_by(annee_int) %>%
  summarize(tot_int_km2 = sum(area_int)) %>%
  st_drop_geometry()

# fwrite(pa_int_yr,
#        "data_tidy/area/pa_int_yr.csv")
```

### Computing total areas without intersection


```r
#At country level ...
pa_area_ctry = data_stat_nofund_nodupl %>%
  #Compute total area at country level
  group_by(iso3) %>%
  summarize(sprfc_tot_km2 = sum(superficie)) %>%
  ungroup() %>%
  #Add information on intersection area in each country. Modify the variable so that NA value -> 0
  left_join(pa_int_ctry, by = "iso3") %>%
  mutate(tot_area_int = case_when(is.na(tot_area_int) == TRUE ~ 0, TRUE ~ tot_area_int)) %>%
  #Compute the total area at country level without intersection
  mutate(sprfc_tot_noint_km2 = sprfc_tot_km2 - tot_area_int) 

#fwrite(pa_area_ctry, "data_tidy/area/pa_area_ctry.csv")

#At DR level ...
pa_area_dr = data_stat_nofund_nodupl %>%
  #Compute total area at dr level
  group_by(direction_regionale) %>%
  summarize(sprfc_tot_km2 = sum(superficie)) %>% 
  ungroup() %>%
  #Add information on intersection area in each DR. Modify the variable so that NA value -> 0
  left_join(pa_int_dr, by = c("direction_regionale" = "drct")) %>%
  mutate(tot_area_int = case_when(is.na(tot_area_int) == TRUE ~0, TRUE ~tot_area_int)) %>%
  #Compute the total area at country level without intersection
  mutate(sprfc_tot_noint_km2 = sprfc_tot_km2 - tot_area_int) %>%
  st_drop_geometry()

#fwrite(pa_area_dr, "data_tidy/area/pa_area_dr.csv")

#At world level
pa_area_wld = sum(data_stat_nofund_nodupl$superficie) - pa_int_wld %>%
  as.data.frame() %>%
  rename("sprfc_tot_noint_km2" = ".")
#fwrite(pa_area_wld, "data_tidy/area/pa_area_wld.csv")
```

## Old code


```r
#Listing variables public or confidential

#Public variables in the PAs dataset
list_var_PA_open = c("id_projet", "nom_du_projet", "nom_ap", "id_concours", 
                "wdpaid", "wdpaid_2", "wdpaid_3", "wdpa_pid",
                "pays", "iso3", "direction_regionale", "annee_octroi",
                "cofinancier_1", "cofinancier_2", "cofinancier_3", 
                "cofinancier_4", "cofinancier_5", "cofinancier_6",
                "kfw_bin", "ffem_bin", "nb_ap_nombre_potentiel", "detail",
                "iucn_cat", "iucn_des", "marine", "superficie",
                "status", "status_yr", "gov_type","own_type", "mang_auth")
#Confidential variables in the PAs dataset
list_var_PA_conf = list_var_joint_conf = 
  c("montant_prevu_concours_euro_octroi", "mt_global_projet_prevu_devise",
    "engagements_nets_euro_octroi" , "montant_total_projet",
    "produit", "libelle_produit",
    "date_de_1er_versement_projet", "dernier_versement_en_date_projet",
    "date_signature_convention_cf", "duree_du_concours_annee",
    "duree_du_concours_annee_et_mois", "beneficiaire_primaire",
    "responsable_equipe_projet", "directeur_trice_dagence", "maitrise_ouvrage"
    )
#Confidential variables among shapefiles
list_var_shp_conf = c("mntnt", "mtglb", "e", "mnt", "prodt", "lbllp",
                      "rspns", "dr", "detal", "projt", "cmmnt")
```


```r
base_nodupl_lea = 
  #fread("data_tidy/base_nodupl_lea.xlsx")
  aws.s3::s3read_using(
  FUN = data.table::fread,
  encoding = "UTF-8",
  # Mettre les options de FUN ici
  object = "data_tidy/BDD_DesStat_nodupl_lea_pub.csv",
  bucket = "projet-afd-eva-ap",
  opts = list("region" = "")) %>%
  select(any_of(names(data_stat_nodupl)))
  
#import a dataset with country anmes and corresponding ISO3 code
data_iso = data_stat %>%
  select(c(pays, iso3)) %>%
  group_by(iso3) %>%
  slice(1)

base_nodupl_old = 
  #fread("data_tidy/base_nodupl_old.csv") 
  aws.s3::s3read_using(
  FUN = data.table::fread,
  encoding = "UTF-8",
  # Mettre les options de FUN ici
  object = "data_tidy/BDD_DesStat_nodupl_old_pub.csv",
  bucket = "projet-afd-eva-ap",
  opts = list("region" = ""))%>%
  select(any_of(names(data_stat_nodupl))) %>%
  select(-iso3) %>%
  left_join(data_iso, by = "pays") %>%
  mutate(iso3 = case_when(
    pays %in% c("Mayotte", "P-N N.Caléd", "P-S N.Caléd", 
                "Polynesie Francaise", "Nlle Caledonie") ~ "FRA",
    pays == "Multi-Pays" ~ "ZZ",
    !(pays %in% c("Mayotte", "P-N N.Caléd", "P-S N.Caléd", 
                "Polynesie Francaise", "Nlle Caledonie", "Multi-Pays")) ~ iso3
  ))

# projet_nodupl_old = 
#   #readxl::read_xlsx("data_tidy/projet_nodupl_old.xlsx")
#   aws.s3::s3read_using(
#   FUN = readxl::read_xlsx,
#   # Mettre les options de FUN ici
#   object = "data_tidy/projet_nodupl_old.xlsx",
#   bucket = "projet-afd-eva-ap",
#   opts = list("region" = ""))

bdd_joint = 
  #fread("data_raw/BDD_joint.xlsx")
  aws.s3::s3read_using(
  FUN = data.table::fread,
  # Mettre les options de FUN ici
  encoding = "UTF-8",
  object = "data_tidy/BDD_joint_tidy_pub.csv",
  bucket = "projet-afd-eva-ap",
  opts = list("region" = "")) %>%
  select(any_of(names(data_stat_nodupl)))
```


```r
#Test : differences between Léa's figures and my results

#PA identified with unique WDPAID. If more than 1 WDPAID, take the earlier annee_octroi
base_id = data_stat %>% 
  filter(!is.na(wdpaid)) %>% 
  group_by(wdpaid) %>% 
  arrange(annee_octroi) %>%
  slice(1) %>%
  ungroup()

test = base_id[duplicated(base_id$wdpaid),]

base_old_id = base_nodupl_old %>% 
  filter(grepl("NA", wdpaid) == FALSE)  %>%
  group_by(wdpaid) %>% 
  slice(1) %>%
  mutate(wdpaid = as.numeric(wdpaid)) %>%
  ungroup()

#Identify PA in common among them with WDPAID reported
test_id = base_old_id %>%
  select(c(id_projet, nom_du_projet, nom_ap, id_concours, wdpaid)) %>%
  left_join(select(base_id, 
                   c(id_projet, nom_du_projet, nom_ap, id_concours, wdpaid)),
            by = "wdpaid")

#PA whose WDPAID is unknown, identified by id1 uniquely
#The duplicates comes from different concours ID, so do not correspond to different PA
base_na = data_stat %>% filter(is.na(wdpaid)) %>%
  mutate(id1 = paste(id_projet, iso3, nom_ap, sep = "_"),
         .before = id_projet) %>%
  group_by(id1) %>%
  arrange(annee_octroi) %>%
  slice(1) %>%
  ungroup()

base_old_na = base_nodupl_old %>% 
  filter(grepl("NA", wdpaid) == TRUE) %>%
  mutate(id1 = paste(id_projet, iso3, nom_ap, sep = "_"),
         .before = id_projet) %>%
  group_by(id1) %>%
  arrange(annee_octroi) %>%
  slice(1) %>%
  ungroup()

test_na = base_na %>%
  select(c(id1, id_projet, nom_du_projet, nom_ap, id_concours)) %>%
  left_join(select(base_old_na, 
                   c(id1, id_projet, nom_du_projet, nom_ap, id_concours)),
            by = "id1")

nrow(test_na %>% subset(is.na(id_projet.y) == TRUE))

#base_nodupl

base_nodupl = rbind(select(base_na, -id1), base_id)
```