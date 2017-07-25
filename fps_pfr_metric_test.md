# FPS-PFR Validation
Joe Mienko  
7/25/2017  





## FPS/PFR Performance Metric 1 - MEASURABLE

### Average number of hours from referral received by provider to first contact with family.

We begin by selecting the date of acceptance for a given referral. This variable will be reused for later measurements.  


```r
var_referral_accept <- tbl(con, "ChildWelfareCaseNotes") %>%
  inner_join(tbl(con, "ChildWelfareCaseNoteTypes")
             ,by = c("typeId" = "id")) %>%
  as_data_frame() %>%
  filter(typeId == 4
         ,is.na(deletedAt.x)) %>%
  mutate(acceptance_dtg = lubridate::ymd_hms(paste(date, time), tz = "America/Los_Angeles")) %>%
  group_by(id = ChildWelfareReferralId) %>%
  summarise(updatedAt = max(updatedAt.x)
            ,acceptance_dtg = min(acceptance_dtg)) %>%
  select(-updatedAt)
```

We then use almost identical logic to select the initial contact. 


```r
var_referral_contact_initial<- tbl(con, "ChildWelfareCaseNotes") %>%
  inner_join(tbl(con, "ChildWelfareCaseNoteTypes")
             ,by = c("typeId" = "id")) %>%
  as_data_frame() %>%
  filter(typeId == 1
         ,is.na(deletedAt.x)) %>%
  mutate(initial_contact_dtg = lubridate::ymd_hms(paste(date, time), tz = "America/Los_Angeles")) %>%
  group_by(id = ChildWelfareReferralId) %>%
  summarise(updatedAt = max(updatedAt.x)
            ,initial_contact_dtg = min(initial_contact_dtg)) %>%
  select(-updatedAt)
```

This leaves us with two variable tables: `var_referral_accept` and `var_referral_contact_initial`. The date-time-groups in each of these tables are then converted to lubridate intervals, and then to lubridate periods. 


```r
var_referral_initial_contact_period <- inner_join(var_referral_accept, var_referral_contact_initial) %>%
  mutate(initial_contact_interval = lubridate::interval(acceptance_dtg, initial_contact_dtg)
         ,initial_contact_period = lubridate::as.period(initial_contact_interval)) %>%
  select(id, initial_contact_period)
```

As of the date of this testing, the staging data actually result in a negative value for `ChildWelfareReferralId = 1`. This is not necessarily a problem, but we should ensure that users are informed that the system will not prevent them from entering erroneous case notes (e.g. initial contacts taking place prior to acceptance dates). The logic above (`updatedAt = max(updatedAt.x)`) should ensure that if this does happen, the performance metrics will reflect date corrections made by the user. 

## FPS/PFR Performance Metric 2 - MEASURABLE

### Average number of hours from first contact with family to first face to face meeting.

This variable is defined identically to the `var_referral_contact_initial` above. Here, we simply select case notes where `typeId == 2` to reflect initial face-to-face contacts. 


```r
var_referral_ftf_initial<- tbl(con, "ChildWelfareCaseNotes") %>%
  inner_join(tbl(con, "ChildWelfareCaseNoteTypes")
             ,by = c("typeId" = "id")) %>%
  as_data_frame() %>%
  filter(typeId == 2
         ,is.na(deletedAt.x)) %>%
  mutate(initial_ftf_dtg = lubridate::ymd_hms(paste(date, time), tz = "America/Los_Angeles")) %>%
  group_by(id = ChildWelfareReferralId) %>%
  summarise(updatedAt = max(updatedAt.x)
            ,initial_ftf_dtg = min(initial_ftf_dtg)) %>%
  select(-updatedAt)
```

Also similar to the above, we join the `var_referral_ftf_initial` to `var_referral_accept` and use lubridate to derive a period. The following chunk does not currently run on staging as we do not have any seeded data representing a referral which has been both accepted and received an initial FTF contact. 


```r
var_referral_initial_ftf_period <- inner_join(var_referral_accept, var_referral_ftf_initial) %>%
  mutate(initial_ftf_interval = lubridate::interval(acceptance_dtg, initial_ftf_dtg)
         ,initial_ftf_period = lubridate::as.period(initial_ftf_interval)) %>%
  select(id, initial_ftf_period)
```

## FPS/PFR Performance Metric 3 - MEASURABLE

### Percent of first face to face contacts made within 48 of service confirmation date initial contact with family.

As mentioned above, we cannot test this metric with staging data at present. However, this metric will be calculated using a conditional statement similar to the following chunk. 


```r
var_referral_initial_ftf_threshold <- var_referral_initial_ftf_period %>% 
  mutate(threshold_met = ifelse(initial_ftf_period <= lubridate::hours(48) & 
                                 initial_ftf_period > lubridate::hours(0), TRUE, FALSE)) %>%
  summarise(percent_threshold_met = 100*mean(threshold_met))
```

## FPS/PFR Performance Metric 4 - MEASURABLE

### Rate of family contact.

For this measure, we will proceed as we have previously by defining a new variable on the basis of casenote type. In this case, the weekly contact casenote type. 


```r
var_referral_contact_weekly <- tbl(con, "ChildWelfareCaseNotes") %>%
  inner_join(tbl(con, "ChildWelfareCaseNoteTypes")
             ,by = c("typeId" = "id")) %>%
  as_data_frame() %>%
  filter(typeId == 3
         ,is.na(deletedAt.x)) %>%
  mutate(initial_contact_dtg = lubridate::ymd_hms(paste(date, time), tz = "America/Los_Angeles")) %>%
  group_by(id = ChildWelfareReferralId) %>%
  summarise(updatedAt = max(updatedAt.x)
            ,initial_contact_dtg = min(initial_contact_dtg)) %>%
  select(-updatedAt)
```

NOTE: We should not be surprised if users end up requesting the ability to associate multiple casenote types to a particular casenote (e.g. `First Face to Face Meeting` AND `Weekly Meeting`). 

From here, we will need to create a date table for each referral spanning the acceptance date to the minimum of the most recent casenote version where `"typeId" = 6` (i.e. End of Service), and the date that the report is being run. 


