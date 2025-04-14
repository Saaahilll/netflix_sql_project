## **Netflix Shows Analysis**
### Overview
***This repository contains SQL queries and analysis performed on the Shows table, which represents Netflix content. The dataset includes information about movies and TV shows, their directors, cast, countries of production, release years, genres, durations, and descriptions. The analysis focuses on extracting insights such as top actors, genres, and content trends.***
### Objectives
1. Analyze the distribution of content types (movies vs TV shows).
2. Identify the most common ratings for movies and TV shows.
3. List and analyze content based on release years, countries, and durations.
4. Explore and categorize content based on specific criteria and keywords.

### Dataset
The data for this project is sourced from the Kaggle dataset:

Dataset Link: Movies Dataset

### Schemas
```sql
CREATE TABLE Shows (
    show_id VARCHAR(10) PRIMARY KEY,
    type VARCHAR(50),
    title VARCHAR(255),
    director VARCHAR(255),
    "cast" VARCHAR(1000),
    country VARCHAR(150),
    date_added VARCHAR(50),
    release_year INT,
    rating VARCHAR(50),
    duration VARCHAR(50),
    listed_in VARCHAR(255),
    description VARCHAR(255)
);
```

### Business Problems and Solutions:

--** Count the Number of Movies vs TV Shows.**
```sql
SELECT 
    type,
    COUNT(*)  as total_content
FROM Shows
GROUP BY 1;
```

 --**Find the Most Common Rating for Movies and TV Shows.**
   
```sql
SELECT rating, COUNT(rating) AS count
FROM Shows
WHERE type IN ('Movie', 'TV Show')
GROUP BY rating
ORDER BY count DESC
LIMIT 1;                            -- gives output as which rating is given the most number of times and its count
```


--:ALTERNATE METHOD

```sql
WITH RankedRatings AS (
    SELECT type, rating, COUNT(rating) AS count,
           RANK() OVER (PARTITION BY type ORDER BY COUNT(rating) DESC) AS rank
    FROM Shows
    WHERE type IN ('Movie', 'TV Show')
    GROUP BY type, rating
)
SELECT type, rating, count
FROM RankedRatings
WHERE rank = 1;                     -- gives most common rating of each type along with its count  
```

--**List All Movies Released in the year 2020 and 2010.**

```sql
SELECT release_year, COUNT(*) AS movie_count
FROM Shows
WHERE type = 'Movie' AND release_year IN (2010, 2020)
GROUP BY release_year
ORDER BY release_year;                    -- gives the count of movies released during the following years
```

```sql
SELECT
    string_agg(CASE WHEN release_year = 2010 THEN title ELSE NULL END, ', ') AS movies_2010,
    string_agg(CASE WHEN release_year = 2020 THEN title ELSE NULL END, ', ') AS movies_2020
FROM
    Shows
WHERE
    type = 'Movie' AND release_year IN (2010, 2020);

                                          -- gives the list of all movies released in the given years
```


--**Find the Top 5 Countries with the Most Content on Netflix**

```sql
SELECT country, COUNT(*) AS content_count
FROM Shows
WHERE country IS NOT NULL
GROUP BY country
ORDER BY content_count DESC
LIMIT 5;
```
-- Or it can be done by:

```sql
SELECT * 
FROM
(
    SELECT 
        UNNEST(STRING_TO_ARRAY(country, ',')) AS country,
        COUNT(*) AS total_content
    FROM Shows
    GROUP BY 1
) AS t1
WHERE country IS NOT NULL
ORDER BY total_content DESC
LIMIT 5;
```


-- **IMPORTANT NOTE** 
-- If the country column always contains only one country, both queries could produce similar results.
-- However, if the country column contains two different countries separated by commas in the same cell multiple times, first query treat it as a single entry and not as separate           contributions from both countries.
-- While the second query tries to correct this by splitting the two counties into two separate entries, thus giving both credit for that content.	



-- **Identify the Longest Movie**

```sql
SELECT title, duration
FROM Shows
WHERE type = 'Movie'
ORDER BY 
    CASE
        WHEN duration LIKE '% min' THEN CAST(REPLACE(duration, ' min', '') AS INTEGER)
        ELSE NULL  -- or some other default value if needed
    END DESC
LIMIT 1;
```

--It can also be done by:

```sql
SELECT 
    *
FROM Shows
WHERE type = 'Movie'
ORDER BY SPLIT_PART(duration, ' ', 1)::INT DESC
LIMIT 1;
```


--**Find Content Added in the Last 5 Years**
```sql
SELECT *
FROM Shows
WHERE CAST(date_added AS DATE) >= CURRENT_DATE - INTERVAL '5 year'
```


--**Find All Movies/TV Shows by Director 'Rajiv Chilaka'**
```sql
SELECT title, type
FROM Shows
WHERE director = 'Rajiv Chilaka';
```


