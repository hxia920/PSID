# Clean PSID Using Stata

This is my note on cleaning the Panel Study of Income Dynamics (PSID) using Stata when I worked for [WiNDC, UW-Madison](https://windc.wisc.edu/) during summer, 2022. PSID is a powerful dataset but is hard to clean if you want to do a intergenerational analysis. During my learning process, I found a paper by [Hoynes et al. (2016)](https://www.aeaweb.org/articles?id=10.1257/aer.20130375). They cleaned PSID using Stata and kindly provided [replication code](https://www.openicpsr.org/openicpsr/project/112914/version/V1/view) for the cleaning and analysis process. This is by far the most efficient way I've seen cleaning PSID by Stata. Thus, I learned from their code, made marginal improvement, and wrote down this notes on how their code works. Initially this note is prepared for cases I forgot what's going on in the cleaning process, and I'd like to share it on GitHub. Hope this note can save your search time and help you learn PSID efficiently. Feel free to contact me if there are any problems in this note. 

## 1. Introduction of PSID

### 1.1 What is PSID?

Let's first look at the introduction from their [official website](https://psidonline.isr.umich.edu/):

> The Panel Study of Income Dynamics (PSID) is the longest running longitudinal household survey in the world.
>
> The study began in 1968 with a nationally representative sample of over 18,000 individuals living in 5,000 families in the United States. Information on these individuals and their descendants has been collected continuously, including data covering employment, income, wealth, expenditures, health, marriage, childbearing, child development, philanthropy, education, and numerous other topics. The PSID is directed by faculty at the University of Michigan, and the data are available on this website without cost to researchers and analysts.
>
> The data are used by researchers, policy analysts, and teachers around the globe. Over 6,800 peer-reviewed publications have been based on the PSID. Recognizing the importance of the data, numerous countries have created their own PSID-like studies that now facilitate cross-national comparative research. The National Science Foundation recognized the PSID as one of the [60 most significant advances funded by NSF](http://www.nsf.gov/news/news_summ.jsp?cntn_id=116902) in its 60 year history.

From the official introduction we can know that:

1. PSID has a long time span and it tracks its original samples repeatedly in every waves of the survey. This makes PSID a powerful **panel data** for economic and sociological analysis and causal inference.
1. PSID automatically absorbs descendants of its current samples, which makes **intergenerational analysis** possible.
1. PSID has representative samples and provides exhaustive information about U.S. households in various fields.

Sounds great! It seems that PSID is reliable and powerful. But wait, a question naturally occurs before we go on. We know these compelling information are hidden in the massive data, but how can we extract these information out of the maze? We need to first find a *map* of the maze. Level of variables in the data and identifiers plays an important role as a map in PSID (and almost all survey data). Hence, let's then get to know how PSID is organized in the next subsection.

### 1.2 How PSID is Organized?

#### 1.2.1 Individual-level & Family-level Data

Generally speaking, PSID main data can be split into two levels: **individual level** and **family level**. Data from supplement survey and other sources are not discussed here. 

For **individual-level** data, the [PSID official user guide](https://psidonline.isr.umich.edu/data/Documentation/UserGuide2019.pdf) states:

> The cross-year individual file contains one record for each individual present in an interviewed family in any survey year.

Individual-level data not only provides information about respondents of the interview (typically  **Reference Person** (or Head before 2017) and **Reference Person's Spouse/Partner** ), but also information about other family members. However, individual-level data of family members only provides limited information. Only basic questions about family members are asked during the interview because usually these questions are answered by respondents of the family, not themselves. Nevertheless, individual data still provides great chance for intergenerational studies. According to the sampling algorithm of PSID, (male) descendants will automatically get enrolled in the survey and become reference people of new households when they split from their original family and get married. Individual-level data starts to record their information before they become respondents, which makes comparisons between their adulthood lives/their new families and their childhood lives/original families possible.

***

For **family-level** data, the [PSID official user guide](https://psidonline.isr.umich.edu/data/Documentation/UserGuide2019.pdf) states:

>The family file contains one record for each family unit interviewed in a given year. It includes all family level variables collected in that year, as well as extensive information about the Reference Person (starting with the 2017 wave, the term ‘Reference Person’ has replaced ‘Head’) and the Spouse/Partner. Therefore, the content of the family file is not restricted to family-level data. 

 Compared to Individual-level data, family-level data provides more information since respondents are the ones who know their families best. Family-level data not only contains information about the family as a whole, but also provides more detailed information about the **Reference Person** and **Reference Person's Spouse/Partner**. Combined with individual-level data, PSID can be used to conduct lots of analyses.

#### 1.2.2 Identifiers

For identifiers, PSID provides excellent videos. Everything about identifiers will be clear after watching these videos. Related videos can be found at [PSID Official YouTube Channel](https://www.youtube.com/channel/UCRKjqCqPGtOZommz9SyaCyQ) and can be watched below.
```html
<iframe width="560" height="315" src="https://www.youtube.com/embed/sYMr6YYhK_w" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
```
To conclude, PSID has following identifiers:
 - 1968 Interview Number + Person Number to identify a specific person in all waves
 - Year-specific Family Interview Number to identify a specific family in a specific year
 - Year-specific Family Interview Number +  Year-specific Sequence Number to identify a specific person in a specific year

With all these above in mind, we can move forward to Stata part.

## 2. An Efficient Way of Cleaning PSID Using Stata

### 2.1 Basic Stata Settings Before Data Cleaning

**Macros** are temporary variables which can be referred to in the current code. Global macros can be accessed everywhere in the code. Local macros can only be accessed in current loops or functions. A common usage of global macros is to set working directories. Here's an example:

```stata
global root= "D:\PSID"
global dofiles= "$root\Dofiles"
global logfiles= "$root\Logfiles"
global raw_data= "$root\Raw_data"
global working_data= "$root\Working_data"
global temp_data= "$root\Temp_data"
global tables= "$root\Tables"
global figures= "$root\Figures"
```

Set working directories using global macros helps to avoid repeatedly enter full file or folder path when accessing data or saving results.

***

**Loop** is the most efficient tool to do tedious repeated work in Stata. The three most common loops in Stata are `while`, `forvalues`,  and `foreach`. Typically,  `while` and `forvalues` are used to loop over values, and `foreach` is used to loop over variables and files. When combined with macros, loop becomes super powerful. Examples will be shown in next subsection.

### 2.2 PSID Cleaning Process

#### 2.2.1 Individual-level Data

The good thing is, individual-level data is stored in only one data file, which means a loop over files is not needed and identifiers are much simpler. 

To start cleaning individual-level data, we first read in raw data using the do-file provided by PSID (here 2019 individual-level data is used):

```stata
do $dofiles\IND2019ER.do
```

Then, we need to generate identifiers for merge and loop. As mentioned before, we need to have **1968 Interview Number**, **1968 Person Number**, **Year-specific Family Interview Number**, and **Year-specific Sequence Number**.  Among them, 1968 Person Number is just a existed variable `ER30002`.  Other identifiers are generated as follows.

***

For Year-specific Interview Number, we can use the following loop:

```stata
local n = 1
foreach y of numlist 1968(1)1997 1999(2)2019 {
	local inumVars: word `n' of ER30001 ER30020 ER30043 ER30067 ER30091 ER30117 ER30138 ER30160 ER30188 ER30217 ER30246 ER30283 ER30313 ER30343 ER30373 ER30399 ER30429 ER30463 ER30498 ER30535 ER30570 ER30606 ER30642 ER30689 ER30733 ER30806 ER33101 ER33201 ER33301 ER33401 ER33501 ER33601 ER33701 ER33801 ER33901 ER34001 ER34101 ER34201 ER34301 ER34501 ER34701
	gen inum`y' = `inumVars'
	local n = `n' + 1 
}
```

In this loop, we create three local macros: `n`, `y`, and `inumVars`. For each wave `y`, we extract the `n`^th^ word in `inumVars` and use this to generate a set of new variables named as *inum + year* . The way we name these variables is important for long-wide transformation later. Among these generated variables, `inum1968` is the 1968 Interview Number.

***

For Year-specific Sequence Number, we can use the following loop:

```stata
local n = 1
foreach y of numlist 1968(1)1997 1999(2)2019 {
	local seqVars: word `n' of ER30003 ER30021 ER30044 ER30068 ER30092 ER30118 ER30139 ER30161 ER30189 ER30218 ER30247 ER30284 ER30314 ER30344 ER30374 ER30400 ER30430 ER30464 ER30499 ER30536 ER30571 ER30607 ER30643 ER30690 ER30734 ER30807 ER33102 ER33202 ER33302 ER33402 ER33502 ER33602 ER33702 ER33802 ER33902 ER34002 ER34102 ER34202 ER34302 ER34502 ER34702
	gen seqnum`y' = `seqVars'
	local n = `n' + 1
}
```

Note that there was no Sequence Number in 1968, so we use the relation to the head of the household instead. Hence, all identifiers are generated.

***

To clean individual-level variables that change over time, we can write similar loops. Here we use `age` as an example:

```stata
local n = 1
foreach y of numlist 1968(1)1997 1999(2)2019 {
	local ageVars: word `n' of ER30004 ER30023 ER30046 ER30070 ER30094 ER30120 ER30141 ER30163 ER30191 ER30220 ER30249 ER30286 ER30316 ER30346 ER30376 ER30402 ER30432 ER30466 ER30501 ER30538 ER30573 ER30609 ER30645 ER30692 ER30736 ER30809 ER33104 ER33204 ER33304 ER33404 ER33504 ER33604 ER33704 ER33804 ER33904 ER34004 ER34104 ER34204 ER34305 ER34504 ER34704
	gen age`y' = `ageVars'
	local n = `n' + 1
}
```

***

Besides, create a variable indicating **Reference Person** and  **Reference Person's Spouse/Partner** is essential for future cleaning. We can do this using the following loop:

```stata
local n = 1
foreach y of numlist 1968(1)1997 1999(2)2019 {
	local headVars: word `n' of ER30003 ER30022 ER30045 ER30069 ER30093 ER30119 ER30140 ER30162 ER30190 ER30219 ER30248 ER30285 ER30315 ER30345 ER30375 ER30401 ER30431 ER30465 ER30500 ER30537 ER30572 ER30608 ER30644 ER30691 ER30735 ER30808 ER33103 ER33203 ER33303 ER33403 ER33503 ER33603 ER33703 ER33803 ER33903 ER34003 ER34103 ER34203 ER34303 ER34503 ER34703
	gen head`y' = .
	if `y'>=1983{
		replace head`y' = 1 if `headVars' == 10 & (seqnum`y'>=1 & seqnum`y'<=20)
		replace head`y' = 2 if (`headVars' == 20 | `headVars' == 22) & (seqnum`y'>=1 & seqnum`y'<=20)
		}
	if `y'<1983{
		replace head`y' = 1 if `headVars' == 1 & (seqnum`y'>=1 & seqnum`y'<=20)
		replace head`y' = 2 if `headVars' == 2 & (seqnum`y'>=1 & seqnum`y'<=20)
		}
	local n = `n' + 1
}
```

By using all loops above, we can finish our cleaning of individual-level data.



#### 2.2.2 Family-level Data

The cleaning of family-level data is much more complicated since we only have separate data files for each year. It would be almost impossible to do all the work manually without loops. We need to find a way to access and extract certain variables across all waves properly. Basically speaking, we need to run a loop over all waves of family-level data. For each wave, we read in the data file, extract variables of our interests, reshape extracted data, and save it as a temporary data file. In the end, we loop over these temporary data file to merge them into one final family-level data cross waves. 

Write a loop to read in raw data and save processed files is relatively easy. We can do this using the following loop:

```stata
local col = 3
foreach y of numlist 1968(1)1997 1999(2)2019 {
	if `y' < 1994 {
		qui do $dofiles\FAM`y'.do
	}
	if `y' >= 1994 {
		qui do $dofiles\FAM`y'ER.do
	}

* =========================================================================
*
* SOME CODE HERE
*
* =========================================================================

	compress
	save $temp_data\psid_fam_temp_`y'.dta, replace
	clear
	
* =========================================================================
* Process the variables for next year
local col = `col' + 1	
	
}
```

The difficult part is to write the code in the middle. Variable names and identifiers are all different across waves. How can we extract an specific variable in a certain year? We might think of macros. We can store a set of variable names in a macro, and use a loop to extract every variable in the macro. However, macros only have one dimension. Only year or variable names can be stored in one macro, not both of them. A natural way to think of this question a step further is to create a two-dimensional macro. Macro itself does not support dimensions more than one. However, we can nest a set of macros into another macro to create a macro of macros. Then, the outer macro has two dimensions. In particular, we first create a set of macros like this in a new do-file (here I call it `01.5psid_fam_vars.do`):

```stata
local fam1	"	VAR			HEAD	1968	1969	1970	1971	1972	1973	1974	1975 "
local fam2	"	inum		1		V3		V442	V1102	V1802	V2402	V3002	V3402	V3802 "
local fam3	"	inum		2		V3		V442	V1102	V1802	V2402	V3002	V3402	V3802 "
local fam4	"	inifamid	1		V3		V534	V1230	V1932	V2533	V3085	V3497	V3909 "
```

This process is like drawing a table. We set proper column names and row names. Then, we will be able to access each element in this table using the two names. Hence, we can update our previous loop as follows:

```stata
local col = 3
foreach y of numlist 1968(1)1997 1999(2)2019 {
	if `y' < 1994 {
		qui do $dofiles\FAM`y'.do
	}
	if `y' >= 1994 {
		qui do $dofiles\FAM`y'ER.do
	}

* =========================================================================
* Read 	in the macro matrix	
	qui include $dofiles\01.5psid_fam_vars.do

* =========================================================================
* Process identifiers(interview id and 1968 id)
	forvalues row = 2(2)4{
		local year: word `col' of `fam1'
		local name: word 1 of `fam`row''
		local value:word `col' of `fam`row''
		capture gen `name'`year' = `value'
		local famIDVars`y' "`famIDVars`y'' `name'`year'"
		macro drop name year value
	}

* =========================================================================
* Save
	compress
	save $temp_data\psid_fam_temp_`y'.dta, replace
	clear
}
```

In the updated code, we extract variables by their column names `year` and row names `fam1` to `fam4`. Original variable names are the first elements of rows. Generated variables are named as *original variable name + year*. For some other variables, we need to know whether the variable records the information about Reference Person or Reference Person's Partner/Spouse. For these variables, just replace the `forvalues` loop with the following code:

```stata
forvalues row = 2(2)4{
		local year: word `col' of `fam1'
		local name: word 1 of `fam`row''
		local head: word 2 of `fam`row''
		local value:word `col' of `fam`row''
		gen `name'`year'`head' = `value'
		local famIDVars`y' "`famIDVars`y'' `name'`year'`head'"
		macro drop name head year value
	}
```

Note that if variable generation uses the macro `head` , then the generated data needs to be reshaped using following code:

```stata
local reshapeVars`y' ""
	forvalues row = 2(2)4{
		local name: word 1 of `fam`row''
		local reshapeVars`y' "`reshapeVars`y'' `name'`y'"
		macro drop name
	}
reshape long `reshapeVars`y'', i(inum`y') j(head`y')
```

By using all loops above, we can finish our cleaning of family-level data.

#### 2.2.3 Merge the Two Cleaned Dataset

Now both individual-level data and family-level data is cleaned. We can merge all cleaned data for a full dataset. 

First, we need to merge individual-level data with all cleaned family-level temporary files in to one:

```stata
use $temp_data\psid_indi_temp.dta, clear

foreach y of numlist 1968(1)1997 1999(2)2019 {
	sort inum`y' head`y'
	merge m:1 inum`y' head`y' using $temp_data\psid_fam_temp_`y'
	drop if _merge == 2
	drop _merge
```

Then, we need to reshape the data once again to obtain a final long-format panel data. This can be done as follows:

```stata
qui include $dofiles\01.5psid_fam_vars.do

local reshapeVars ""

forvalues row = 2(2)4 {
	local name: word 1 of `famDemo`row''
	local reshapeVars`y' "`reshapeVars`y'' `name'"
	macro drop name
}

gen inumID = inum1968
ren person1968 personID
reshape long `reshapeVars', i(inumID personID) j(year)
```

Now, we have successfully merged individual-level and family-level data into one. Note that every person in PSID sample will have a 1968 Interview ID and a 1968 Person Number even if  they were not in the sample at that time. Don't forget to exclude these people before they enter the sample. Once all variables are recoded according to the codebook, the  general cleaning process will be done and the data will be ready to use after personalized adjustments.
