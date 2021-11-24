# Covid-19 Project

## About

This is a portfolio project to practice on SQL and Tableau. In it, I used what I have learned on SQL so far, along with some new knowledge acquired while trying to solve some problems that occured. Some ways may be faster or easier than others, but I needed the practice and used all of them.
I got the dataset from [Our World In Data](https://ourworldindata.org/covid-deaths) and split it in 2 (cases/deaths and vaccinations), so that I could also practice JOINs. I was very curious to explore Covid data myself, so the whole project was incredibly interesting to me.

Here is the [Talbeau dashboard](https://public.tableau.com/views/CovidProject_16377520725710/Dashboard1?:language=en-US&:display_count=n&:origin=viz_share_link) I created, with some of the resulting tables here.

## Project

After splitting the dataset in Excel, I imported the data into SQL Server into 2 tables (CovidDeaths and CovidVaccinations), using the Import and Export Wizard.

First, I created a view containing only the variables I would be using most often. The "location" field contains not just country names, but also aggregate fields like "Asia" or "European Union" and "World" (I have no idea why). The "continent" fields of these particular records were left empty, so I tried to filter them out using ```IS NOT NULL```, but it did not work. It turned out that these "continent" fields were not actually blank but contained a space, so I used ```NOT IN(' ')```. The "total_cases" and "total_deaths" fields contain the total values up to each day, so the max values would be the actual total cases and deaths up to 17/11/2021. Their initial data type was bigint, but I needed to perform divisions which would result in decimal numbers, so I cast them as floats. 
```
USE covid_project
GO
CREATE VIEW covid_data AS
SELECT continent,
       location,
       population,
       MAX(CAST(total_cases AS float)) AS reported_cases,
       MAX(CAST(total_deaths AS float)) AS reported_deaths
FROM covid_project..CovidDeaths
WHERE continent NOT IN (' ')
GROUP BY continent,
         location,
         population
```
### Global Data
The first thing I wanted to see was some global aggregate numbers. In the beginning, The query I wrote at first kept getting the "is invalid in the select list because it is not contained in either an aggregate function or the GROUP BY clause" error. After searching for an appropriate solution, I solved my problem using a CTE. Since there were some zeros in "population" and "reported_cases", which I used as denominators, I had to use the ```NULLIF()``` function, to avoid any possible Jumanji level events.
```
WITH gl_data (population,
              reported_cases,
              reported_deaths)
AS (
    SELECT SUM(population) AS population,
    SUM(reported_cases) AS reported_cases,
    SUM(reported_deaths) AS reported_deaths
    FROM covid_project..covid_data
   )
SELECT *,
       ROUND(reported_cases / NULLIF(population, 0) * 100, 2) AS infected_percentage,
       ROUND(reported_deaths / NULLIF(reported_cases, 0) * 100, 2) AS death_percentage
FROM gl_data
```

Then, I wanted to visualize infection and death percentages over time, so I needed to create a relevant table. In order to avoid the same ```GROUP BY``` error as before, I tried creating and using a view this time. 
```
USE covid_project
GO
CREATE VIEW covid_date_data AS
SELECT continent,
       location,
       population,
       date,
       MAX(CAST(total_cases AS float)) AS reported_cases,
       MAX(CAST(total_deaths AS float)) AS reported_deaths
FROM covid_project..CovidDeaths
WHERE continent NOT IN (' ')
GROUP BY continent,
         location,
         population,
         date
SELECT location,
       population,
       reported_cases,
       reported_deaths,
       (reported_cases / NULLIF(population, 0)) * 100 AS infected_percentage,
       (reported_deaths / NULLIF(reported_cases, 0)) * 100 AS death_percentage
FROM covid_project..covid_date_data
```

### Continental Data
This time, I wanted to create separate graphs for infection and death percentages by continent. First, I created a view with the data I needed and then I wrote my query.
```
USE covid_project
GO
CREATE VIEW continent_data AS
SELECT continent,
       SUM(population) AS population,
       SUM(reported_cases) AS reported_cases,
       SUM(reported_deaths) AS reported_deaths
FROM covid_project..covid_data
GROUP BY continent

SELECT *,
       ROUND(reported_cases / NULLIF(population, 0) * 100, 2) AS infected_percentage,
       ROUND(reported_deaths / NULLIF(reported_cases, 0) * 100, 2) AS death_percentage
FROM covid_project..continent_data
```

### Country Data
I already had the data I needed ready for this, so I only needed to write a simple query (as opposed to the other, mindbogglingly complicated, queries). This time, I did not round up the results, because during the first several days of reporting the percentages were very small and the values would be rounded to zeros.
```
SELECT location,
	   population,
	   reported_cases,
	   reported_deaths,
	   (reported_cases / NULLIF(population, 0)) * 100 AS infected_percentage,
	   (reported_deaths / NULLIF(reported_cases, 0)) * 100 AS death_percentage
FROM covid_project..covid_data
```

### Income Data
The "location" field also contained a few income records. I decided it would be interesting to look at the matter from this angle, so I wrote the following query to create a table to visualize. The records were split into high income, upper-middle income, lower-middle income, and low income, which is why I used ```LIKE '%income%'```. Since I had filtered these records out when I created the "covid_data" view, I decided to create a temp table for this. 
```
IF OBJECT_ID('tempdb.dbo.#income_data', 'U') IS NOT NULL
  DROP TABLE #income_data;
CREATE TABLE #income_data
	(
	  income varchar(30),
	  population bigint,
	  reported_cases float,
	  reported_deaths float
	 )
INSERT INTO #income_data
SELECT location AS income,
       SUM(DISTINCT population) AS population,
       CAST(MAX(total_cases) AS float) AS reported_cases,
       CAST(MAX(total_deaths) AS float) AS reported_deaths
FROM covid_project..CovidDeaths
WHERE location LIKE '%income%'
GROUP BY location

SELECT *,
       ROUND((reported_cases / population) * 100, 2) AS infection_percentage,
       ROUND((reported_deaths / reported_cases) * 100, 2) AS death_percentage
FROM #income_data
```

### Vaccination Data
Finally, this one I only wanted to explore without visualizing it, and it also served as practice for joining tables
```
IF OBJECT_ID('tempdb.dbo.#vac_per', 'U') IS NOT NULL
  DROP TABLE #vac_per;
CREATE TABLE #vac_per
	(
	  continent varchar(30),
	  location varchar(30),
	  date datetime,
	  population bigint,
	  new_vaccinations float,
	  vaccinations_cont float
	 )
INSERT INTO #vac_per
SELECT cd.continent,
       cd.location,
       cd.date,
       cd.population,
       cv.new_vaccinations,
       SUM(cv.new_vaccinations) 
         OVER (PARTITION BY cd.location ORDER BY cd.location, cd.date) AS vaccinations_cont
FROM covid_project..CovidDeaths cd
  JOIN covid_project..CovidVaccinations cv
	ON cd.location = cv.location
	AND cd.date = cv.date
WHERE cd.continent NOT IN (' ')

SELECT *,
       ROUND(vaccinations_cont / NULLIF(population, 0) * 100, 3) AS vaccination_percentage
FROM #vac_per
ORDER BY location,
         date
```






