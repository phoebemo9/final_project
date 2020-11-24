Nursing home data edit
================
Minjie Bao
2020-11-22

``` r
library(tidyverse)
library(readr)
library(plotly)
library(gapminder)
knitr::opts_chunk$set(
  fig.width = 6,
  fig.asp = .6,
  out.width = "90%"
)
theme_set(theme_minimal() +  theme(legend.position = "bottom"))
options(
  ggplots2.continuous.color = "viridis",
  ggplots2.continuous.fill = "viridus"
)
scale_colour_discrete = scale_colour_viridis_d
scale_fill_discrete = scale_fill_viridis_d
```

# 1\. Data import

## 1.1 Read in Nursing Home raw data; Filter by month or filter by day of the last week of month

``` r
# Read in Covid Nursing Home data and filter data from only NY state
NHCovid_df = 
  read_csv("./Data/COVID-19 Nursing Home Data - Archived Data - Week Ending 10_25_20.csv") %>% 
  janitor::clean_names() %>% 
  filter(provider_state == "NY")
```

    ## Warning: 40533 parsing failures.
    ##  row                                       col           expected actual                                                                           file
    ## 1841 Number of Ventilators in Facility         1/0/T/F/TRUE/FALSE     13 './Data/COVID-19 Nursing Home Data - Archived Data - Week Ending 10_25_20.csv'
    ## 1841 Any Current Supply of Ventilator Supplies 1/0/T/F/TRUE/FALSE     Y  './Data/COVID-19 Nursing Home Data - Archived Data - Week Ending 10_25_20.csv'
    ## 1841 One-Week Supply of Ventilator Supplies    1/0/T/F/TRUE/FALSE     Y  './Data/COVID-19 Nursing Home Data - Archived Data - Week Ending 10_25_20.csv'
    ## 1842 Number of Ventilators in Facility         1/0/T/F/TRUE/FALSE     13 './Data/COVID-19 Nursing Home Data - Archived Data - Week Ending 10_25_20.csv'
    ## 1842 Any Current Supply of Ventilator Supplies 1/0/T/F/TRUE/FALSE     Y  './Data/COVID-19 Nursing Home Data - Archived Data - Week Ending 10_25_20.csv'
    ## .... ......................................... .................. ...... ..............................................................................
    ## See problems(...) for more details.

``` r
# Break down specific date into month, day, year
NHCovid_df_by_month = 
  NHCovid_df %>% 
  separate(week_ending, into = c("month", "day", "year"), sep = "/") %>% 
  mutate(
    month = factor(month, levels = c("05", "06", "07", "08", "09", "10")),
    month = str_replace(month, "05", "5"),
    month = str_replace(month, "06", "6"),
    month = str_replace(month, "07", "7"),
    month = str_replace(month, "08", "8"),
    month = str_replace(month, "09", "9"),
  ) %>% 
  mutate(
    day = as.numeric(day)
  )  
# Filter by day to get monthly cases and deaths
NHCovid_df_by_day =
  NHCovid_df_by_month %>% 
  filter(day > 24)
```

# 1.2 Read in NY Nursing Home Facility Info; Calculate mean staff flu vaccination rate of each county

``` r
# Read in facility info data
NH_staff_flu_vaccination_rate = 
  read_csv("./Data/NH_20Profiles/FACILITY_INFO.csv") %>% 
  janitor::clean_names() %>%
# Combined employee flu vaccination rate by taking the mean value from the same county
  group_by(county) %>% 
  summarize(
    staff_flu_vaccination_rate = mean(employee_flu_vaccination_rate)
  )
write_csv(NH_staff_flu_vaccination_rate, path = "./NH_staff_flu_vaccination_rate.csv")
```

# 2\. Data Cleaning and manipulation

# 2.1 Summarize using “residents total covid-19 confirmed and deaths number”

``` r
# data (Summarize monthly death)
NHCovid_df_county_monthly_death =
  NHCovid_df_by_day %>% 
  # to eliminate N/A, filter by submitted data and quality check
  filter(submitted_data == "Y") %>% 
  group_by(month, county) %>% 
  summarise(residents_covid19_deaths_per_month = sum(residents_total_covid_19_deaths)) %>% 
  drop_na()
#plot
monthly_death_plot =
ggplot(NHCovid_df_county_monthly_death, aes(x = county, y = residents_covid19_deaths_per_month, color = month)) +
  geom_point() +
  theme(axis.text.x = element_text(angle = 90, vjust = 0.5, hjust = 1)) +
  theme(legend.position = "right") 
monthly_death_plot
```

