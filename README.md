# sARDen
sARDen(SAS Analysis Results Display engine) is a SAS toolkit for producing CDISC ARS–aligned Analysis Results Data (ARD). Inspired by the R pharmaverse cards schema, it respectfully mirrors ARD conventions for cross-language workflows.  
Macros cover summaries, tabulations, and hierarchical stacking, outputting long-format ARDs plus ARDM-like metadata via XATTR for traceability.  

<img width="360" height="360" alt="sARDen_small" src="https://github.com/user-attachments/assets/eb930364-b5eb-40f0-9a98-62f3362b8c03" />

## Test Data
We use ADSL and ADAE created as test data with the “sas_faker” package (https://github.com/Morioka-Yutaka/sas_faker),   
but you can freely substitute typical ADSL/ADAE datasets instead, so feel free to adapt that part as you like.
~~~sas
%loadPackage(sas_faker)
%sas_faker(
n_groups=3, 
n_per_group=50,
output_lib=WORK,
seed =123456,
create_dm = N,
create_ae = N,
create_sv =  N,
create_vs = N,
create_adsl = Y,
create_adae = Y,
create_advs = N
);
~~~~
[ADSL]  
<img width="767" height="176" alt="image" src="https://github.com/user-attachments/assets/af49ad5f-15c6-4156-ba91-ee336cb86317" />  
[ADAE]  
<img width="758" height="235" alt="image" src="https://github.com/user-attachments/assets/1a0beab0-c842-4124-ac21-2fa284a642a8" />  

## `%sard_summary()` macro <a name="sardsummary-macro-4"></a> ######
### Purpose:  
  Generate ARD-style summary statistics (long format) for continuous variables,  
  aligned to cards/ARS-like naming: group#, variable, context, stat_name, stat_label, stat, fmt_fun.  
  Can either compute statistics from raw data (data=) or post-process external stat data (statdata=).  
  
### Parameters:  
~~~text
  data=                 Source dataset for raw summaries (mutually exclusive with statdata=).  
  statdata=             Pre-computed statistics dataset (mutually exclusive with data=).  
  statdata_value=       Numeric value column in statdata to map into stat (default: COL1).  
  by=                   Grouping variables (space-separated). Must be non-empty for this version.  
  variable=             Analysis variables to summarize (space-separated). Required when data= is used.  
  statistic=            Statistics to keep (space-separated).  
                         Expected tokens (case-insensitive): N MEDIAN MEAN SD MIN MAX P25 P75 etc.  
  classdata=            Optional CLASSDATA= dataset for PROC SUMMARY to force level display.  
  context=              Context label stored in output (default: summary).  
  out=                  Output ARD dataset name.  
~~~
### Outputs:  
~~~text
  out= dataset with columns:  
    group1-groupN, group1_level-groupN_level, variable, context, stat_name, stat_label, stat, fmt_fun  
  Ordering variables_sort/statistic_sort used internally to preserve requested order.  
~~~

### Notes:  
  - When data= is used, fmt_fun is auto-derived from decimal precision (capped at 3 dp).  
  - When statdata= is used, fmt_fun is taken as-is; statdata must contain _NAME_ in "var_stat" form.  
  - stat_name "stddev" is normalized to "sd".  
  
### Example:  
~~~sas
  %sard_summary(  
    data=ADSL,  
    by=TRT01P,  
    variable=AGE WEIGHTBL,  
    statistic=N MEDIAN MIN MAX MEAN SD,  
    out=sard_summary_mean  
  );
~~~
<img width="1366" height="510" alt="image" src="https://github.com/user-attachments/assets/c7a4b6ca-0d36-4f98-930c-fdf0c6f2d09d" />  

~~~sas
proc sort data=ADSL out=_ADSL;
   by TRT01P;
run;
proc surveymeans data=_ADSL geomean  gmstderr ;
   by TRT01P;
  var WEIGHTBL HEIGHTBL;
  ods output GeometricMeans=geomean_1;
run;
proc sort data=geomean_1;
  by  TRT01P VarName;
run;
proc transpose data=geomean_1 out=geomean_2(rename=(_NAME_=__NAME_)) prefix=value;
  by TRT01P VarName;
run;
data geomean_3;
set geomean_2;
_NAME_=catx("_",varname,__NAME_);
fmt_fun ="2";
run;
%sard_summary(
  statdata = geomean_3,
  statdata_value = value1,
  by =TRT01P,
  variable = WEIGHTBL HEIGHTBL,
  statistic = GeoMean GMStdErr,
  out = sard_summary_geomean
)
~~~
<img width="1286" height="376" alt="image" src="https://github.com/user-attachments/assets/c6eefb3d-00a3-4248-900b-04a2e43a8e16" />

  
---
## `%sard_tabulate()` macro <a name="sardtabulate-macro-5"></a> ######
### Purpose:  
  Generate ARD-style tabulations (long format) for categorical variables,  
  producing n, BigN (denominator), and p (proportion), compatible with cards tabulate ARD.  
  
### Parameters:  
~~~text
  data=                   Source dataset.  
  variable=               Categorical variables to tabulate (space-separated). Required.  
  by=                     Optional grouping variables (space-separated).  
  statistic=              Stats to output (space-separated). Default: n P BigN.  
                           n    = count within variable level  
                           BigN = denominator for the by-group  
                           p    = n/BigN  
  denominator_dataset=    Optional dataset to define denominators explicitly.  
                           If blank, denominators are derived from data=.  
  classdata=              Optional CLASSDATA= for PROC SUMMARY to force level display.  
  context=                Context label stored in output (default: tabulate).  
  out=                    Output ARD dataset name.  
~~~

### Outputs:  
  out= dataset with columns:  
    group1-groupN, group1_level-groupN_level (if by provided),  
    variable, variable_level, context, stat_name, stat_label, stat, fmt_fun  
  fmt_fun:  
    n/BigN -> "0", p -> "xx.x" (intended display template).  
   
### Notes:  
  - If by= is empty, denominators are scalar and applied to all levels.  
  - variable_level stored as formatted value (vvalue).  
  - Assumes variable/bystats exist and are non-missing for numerator records.  
  
### Example:  
~~~sas
  %sard_tabulate(  
    data=ADSL,  
    by=TRT01P,  
    variable=SEX,  
    statistic=n P BigN,  
    out=sard_tabulate  
  );
~~~

<img width="1248" height="508" alt="image" src="https://github.com/user-attachments/assets/f1e5fd8c-4782-41df-b1a9-cfd266e3f803" />

  
---

## `%sard_stack_hierarchical()` macro <a name="sardstackhierarchical-macro-2"></a> ######
### Purpose:  
  Generate hierarchical ARD for rate-style stacking (subject-level unique counting).  
  Designed for event hierarchies like AE SOC/PT, producing overall + per-level summaries by group.  
  
### Parameters:
~~~text
  data=                   Source event dataset (e.g., ADAE).  
  variable=               Hierarchical variables (space-separated), ordered top->bottom (e.g., AEBODSYS AEDECOD).  
  variable_hieral_code=   Optional ordering code variables aligned with variable= list (space-separated).  
  by=                     Grouping variables (space-separated), e.g., TRTA.  
  id=                     Subject identifier for rate counting (unique per id within each hierarchy level). Required.  
  statistic=              Stats to output (space-separated). Default: n P BigN.  
  denominator_dataset=    Dataset to define denominators (usually ADSL). Required for p/BigN usage.  
  classdata=              Optional CLASSDATA= for PROC SUMMARY.  
  over_variables=         Y/N.  
                           Y -> include overall ("Any event") row via dummy variable.  
                           N -> no overall row.  
  out=                    Output ARD dataset name.  
~~~

### Outputs:  
  out= dataset with columns:  
    group#, group#_level, variable, variable_level, context="hierarchical",  
    stat_name, stat_label, stat, fmt_fun, plus optional variable_hieral_code columns.  
  Includes overall row where variable="hierarchical_overall", variable_level="Y" if over_variables=Y.  

### Notes:  
  - Subject-level de-duplication is enforced per (id, by, variable level).  
  - variable_hieral_code assumes same length as variable= and key uniqueness in data=.  
  - by= must be non-empty for this version.  
  
### Example:  
~~~sas
  %sard_stack_hierarchical(  
    data=ADAE,  
    variable=AEBODSYS AEDECOD,  
    variable_hieral_code=AEBDSYCD F_AEPTCD,  
    by=TRTA,  
    id=USUBJID,  
    denominator_dataset=ADSL(rename=(TRT01A=TRTA)),  
    out=sard_stack_hierarchical  
  );
~~~
<img width="1506" height="882" alt="image" src="https://github.com/user-attachments/assets/29edd77d-4653-4b47-a64e-2b513c4d30cb" />

  
---

## `%sard_stack_hierarchical_count()` macro <a name="sardstackhierarchicalcount-macro-3"></a> ######
### Purpose:  
  Generate hierarchical ARD for count-style stacking (record/event counting).  
  Similar to sard_stack_hierarchical, but does not de-duplicate by subject id.  
  
### Parameters:  
~~~text
  data=                   Source event dataset (e.g., ADAE).  
  variable=               Hierarchical variables (space-separated), ordered top->bottom.  
  variable_hieral_code=   Optional ordering code variables aligned with variable= list.  
  by=                     Grouping variables (space-separated).  
  statistic=              Stats to output (space-separated). Default: n.  
  denominator_dataset=    Dataset to define denominators (optional unless p/BigN requested).  
  classdata=              Optional CLASSDATA= for PROC SUMMARY.  
  over_variables=         Y/N to include overall dummy row.  
  out=                    Output ARD dataset name.  
~~~

### Outputs:  
  out= dataset with columns:  
    group#, group#_level, variable, variable_level, context="hierarchical",  
    stat_name, stat_label, stat, fmt_fun, plus optional variable_hieral_code columns.  
   
### Notes:  
  - Counts reflect number of records/events, not number of subjects.  
  - Denominator/pct layers are computed even if statistic excludes them (harmless overhead).  
  - variable_hieral_code assumes same length as variable=.  
  
### Example:  
~~~sas
  %sard_stack_hierarchical_count(  
    data=ADAE,  
    variable=AEBODSYS AEDECOD,  
    variable_hieral_code=AEBDSYCD F_AEPTCD,  
    by=TRTA,  
    statistic=n,  
    denominator_dataset=ADSL(rename=(TRT01A=TRTA)),  
    out=sard_stack_hierarchical_count  
  );
~~~

<img width="1448" height="826" alt="image" src="https://github.com/user-attachments/assets/9f2d6871-37d3-4938-b788-cdedd39861f7" />


---

## `%bind_ard()` macro <a name="bindard-macro-1"></a> ######
### Purpose:  
  Bind multiple ARD datasets (long format) into a single ARD,  
  optionally removing duplicates and normalizing column order to ARD-friendly sequence.  
  
### parameters:  
~~~text
  indata=          Space-separated list of ARD datasets to stack.  
  outdata=         Output dataset name.  
  distinct=        Y/N.  
                   Y -> remove duplicate rows across inputs (excluding dsno/record_no).  
                   N -> keep all rows.  
  drop_sortno=     Y/N.  
                   Y -> drop internal dsno and record_no from output.  
                   N -> keep them.  
~~~

### Outputs:  
  outdata= dataset containing stacked ARD rows.  
  Column order is set to: group# / group#_level / variable / variable_level / context /  
  stat_name / stat_label / stat / fmt_fun (any other columns follow).  
   
### Notes:  
  - dsno increments when CUROBS resets between input datasets.  
  - distinct=Y assumes compatible column structure across ARDs to avoid unintended row drops.  
  - Current version uses "informat &vsort;" as a placeholder; column order relies on DICTIONARY sequence.  
  
### Example:  
~~~sas
  %bind_ard(  
    indata=Sard_summary_mean Sard_tabulate Sard_stack_hierarchical_count,  
    outdata=ard_tab_14_2_1,  
    distinct=Y,  
    drop_sortno=Y  
  );
~~~
<img width="1352" height="846" alt="image" src="https://github.com/user-attachments/assets/1de779d4-081c-41e8-acac-3484abadbe04" />

  
---

## `%xrdm_set()` macro <a name="xrdmset-macro-6"></a> ######
### Purpose:  
  Attach ARDM/ARM-TS-like metadata to an ARD dataset using SAS extended attributes (XATTR).  
  Supports partial updates; only non-empty parameters are written.  
  
### Parameters:
~~~sas
  lib=              Library where ARD dataset resides (default: WORK).  
  ard=              ARD dataset name to modify. Required.  
  result_id=        Result identifier (e.g., R001).  
  analysis_id=      Analysis identifier (e.g., A001).  
  method_id=        Method identifier (e.g., M001).  
  result_context=   Free-text description of result context.  
  table_id=         Table/TFL identifier (e.g., T14.2.1).  
  display_label=    Free-text display label for output.  
  print=            Y/N.  
                    Y -> print ExtendedAttributesDS section via PROC CONTENTS after update.  
                    N -> no print.  
~~~

### Outputs:  
  Updates dataset-level extended attributes on &lib..&ard.  
  Optionally prints attribute listing to log/output.  
  
### Dependencies / Side Effects:  
  Uses PROC DATASETS MODIFY with XATTR SET DS.  
  Does not create intermediate datasets.  
  
### Notes:  
  - Free-text fields should be passed with %nrbquote(...) if they contain special characters.  
  - Existing keys are overwritten when re-specified.  
  
### Example:  
~~~sas
  %xrdm_set(  
    lib=WORK,  
    ard=ard_tab_14_2_1,  
    result_id=R001,  
    analysis_id=A001,  
    method_id=M001,  
    result_context=%nrbquote(Drug vs Placebo on CHG),  
    table_id=%nrbquote(T14.2.1),  
    display_label=%nrbquote(Mean Difference (Drug–Placebo))  
  );
~~~

<img width="732" height="406" alt="image" src="https://github.com/user-attachments/assets/35dbf20c-7751-4d56-832e-86dc18b43987" />

  
---

# version history<br>
0.1.0(01December2025): Initial version<br>

## What is SAS Packages?

The package is built on top of **SAS Packages Framework(SPF)** developed by Bartosz Jablonski.

For more information about the framework, see [SAS Packages Framework](https://github.com/yabwon/SAS_PACKAGES).

You can also find more SAS Packages (SASPacs) in the [SAS Packages Archive(SASPAC)](https://github.com/SASPAC).

## How to use SAS Packages? (quick start)

### 1. Set-up SAS Packages Framework

First, create a directory for your packages and assign a `packages` fileref to it.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~sas
filename packages "\path\to\your\packages";
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Secondly, enable the SAS Packages Framework.
(If you don't have SAS Packages Framework installed, follow the instruction in 
[SPF documentation](https://github.com/yabwon/SAS_PACKAGES/tree/main/SPF/Documentation) 
to install SAS Packages Framework.)

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~sas
%include packages(SPFinit.sas)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


### 2. Install SAS package

Install SAS package you want to use with the SPF's `%installPackage()` macro.

- For packages located in **SAS Packages Archive(SASPAC)** run:
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~sas
  %installPackage(packageName)
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- For packages located in **PharmaForest** run:
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~sas
  %installPackage(packageName, mirror=PharmaForest)
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- For packages located at some network location run:
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~sas
  %installPackage(packageName, sourcePath=https://some/internet/location/for/packages)
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  (e.g. `%installPackage(ABC, sourcePath=https://github.com/SomeRepo/ABC/raw/main/)`)


### 3. Load SAS package

Load SAS package you want to use with the SPF's `%loadPackage()` macro.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~sas
%loadPackage(packageName)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


### Enjoy!

