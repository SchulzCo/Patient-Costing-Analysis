This project consists of a custom-built Qlik Sense application designed to analyze and optimize patient costing data for healthcare decision-making. The app includes dynamic dashboards, KPIs, and automated visualizations that provide insights into operational costs, patient categories, and service utilization.

### Key Features:
- 🚑 Multidimensional analysis of patient costs across departments and treatment types.
- 📈 Strategic KPIs for identifying cost drivers, inefficiencies, and improvement opportunities.
- 🧠 Integration of external data sources (Excel/SQL) for flexible and scalable exploration.
- 🎯 Interactive visualizations and filters allowing users to drill down into cost metrics.
- ⚙️ Automated load scripts and data model optimized for performance and maintenance.

### Technologies:
- Qlik Sense | Qlik Script | Data Modeling | Set Analysis | KPI Design | Excel | SQL

This app empowers stakeholders to make informed decisions on resource allocation, patient care optimization, and financial planning within healthcare environments.



⚙️ </> Developed Script </> 
-------------------------------------

Set dataManagerTables = '','Patient Details','AdmissionMethod','Resources','DischargeMethods','HealthServiceCalendar','HCResGroup','Gender','Specialties';

For each name in $(dataManagerTables) 
    Let index = 0;
    Let currentName = name; 
    Let tableNumber = TableNumber(name); 
    Let matches = 0; 
    Do while not IsNull(tableNumber) or (index > 0 and matches > 0)
        index = index + 1; 
        currentName = name & '-' & index; 
        tableNumber = TableNumber(currentName) 
        matches = Match('$(currentName)', $(dataManagerTables));
    Loop 
    If index > 0 then 
            Rename Table '$(name)' to '$(currentName)'; 
    EndIf; 
Next; 
Set dataManagerTables = ;


Unqualify *;

__countryAliasesBase:
LOAD
	Alias AS [__Country],
	ISO3Code AS [__ISO3Code]