<img src="Minjie_files/figure-gfm/unnamed-chunk-4-1.png" width="90%" />

``` r
#data (Summarize monthly cases)
NHCovid_df_county_monthly_confirmed = 
  NHCovid_df_by_day %>% 
  # to eliminate N/A, filter by submitted data and quality check
  filter(submitted_data == "Y") %>% 
  group_by(month, county) %>% 
  summarise(residents_covid19_cases_per_month = sum(residents_total_confirmed_covid_19)) %>% 
  drop_na()
#plot
monthly_confirmed_plot =
ggplot(NHCovid_df_county_monthly_confirmed, aes(x = county, y = residents_covid19_cases_per_month, color = month)) +
  geom_point() +
  theme(axis.text.x = element_text(angle = 90, vjust = 0.5, hjust = 1)) +
  theme(legend.position = "right") 
monthly_confirmed_plot
```

<img src="Minjie_files/figure-gfm/unnamed-chunk-5-1.png" width="90%" />

``` r
#data (Calculate case fatality rate)
NHCovid_df_fatality = 
  left_join(NHCovid_df_county_monthly_death, NHCovid_df_county_monthly_confirmed, by = c("county", "month")) %>% 
  mutate(fatality_rate = residents_covid19_deaths_per_month/residents_covid19_cases_per_month) %>% 
  drop_na()
write_csv(NHCovid_df_fatality, path = "./NHCovid_df_fatality.csv")
#plot
monthly_fatality_plot =
ggplot(NHCovid_df_fatality, aes(x = county, y = fatality_rate, color = month)) +
  geom_point() +
  theme(axis.text.x = element_text(angle = 90, vjust = 0.5, hjust = 1)) +
  theme(legend.position = "right") 
monthly_fatality_plot
```

<img src="Minjie_files/figure-gfm/unnamed-chunk-6-1.png" width="90%" />

## 2.2 Summarize using Total Resident Confirmed/Deaths COVID-19 Cases Per 1,000 Residents

``` r
# data (Summarize monthly death)
NHCovid_df_county_monthly_death_2 =
  NHCovid_df_by_day %>% 
  # to eliminate N/A, filter by submitted data and quality check
  filter(submitted_data == "Y") %>% 
  group_by(month, county) %>% 
  summarise(residents_covid19_deaths_per_month = sum(total_resident_covid_19_deaths_per_1_000_residents)) %>% 
  drop_na()
 
#plot
monthly_death_plot_2 =
ggplot(NHCovid_df_county_monthly_death_2, aes(x = county, y = residents_covid19_deaths_per_month, color = month)) +
  geom_point() +
  theme(axis.text.x = element_text(angle = 90, vjust = 0.5, hjust = 1)) +
  theme(legend.position = "right") 
monthly_death_plot
```

<img src="Minjie_files/figure-gfm/unnamed-chunk-7-1.png" width="90%" />

``` r
#data
NHCovid_df_county_monthly_confirmed_2 = 
  NHCovid_df_by_day %>% 
  # to eliminate N/A, filter by submitted data and quality check
  filter(submitted_data == "Y") %>% 
  group_by(month, county) %>% 
  summarise(residents_covid19_cases_per_month =  sum(total_resident_confirmed_covid_19_cases_per_1_000_residents)) %>% 
  drop_na()
#plot
monthly_confirmed_plot_2 =
ggplot(NHCovid_df_county_monthly_confirmed_2, aes(x = county, y = residents_covid19_cases_per_month, color = month)) +
  geom_point() +
  theme(axis.text.x = element_text(angle = 90, vjust = 0.5, hjust = 1)) +
  theme(legend.position = "right") 
monthly_confirmed_plot
```

<img src="Minjie_files/figure-gfm/unnamed-chunk-8-1.png" width="90%" />