--**List All TV Shows with More Than 5 Seasons**
```sql
SELECT title, duration
FROM Shows
WHERE type = 'TV Show' AND 
      duration LIKE '% Seasons' AND 
      CAST(REPLACE(duration, ' Seasons', '') AS INTEGER) > 5;
```

--OR

```sql
SELECT *
FROM netflix
WHERE type = 'TV Show'
  AND SPLIT_PART(duration, ' ', 1)::INT > 5;
```

--Both approaches achieve the same result, but the second query is more concise and avoids explicitly removing text like "Seasons". It directly extracts the numeric part using SPLIT_PART.
-- Also, first query selects only specific columns (title, duration) to focus on relevant information, while second query selects all columns (SELECT *) from the Shows table. This  returns more data (all columns), which may be useful if additional details about TV shows are needed. 



 --**Count the Number of Content Items in Each Genre**
```sql
SELECT listed_in AS genre, COUNT(*) AS genre_count
FROM Shows
WHERE listed_in IS NOT NULL
GROUP BY listed_in
ORDER BY genre_count DESC;
```

-- Can also be done by:
```sql
SELECT 
    UNNEST(STRING_TO_ARRAY(listed_in, ',')) AS genre,
    COUNT(*) AS total_content
FROM Shows
GROUP BY 1;
```


--**Find each year and the average numbers of content release in India on netflix.**

```sql
SELECT
    EXTRACT(YEAR FROM CAST(date_added AS TIMESTAMP)) AS year,
    AVG(CASE WHEN country LIKE '%India%' THEN 1 ELSE 0 END) AS avg_content_india
FROM
    Shows
WHERE country IS NOT NULL AND date_added IS NOT NULL
GROUP BY
    year
ORDER BY
    year;                             --Aims to calculate the average presence of "India" in the country field for each year based on the date_added column. This provides a proportion                                           of content that is associated with India for each date_added year.
```

-- Can also be done by:

```sql
SELECT 
    country,
    release_year,
    COUNT(show_id) AS total_release,
    ROUND(
        COUNT(show_id)::numeric /
        (SELECT COUNT(show_id) FROM Shows WHERE country = 'India')::numeric * 100, 2
    ) AS avg_release
FROM Shows
WHERE country = 'India'
GROUP BY country, release_year
ORDER BY avg_release desc;            --Aims to calculate the percentage of content released in India in each release year, relative to the total number of shows with "India" in the 
                                        country column across all years. It provides a year-by-year breakdown of India's contribution.

```


--**List All Movies that are Documentaries**

```sql
SELECT title
FROM Shows
WHERE type = 'Movie' AND listed_in = 'Documentaries';
```

--:If the documentaries are listed with other genres, we can use LIKE:

```sql
SELECT title
FROM Shows
WHERE type = 'Movie' AND listed_in LIKE '%Documentaries%';
```


--**Find All Content Without a Director**

```sql
SELECT title, type
FROM Shows
WHERE director IS NULL;
```


--**Find How Many Movies Actor 'Salman Khan' Appeared in the Last 10 Years**
```sql
SELECT COUNT(*) AS movie_count
FROM Shows
WHERE type = 'Movie'
  AND release_year >= EXTRACT(YEAR FROM CURRENT_DATE) - 10
  AND "cast" LIKE '%Salman Khan%';
```



-- **Find the Top 10 Actors Who Have Appeared in the Highest Number of Movies Produced in India**

```sql
SELECT
    actor,
    COUNT(*) AS movie_count
FROM (
    SELECT
        UNNEST(STRING_TO_ARRAY("cast", ', ')) AS actor
    FROM
        Shows
    WHERE
        type = 'Movie' AND country LIKE '%India%'
) AS t1
GROUP BY
    actor
ORDER BY
    movie_count DESC
LIMIT 10;
```


--**Categorize Content Based on the Presence of 'Kill' and 'Violence' Keywords**

```sql
SELECT
    title,
    CASE
        WHEN description LIKE '%Kill%' THEN 1
        WHEN description LIKE '%Violence%' THEN 1
        ELSE 0
    END AS violence_flag
FROM
    Shows;
```

-- Alternate way is:
```sql
SELECT 
    category,
    COUNT(*) AS content_count
FROM (
    SELECT 
        CASE 
            WHEN description ILIKE '%kill%' OR description ILIKE '%violence%' THEN 'Bad'
            ELSE 'Good'
        END AS category
    FROM Shows
) AS categorized_content
GROUP BY category;
```




### **Insights Derived**

**Content Trends:** Identify years with high content releases and average releases by country.

**Actor Analytics:** Determine top actors based on appearances in Indian movies.

**Genre Popularity:** Understand which genres dominate Netflix's catalog.

**Keyword-Based Categorization:** Flag violent or thriller-related content using keywords.



