FROM [lib://__GEO_TABLES/countryAliases.qvd]
(qvd);

__countryGeoBase:
LOAD
	ISO3Code AS [__ISO3Code],
	ISO2Code AS [__ISO2Code],
	Polygon AS [__Polygon]
FROM [lib://__GEO_TABLES/countryGeo.qvd]
(qvd);

__cityAliasesBase:
LOAD
	Alias AS [__City],
	geoKey AS [__geoKey],
	CountryCode AS [__CityCountryCode]
FROM [lib://__GEO_TABLES/cityAliases.qvd]
(qvd);

__cityGeoBase:
LOAD
	geoKey AS [__geoKey],
	geoPoint AS [__GeoPoint]
FROM [lib://__GEO_TABLES/cityGeo.qvd]
(qvd);

__countryName2IsoThree:
MAPPING LOAD
	__Country,
	__ISO3Code
RESIDENT __countryAliasesBase;

__countryCodeIsoThree2Polygon:
MAPPING LOAD
	__ISO3Code,
	__Polygon
RESIDENT __countryGeoBase;

__countryCodeAndCityName2Key:
MAPPING LOAD
	__CityCountryCode & __City,
	__geoKey
RESIDENT __cityAliasesBase;

__cityKey2GeoPoint:
MAPPING LOAD
	__geoKey,
	__GeoPoint
RESIDENT __cityGeoBase;

[PatientTypeMapping]:
MAPPING LOAD * INLINE
[
PatientTypeMapping-FROM,PatientTypeMapping-TO
NonElective,Non Elective
DayCase,Day Case
];
[Patient Details]:
LOAD
	[DateKey],
	[AdmissionMethod],
	[Age],
	[Consultant],
	[Consultant Comments],
	[Cost],
	[cResElem],
	[cResElem2Name],
	If (Match([cResElem3], 'null'), NULL(), [cResElem3]) AS [cResElem3],
	[cResElemName],
	[cResElem3Name],
	[cResItem],
	[DischargeMethod],
	[DominantDiagnosis],
	[DominantProcedure_Code],
	[AdmissionDate],
	[DischargeDate],
	[EpisodeID],
	[HRG] AS [HRG-sid_HRGKey],
	[IncomeChildTopup],
	[IncomeExcessBedDays],
	[IncomeTariff],
	If (Match([IncomeTopUp], '0'), NULL(), [IncomeTopUp]) AS [IncomeTopUp],
	[PatientID],
	APPLYMAP( 'PatientTypeMapping', [PatientType]) AS [PatientType],
	[PercentageSplit],
	[resItemName],
	[GenderId],
	[Specialty],
	[Units(days)],
	[Country],
	[State],
	[City],
	APPLYMAP( '__countryCodeIsoThree2Polygon', APPLYMAP( '__countryName2IsoThree', LOWER([Country])), '-') AS [Patient Details.Country_GeoInfo],
	APPLYMAP( '__cityKey2GeoPoint', APPLYMAP( '__countryCodeAndCityName2Key', APPLYMAP( '__countryName2IsoThree', LOWER([Country])) & LOWER([City])), '-') AS [Patient Details.City_GeoInfo],
	Today() -[DischargeDate] AS [DaysSinceDischarge],
	[DischargeDate] - [AdmissionDate] AS [Length Of Stay],
	If([Age] < 12, Dual('0-11', 1),If([Age] >= 12 and [Age] < 18, Dual('12-17', 2),If([Age] >= 18 and [Age] < 30, Dual('18-29', 3),If([Age] >= 30 and [Age] < 50, Dual('30-49', 4),If([Age] >= 50 and [Age] < 65, Dual('50-64', 5),If([Age] >= 65 and [Age] < 75, Dual('65-74', 6),If([Age] >= 75 and [Age] < 85, Dual('75-84', 7),If([Age] >= 85, Dual('85+', 8))))))))) AS [Age Group]
 FROM [lib://Section2SampleDataFiles/PatientDetails.xlsx]
(ooxml, embedded labels, table is [Patient Details]);

[AdmissionMethod]:
LOAD
	[AdmissionMethod],
	[Admission Type],
	[Admission Desc Group],
	[Admission Description]
 FROM [lib://Section2SampleDataFiles/AdmissionMethods.xlsx]
(ooxml, embedded labels, table is AdmissionMethod);

[Resources]:
LOAD
	[cResElem],
	[cResElementName]
 FROM [lib://Section2SampleDataFiles/Resources.xlsx]
(ooxml, embedded labels, table is Resources);

[DischargeMethods]:
LOAD
	[DischargeMethod],
	[Discharge Description],
	[Discharge Type]
 FROM [lib://Section2SampleDataFiles/DischargeMethods.xlsx]
(ooxml, embedded labels, table is DischargeMethods);

[HealthServiceCalendar]:
LOAD
	[DateKey],
	[DayAbbr],
	[MonthDayNum],
	[YearNum],
	[YearMonth],
	[MonthAbbr],
	[MonthNum],
	[DayDate],
	[MonthStartDate],
	[MonthEndDate],
	[PrevYearNum],
	[PrevMonthNum]
 FROM [lib://Section2SampleDataFiles/HealthServiceCalendar.xlsx]
(ooxml, embedded labels, table is HealthServiceCalendar);

[HCResGroup]:
LOAD
	[sid_HRGKey] AS [HRG-sid_HRGKey],
	[HRG Label]
 FROM [lib://Section2SampleDataFiles/HCResGroup.xlsx]
(ooxml, embedded labels, table is HCResGroup);

[Gender]:
LOAD
	[GenderDetails],
	Trim(left([GenderDetails], 1)) AS [GenderId],
	Trim(mid([GenderDetails], 1 + 1)) AS [Gender]
 FROM [lib://Section2SampleDataFiles/Gender.txt]
(txt, codepage is 28591, embedded labels, delimiter is spaces, msq);

[Specialties]:
LOAD
	[SpecialtyDescription] AS [Specialty Description],
	[Surgical Specialties],
	[Comments],
	Trim(left([SpecialtyDescription], 3)) AS [Specialty],
	Trim(left(Trim(mid([SpecialtyDescription], 3 + 1)), 22)) AS [Specialty.Description]
 FROM [lib://Downloads/Section2SampleDataFiles/Specialties.xlsx]
(ooxml, embedded labels, table is Specialties);



TAG FIELD [Country] WITH '$geoname', '$relates_Patient Details.Country_GeoInfo';
TAG FIELD [Patient Details.Country_GeoInfo] WITH '$geopolygon', '$hidden', '$relates_Country';
TAG FIELD [City] WITH '$geoname', '$relates_Patient Details.City_GeoInfo';
TAG FIELD [Patient Details.City_GeoInfo] WITH '$geopoint', '$hidden', '$relates_City';

DROP TABLES __countryAliasesBase, __countryGeoBase, __cityAliasesBase, __cityGeoBase;
[autoCalendar]: 
  DECLARE FIELD DEFINITION Tagged ('$date')
FIELDS
  Dual(Year($1), YearStart($1)) AS [Year] Tagged ('$axis', '$year'),
  Dual('Q'&Num(Ceil(Num(Month($1))/3)),Num(Ceil(NUM(Month($1))/3),00)) AS [Quarter] Tagged ('$quarter', '$cyclic'),
  Dual(Year($1)&'-Q'&Num(Ceil(Num(Month($1))/3)),QuarterStart($1)) AS [YearQuarter] Tagged ('$yearquarter', '$qualified'),
  Dual('Q'&Num(Ceil(Num(Month($1))/3)),QuarterStart($1)) AS [_YearQuarter] Tagged ('$yearquarter', '$hidden', '$simplified'),
  Month($1) AS [Month] Tagged ('$month', '$cyclic'),
  Dual(Year($1)&'-'&Month($1), monthstart($1)) AS [YearMonth] Tagged ('$axis', '$yearmonth', '$qualified'),
  Dual(Month($1), monthstart($1)) AS [_YearMonth] Tagged ('$axis', '$yearmonth', '$simplified', '$hidden'),
  Dual('W'&Num(Week($1),00), Num(Week($1),00)) AS [Week] Tagged ('$weeknumber', '$cyclic'),
  Date(Floor($1)) AS [Date] Tagged ('$axis', '$date', '$qualified'),
  Date(Floor($1), 'D') AS [_Date] Tagged ('$axis', '$date', '$hidden', '$simplified'),
  If (DayNumberOfYear($1) <= DayNumberOfYear(Today()), 1, 0) AS [InYTD] ,
  Year(Today())-Year($1) AS [YearsAgo] ,
  If (DayNumberOfQuarter($1) <= DayNumberOfQuarter(Today()),1,0) AS [InQTD] ,
  4*Year(Today())+Ceil(Month(Today())/3)-4*Year($1)-Ceil(Month($1)/3) AS [QuartersAgo] ,
  Ceil(Month(Today())/3)-Ceil(Month($1)/3) AS [QuarterRelNo] ,
  If(Day($1)<=Day(Today()),1,0) AS [InMTD] ,
  12*Year(Today())+Month(Today())-12*Year($1)-Month($1) AS [MonthsAgo] ,
  Month(Today())-Month($1) AS [MonthRelNo] ,
  If(WeekDay($1)<=WeekDay(Today()),1,0) AS [InWTD] ,
  (WeekStart(Today())-WeekStart($1))/7 AS [WeeksAgo] ,
  Week(Today())-Week($1) AS [WeekRelNo] ;

DERIVE FIELDS FROM FIELDS [DateKey], [AdmissionDate], [DischargeDate], [DayDate], [MonthStartDate], [MonthEndDate] USING [autoCalendar] ;