``` r
#data
NHCovid_df_fatality_2 = 
  left_join(NHCovid_df_county_monthly_death_2, NHCovid_df_county_monthly_confirmed_2, by = c("county", "month")) %>% 
  mutate(fatality_rate = residents_covid19_deaths_per_month/residents_covid19_cases_per_month) %>% 
  drop_na()
write_csv(NHCovid_df_fatality_2, path = "./NHCovid_df_fatality_2.csv")
#plot
monthly_fatality_plot_2 =
ggplot(NHCovid_df_fatality_2, aes(x = county, y = fatality_rate, color = month)) +
  geom_point() +
  theme(axis.text.x = element_text(angle = 90, vjust = 0.5, hjust = 1)) +
  theme(legend.position = "right") 
monthly_fatality_plot_2
```

<img src="Minjie_files/figure-gfm/unnamed-chunk-9-1.png" width="90%" />

## 2.3 NH occupancy rate

``` r
NH_occupancy_rate = 
  NHCovid_df_by_month %>%
  select(month, day, provider_name, county, total_number_of_occupied_beds, number_of_all_beds) %>% 
  drop_na() %>% 
  mutate(occupancy_rate =
           total_number_of_occupied_beds/number_of_all_beds) %>% 
  group_by(month, county) %>% 
  summarize(
    occupancy_rate = mean(occupancy_rate))
write_csv(NH_occupancy_rate, path = "./NH_occupancy_rate.csv")
```

## 2.4 NH monthly prevalence of Covid-19

``` r
NH_monthly_prevalence =
  NHCovid_df_by_day %>% 
  select(month, day, county, total_number_of_occupied_beds, residents_total_confirmed_covid_19) %>% 
  drop_na() %>% 
  group_by(month, county) %>% 
  mutate(
    county_occupancy = sum(total_number_of_occupied_beds),
    county_cases = sum(residents_total_confirmed_covid_19),
    county_prevalence = county_cases/county_occupancy
    ) 
write_csv(NH_monthly_prevalence, path = "./NH_monthly_prevalence.csv")
```

## 2.5 Functions set up to reveal the actual increment of cases and deaths by month

``` r
# data (summarize monthly death by function)
# NHCovid_df_county_delta_death = function(data, i, c) {
  
 if (i < 6) {
    delta = data %>% ungroup() %>%
      filter(county == c, month = 5) %>% 
      select(residents_covid19_deaths_per_month) %>% 
      as.numeric()
    
    delta
  }
```

    ## Error in eval(expr, envir, enclos): object 'i' not found

``` r
  if (i >= 6) {
    death_mo_i = data %>% ungroup() %>%
      filter(county == c, month = i) %>% 
      select(residents_covid19_deaths_per_month) %>% 
      as.numeric()
    death_mo_i_1 = data %>% ungroup() %>%
      filter(county == c, month = i - 1) %>% 
      select(residents_covid19_deaths_per_month) %>% 
      as.numeric
    
    delta = death_mo_i - death_mo_i_1
    delta
  }
```

    ## Error in eval(expr, envir, enclos): object 'i' not found

``` r
# output = purrr::map(NHCovid_df_mortality, NHCovid_df_county_delta_death)
```

## 2.6.1 Combine weekly shortage of stuff

``` r
#data
shortage_of_stuff =
NHCovid_df_by_month %>% 
  filter(submitted_data == "Y") %>% 
  select(month, day, county, starts_with("shortage")) %>% 
    mutate(
     total_shortage_stuff = case_when(
      shortage_of_nursing_staff != "N"  ~ 1,
      shortage_of_clinical_staff != "N" ~ 1,
      shortage_of_aides != "N"          ~ 1,
      shortage_of_other_staff != "N"    ~ 1)) %>% 
replace_na(list(total_shortage_stuff = 0)) %>% 
  select(month, day, county, total_shortage_stuff)
```

## 2.6.2 convert weekly shortage of stuff to monthly

``` r
monthly_shortage_of_stuff = 
shortage_of_stuff %>% 
  mutate(month = as.numeric(month)) %>% 
  group_by(month, county) %>% 
  summarise(
    total_shortage_stuff_month = sum(total_shortage_stuff)
  ) %>% 
  mutate(
   total_shortage_stuff_monthly = case_when(
     total_shortage_stuff_month > 0  ~ 1
   ) 
  ) %>% 
  replace_na(list(total_shortage_stuff_monthly = 0)) %>% 
  select(month, county, total_shortage_stuff_monthly)
```

    ## `summarise()` regrouping output by 'month' (override with `.groups` argument)

