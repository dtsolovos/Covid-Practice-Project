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
![Global Stats](https://github.com/dtsolovos/Covid-Practice-Project/blob/main/Global%20Stats.png)
According to the data, up until 17/11/2021, 3.26% of the global population has been infected and 2.01% of the infected have died.


Then, I wanted to visualize infection percentages over time, so I needed to create a relevant table. In order to avoid the same ```GROUP BY``` error as before, I tried creating and using a view this time. 
```
USE covid_project
GO
CREATE VIEW covid_date_data AS
SELECT continent,
       location,
       population,
       date,
       MAX(CAST(total_cases AS float)) AS reported_cases
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
       (reported_cases / NULLIF(population, 0)) * 100 AS infected_percentage
FROM covid_project..covid_date_data
```
![Infections Over Time](https://github.com/dtsolovos/Covid-Practice-Project/blob/main/Infections%20Over%20Time.png)
The virus spread exploded in the winter of 2020 and has been constantly rising ever since. There are a few small drops and spikes in the infection rate, which generally seem to coincide with high tourist seasons.

### Continental Data
Next, I wanted to create graphs for infection and death counts by continent.
```
SELECT continent,
       SUM(population) AS population,
       SUM(reported_cases) AS reported_cases,
       SUM(reported_deaths) AS reported_deaths
FROM covid_project..covid_data
GROUP BY continent
```
![Continental Graphs](https://github.com/dtsolovos/Covid-Practice-Project/blob/main/ContGraph.png)
According to the data, Asia and Europe have the most recorded cases and deaths, which is not suriprising, given that they are the 2 most densely populated continents. While South America has nearly 20,000,000 less recorded Covid-19 cases than North America, it also has less than 30,000 fewer recorded deaths, with the death rate being the highest in the world at 3,04%. Surprisingly, despite having the 2nd largest population count (~1.4bn) and 3rd largest population density (33.66 per squared km), Africa has only 8,570,208 recorded Covid-19 cases, at just 0,62% of its population. The low median age (18 years) of sub-Saharan Africa, insufficient data collection, and lack of long-term care facilities (most elderly people live with their families) are some of the most prevalent theories pertaining to this. Oceania has the second lowest infection percentage (0,67% of the population), which is not unexpected, considering the continent's isolated location, incredibly low population density (just 3.12 per squared km) and fast goverment responses to the situation.

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






