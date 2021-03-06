---
title: "Examine the Predictiveness of Teacher Effects"
output: 
  html_document:
    theme: simplex
    css: styles.css
    highlight: NULL
    keep_md: true
    toc: true
    toc_depth: 4
    toc_float: true
    number_sections: false
---





<div class="navbar navbar-default navbar-fixed-top" id="logo">
<div class="container">
<img src="OpenSDP-Banner_crimson.jpg" style="display: block; margin: 0 auto; height: 115px;">
</div>
</div>

[OpenSDP Analysis](http://opensdp.github.io/analysis) / [Human Capital Analysis: Evaluation](Human_Capital_Analysis_Evaluation.html) / Examine the Predictiveness of Teacher Effects

![](Predictive_Teacher_Effects_Average_Middle School Math.png)

###Preparation
####Purpose

To show the extent to which prior estimates of a teacher's effectiveness predict effectiveness in future years.

####Required analysis file variables

 - `tid`
 - `school_year`
 - `school_lvl`
 - `curr2year_tre_m`
 - `current_tre_m`


####Analysis-specific sample restrictions

 - Keep only records for teachers with three years of effectiveness estimates in the given subject.  
 - If school_level restriction is chosen, keep only records for elementary or middle school teachers. 
 
 
####Ask yourself
 
- Value-added estimates are more likely to be accurate for teachers at the high and low ends of the distribution. What kinds of decisions would knowing which teachers consistently perform at the top and bottom ends of the distribution of value-added estimates help your agency make? How might this information be used for probationary teachers? For veteran teachers? 
 
####Potential further analyses

If you have sufficient sample space, you can restrict the sample to teachers in their third year at the agency to 
understand how performance of probationary teachers in their first two years compares to performance in the third year.

###Analysis

####Step 1: Choose the subject and school level. 

Choose the subject (math [m] or ela [e]) and school level (elem or middle) for the analysis. Note: to make multiple charts at the same time, put loops for subject and level around the analysis and graphing code. To include all grade levels in the analysis, comment out the local level command below.


```stata
local subject m
local level middle
```

####Step 2: Load data.


```stata
use "${analysis}/Teacher_Year_Analysis.dta", clear
isid tid school_year
```

####Step 3: Restrict the sample.

Keep years for which teacher effects value added estimates are available. Keep only records for which teachers have pooled teacher effects estimates (pooled estimates use information from all available years for each teacher). If school level restriction is chosen, keep only records from either elementary or middle schools.


```stata
keep if school_year >= 2012 & school_year <= 2015
keep if !missing(current_tre_`subject')
keep if !(sch_high == 1)
if "`level'" == "elem" {	
	keep if sch_elem == 1
}
if "`level'" == "middle" {
	keep if sch_middle == 1
}
```

####Step 4: Review variables.


```stata
tab school_year
summ current_tre_`subject', detail 
summ curr2year_tre_`subject', detail
```


####Step 5: Create "year 3".

Identify the most recent year a teacher is present in the data and tag as "year 3."


```stata
egen max_school_year = max(school_year), by(tid)
gen year3 = max_school_year == school_year
drop max_school_year
tab year3, mi
```

####Step 6: Create time series. 

Set time series structure and use lead operators to identify years 2 and 1. 


```stata
tsset tid school_year	
gen year1 = 0
gen year2 = 0
bysort tid: replace year2 = 1 if F.year3 == 1
bysort tid: replace year1 = 1 if F.year2 == 1
tab year2 year3, mi
tab year1 year3, mi
```

####Step 7: Keep balanced panel. 

Keep a balanced panel which includes only teachers with observations for all 3 years.


```stata
bysort tid: egen balanced = max(year1)
keep if balanced == 1
drop balanced
unique tid
```

####Step 8: Assign teacher quartiles.

Assign teachers to quartiles based on two-year pooled teacher effects in year 2, and generate dummy variables for quartiles.


```stata
assert !missing(curr2year_tre_`subject') if year2 == 1
xtile quart_temp = curr2year_tre_`subject' if year2 == 1, nq(4)
bysort tid: egen quart = max(quart_temp)
tab quart if year2 == 1, mi
tab quart, gen(quart)
```

####Step 9: Drop extra records. 

Drop records for years 1 and 2, reducing data to one record per teacher.


```stata
keep if year3 == 1
isid tid
```

####Step 10: Get sample size. 


```stata
sum tid
local unique_teachers = string(r(N), "%9.0fc")
```

####Step 11: Get significance. 


```stata
gen sig = 0
forval quartile = 1/4 {
	reg current_tre_`subject' quart`quartile', robust
	test _b[quart`quartile'] == 0
	gen sig`quartile' = r(p) < .05
	replace sig = sig`quartile' if quart`quartile' == 1
}
```

####Step 12: Collapse the data for graphing. 


```stata
collapse (mean) current_tre_`subject' sig, by(quart) 
```

####Step 13: Concatenate value labels and significance asterisks.

```stata
gen tre_str = string(current_tre_`subject', "%9.3f")
gen star = ""
forval quartile = 1/4 { 
	replace star = "*" if quart == `quartile' & sig == 1 
} 
egen tre_label = concat(tre_str star), format(%9.3f)	
```

####Step 14: Define subject and school level titles.

```stata
if "`subject'" == "m" {
	local subj_foot "math"
	local subj_title "Math"
}

if "`subject'" == "e" {
	local subj_foot "English/Language Arts"
	local subj_title "ELA"
}
local gradespan "5th through 8th"

if "`level'" == "middle" {
	local subj_title "Middle School `subj_title'"
	local gradespan "6th through 8th"
}

if "`level'" == "elem" {
	local subj_title "Elementary School `subj_title'"
	local gradespan "5th"
}	
```

####Step 15: Make chart.

```stata
#delimit ;
twoway (bar current_tre_`subj' quart if quart == 4, 
		horizontal fcolor(dkorange) lcolor(dkorange) lwidth(0))
	(bar current_tre_`subj' quart if quart == 3, 
		horizontal fcolor(forest_green) lcolor(forest_green) lwidth(0))
	(bar current_tre_`subj' quart if quart == 2, 
		horizontal fcolor(maroon) lcolor(maroon) lwidth(0))
	(bar current_tre_`subj' quart if quart == 1, 
		horizontal fcolor(dknavy) lcolor(dknavy) lwidth(0))
	(scatter quart current_tre_`subj' if current_tre_`subj' >= 0,
		mlabel(tre_label) msymbol(i) mlabpos(3) mlabcolor(black) mlabgap(.2))
	(scatter quart current_tre_`subj' if current_tre_`subj' < 0,
		mlabel(tre_label) msymbol(i) mlabpos(9) mlabcolor(black) mlabgap(.2)),
	title("`subj_title' Teacher Effects in Third Year", span)
	subtitle("by Quartile Rank During Previous Two Years", span)
	xtitle("Current Average Teacher Effect", size(medsmall))
		xscale(range(-0.15 (.05) 0.15))
		xlabel(-0.15 (.05) 0.15, format(%9.2f) labsize(medsmall)) 
	ytitle("Previous Teacher Effects Quartile", size(medsmall))
		yscale(range(1(1)4))
		ylabel(1 `""Least" "Effective""' 2 "2nd" 3 "3rd" 4 `""Most" "Effective""', 
		labsize(medsmall) nogrid)
	legend(off)
	graphregion(color(white) fcolor(white) lcolor(white))
	plotregion(color(white) fcolor(white) lcolor(white) margin(5 5 2 2))
	
	note(" " "*Significantly different from zero, at the 95 percent confidence level." 
"Notes: Sample includes `unique_teachers' `gradespan' grade `subj_foot' teachers
with three years of teacher effects estimates in school" "years 2011-12 through 2014-15.
Teacher effects are measured in student test score standard deviations, with
teacher-specific" "shrinkage factors applied to adjust for differences in sample
reliability.", 
	span size(vsmall));
#delimit cr
```

####Step 16: Save chart.

```stata
graph save "${graphs}\Predictive_Tchr_Effects_Avg_`subj_title'.gph" , replace
graph export "${graphs}\Predictive_Tchr_Effects_Avg_`subj_title'.emf" , replace
```



---

Previous Analysis: [Examine the Distribution of Teacher Effects](Distribution_of_Teacher_Effects.html)

Next Analysis: [Compare Teacher Effects for Teachers Ranked Most and Least Effective in the Prior Two Years](Predictive_Teacher_Effects_Distribution.html)