``` r
#plot
monthly_shortage_of_stuff_plot =
  monthly_shortage_of_stuff %>% 
  filter(total_shortage_stuff_monthly == 1) %>% 
ggplot(aes(x = county, y = month, color = month)) +
  geom_point() +
  theme(axis.text.x = element_text(angle = 90, vjust = 0.5, hjust = 1)) +
  theme(legend.position = "right") 
monthly_shortage_of_stuff_plot
```

<img src="Minjie_files/figure-gfm/unnamed-chunk-14-1.png" width="90%" />

## 2.7.1 Combine one-week supply of ppe

``` r
#data
supply_of_ppe =
NHCovid_df_by_month %>% 
  filter(submitted_data == "Y") %>% 
  select(month, county, day, starts_with("one_week_supply"), -one_week_supply_of_ventilator_supplies) %>% 
    mutate(
     total_supply_ppe = case_when(
      one_week_supply_of_n95_masks      != "N" ~ 1,
      one_week_supply_of_surgical_masks != "N" ~ 1,
      one_week_supply_of_eye_protection != "N" ~ 1,
      one_week_supply_of_gowns          != "N" ~ 1,
      one_week_supply_of_gloves         != "N" ~ 1,
      one_week_supply_of_hand_sanitizer != "N" ~ 1)) %>% 
replace_na(list(total_supply_ppe = 0)) %>% 
  select(month, day, county, total_supply_ppe)

#supply_of_ppe %>% 
#  filter(total_supply_ppe == 0) only 105/13983, very small portion didn't have weekly supply
```

## 2.7.2 monthly supply of ppe

``` r
monthly_supply_of_ppe = 
supply_of_ppe %>% 
  mutate(month = as.numeric(month)) %>% 
  group_by(month, county) %>% 
  summarise(
    total_supply_ppe_month = sum(total_supply_ppe)
  ) %>% 
  mutate(
   total_supply_ppe_monthly = case_when(
     total_supply_ppe_month > 0  ~ 1
   ) 
  ) %>% 
  replace_na(list(total_supply_ppe_monthly = 0)) %>% 
  select(month, county, total_supply_ppe_monthly)
```

    ## `summarise()` regrouping output by 'month' (override with `.groups` argument)

``` r
##every month there's ppe supply
```

## 2.8.1 Ventilator dependent unit

``` r
ventilator_dependent_unit =
NHCovid_df_by_month %>% 
  filter(submitted_data == "Y") %>% 
  select(month, day, county, ventilator_dependent_unit) %>% 
    mutate(
     ventilator_dependent_unit = case_when(
      ventilator_dependent_unit != "N"  ~ 1)) %>% 
replace_na(list(ventilator_dependent_unit = 0)) %>% 
  select(month, day, county, ventilator_dependent_unit)
```

## 2.8.2 monthly Ventilator dependent unit

``` r
monthly_ventilator_dependent_unit = 
ventilator_dependent_unit %>% 
  mutate(month = as.numeric(month)) %>% 
  group_by(month, county) %>% 
  summarise(
    total_ventilator_dependent_unit = sum(ventilator_dependent_unit)
  ) %>% 
  mutate(
   ventilator_dependent_unit_monthly = case_when(
     total_ventilator_dependent_unit > 0  ~ 1
   ) 
  ) %>% 
  replace_na(list(ventilator_dependent_unit_monthly = 0)) %>% 
  select(month, county, ventilator_dependent_unit_monthly)
```

    ## `summarise()` regrouping output by 'month' (override with `.groups` argument)

``` r
#plot
monthly_ventilator_dependent_unit_plot =
  monthly_ventilator_dependent_unit %>% 
  filter(ventilator_dependent_unit_monthly == 1) %>% 
ggplot(aes(x = county, y = month, color = month)) +
  geom_point() +
  theme(axis.text.x = element_text(angle = 90, vjust = 0.5, hjust = 1)) +
  theme(legend.position = "right") 
monthly_ventilator_dependent_unit_plot
```

<img src="Minjie_files/figure-gfm/unnamed-chunk-18-1.png" width="90%" />

## 2.9 Merge 3 categorical variables by month for regression

``` r
two_df =
left_join(monthly_supply_of_ppe, monthly_shortage_of_stuff, by = c("county", "month")) 

categorical_df =
  left_join(two_df, monthly_ventilator_dependent_unit, by = c("county", "month")) 
write_csv(categorical_df, path = "./data/categorical_df.csv")
```