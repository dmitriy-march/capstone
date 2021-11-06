# capstone

//**load main ER data set**//

data MCD_DATA ;
	infile '/home/u49451409/MCDDATA.csv' dlm=',' firstobs=2;
	input Year	$ Primary_County	$ Dual_Eligible	 $ Major_Diagnostic_Category$	Episode_Disease_Category	$
	Beneficiaries_with_Condition	Beneficiaries_with_Admissions	Total_Inpatient_Admissions	
	Beneficiaries_with_ER_Visits	Total_ER_Visits
;
run;

//**filter main data set by depression**//


DATA MCD_DATA_sub ;
    SET  MCD_DATA ;
    WHERE Episode_Disease_Category like  "Dep%" ;
RUN;

//**load population data**//

data WORK.POP ;
	infile '/home/u49451409/nyc_pop_data.csv' dlm=',' firstobs=2;
	input Primary_County $ CENSUS2010POP;

run;


//**cross reference population data with ER MCD dataset**//

libname libref 'SAS-library';
proc sql ;
 		create table GEO_CROSS as 
      select   m.Primary_County,  CENSUS2010POP,  Beneficiaries_with_ER_Visits,
      Beneficiaries_with_Admissions, Beneficiaries_with_Condition, Total_ER_Visits, 
      Total_Inpatient_Admissions, Year, Dual_Eligible
          from  WORK.MCD_DATA_sub m inner join  WORK.POP g
           on m.Primary_County = g.Primary_County
     	  order by CENSUS2010POP desc;


//**add ER per ben, and Ip per ben**//

data GEO_CROSS_1;
	set  WORK.GEO_CROSS;
	ER_PER_BEN =  (Total_ER_Visits/Beneficiaries_with_ER_Visits);
	
data GEO_CROSS_FINAL;
	set  WORK.GEO_CROSS_1;
	IP_PER_BEN =  ( Total_Inpatient_Admissions/ Beneficiaries_with_Admissions);
	
	


//**first hypothesis test  non-dual**//	

proc logistic data=WORK.GEO_CROSS_FINAL;
model Dual_Eligible(event='Non-Dual')=ER_PER_BEN / link=logit technique=fisher;
run;

//**first hypothesis test  dual**//	
proc logistic data=WORK.GEO_CROSS_FINAL;
model Dual_Eligible(event='Dual')=ER_PER_BEN / link=logit technique=fisher;
run;

//**second hypothesis test - linear regression population to er visits**//

proc reg data=WORK.GEO_CROSS_FINAL alpha=0.05 plots(only)=(diagnostics 
		residuals fitplot observedbypredicted);
	model ER_PER_BEN=CENSUS2010POP /;
	run;
quit;


//**correlation graph population to er visits**//


proc sgplot data=WORK.GEO_CROSS_FINAL;
	scatter x=CENSUS2010POP y=ER_PER_BEN /;
	xaxis grid;
	yaxis grid;
run;


//**load income data**//

FILENAME REFFILE '/home/u49451409/nyc_inc_data.csv';
PROC IMPORT DATAFILE=REFFILE
	DBMS=CSV
	OUT=WORK.INC;
	GETNAMES=YES;
RUN;
PROC CONTENTS DATA=WORK.INC; RUN;



//**cross reference income data**//

libname libref 'SAS-library';
proc sql ;
 		create table INC_CROSS as 
      select   *
          from  WORK.MCD_DATA_sub m inner join  WORK.INC g
           on m.Primary_County = g.Primary_County
     	  order by Median_income desc;



//**third hypothesis test**// 

proc reg data=WORK.INC_CROSS alpha=0.05 plots(only)=(diagnostics residuals 
		fitplot observedbypredicted);
	model Beneficiaries_with_Condition=Median_income /;
	run;
quit;


//**correlation graph median income to beneficiaries with depression**//


proc sgplot data=WORK.INC_CROSS;
	scatter x=Median_income y=Beneficiaries_with_Condition /;
	xaxis grid;
	yaxis grid;
run;



//**summary data - dual & non-dual**//

//**non-dual**//

libname libref 'SAS-library';
proc sql ;
create table Non_Dual as 
 	     select    Dual_Eligible,  ER_PER_BEN
	FROM  WORK.GEO_CROSS_FINAL
	where  Dual_Eligible = 'Non-Dual'	;
     	  
proc sql;
    select avg(ER_PER_BEN) as Avg_ER
    from work.Non_Dual;
quit;

//**dual**//

libname libref 'SAS-library';
proc sql ;
create table Dual as 
 	     select    Dual_Eligible,  ER_PER_BEN
	FROM  WORK.GEO_CROSS_FINAL
	where  Dual_Eligible = 'Dual'	;
     	  
proc sql;
    select avg(ER_PER_BEN) as Avg_ER_2
    from work.Dual;
quit;









