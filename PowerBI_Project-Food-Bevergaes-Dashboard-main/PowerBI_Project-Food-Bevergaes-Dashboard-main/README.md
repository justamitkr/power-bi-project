# PowerBI_Project-Food-Bevergaes-Dashboard

## Data Model

![2023-07-09T12_27_20](https://github.com/Siddarameshwaruh/PowerBI_Project-Food-Bevergaes-Dashboard/assets/127327782/e0e79000-97f4-49de-b6c6-e4d92d43a4a5)

## Dashboard

![2023-07-08T20_08_34](https://github.com/Siddarameshwaruh/PowerBI_Project-Food-Bevergaes-Dashboard/assets/127327782/eabed5d7-bcb9-46f1-b90f-e761a805b0e9)


**1. Demographic Insights (examples)**

***a. Who prefers energy drink more? (male/female/non-binary?)***

	SELECT 
	    gender AS Gender,
	    COUNT(respondent_id) AS `Total Respondents`
	FROM
	    dim_repondents
	GROUP BY gender
	ORDER BY COUNT(respondent_id) DESC;

***b. Which age group prefers energy drinks more?***

	SELECT 
	    age, COUNT(respondent_id) AS total_respondents
	FROM
	    dim_repondents
	GROUP BY age
	ORDER BY total_respondents DESC;

***c. Which type of marketing reaches the most Youth (15-30)?***

	WITH cte1 AS
	(
	SELECT  r.age, 
				s.marketing_channels as `Marketing Channels`,
	            COUNT(s.respondent_id) as `Total Respondents`, 
	            DENSE_RANK() OVER(PARTITION BY r.age ORDER BY count(s.respondent_id) DESC) AS rank1
		FROM dim_repondents r JOIN fact_survey_responses s
	     USING(respondent_id) 
	     WHERE Age IN ("15-18", "19-30")
	      GROUP BY Age, `Marketing Channels`
	      ORDER BY count(s.respondent_id) DESC
	      )
	SELECT age AS "Age",  `Marketing Channels`, `Total Respondents` FROM cte1 
		WHERE `Total Respondents` IN 
			(SELECT max(`Total Respondents`) FROM cte1	
	         GROUP BY age
				) AND rank1 = 1


**2. Consumer Preferences:**

***a. What are the preferred ingredients of energy drinks among respondents?***

	SELECT DISTINCT
	    ingredients_expected AS `Expected Ingredients`,
	    COUNT(Respondent_ID) AS `Total Respondents`
	FROM
	    fact_survey_responses
	GROUP BY Ingredients_expected
	ORDER BY COUNT(Respondent_ID) DESC

***b. What packaging preferences do respondents have for energy drinks?***

	SELECT DISTINCT
	    packaging_preference, COUNT(Respondent_ID) total_respondents
	FROM
	    fact_survey_responses
	GROUP BY packaging_preference
	ORDER BY total_respondents DESC


**3. Competition Analysis:**

***a. Who are the current market leaders?***

	SELECT DISTINCT
	    current_brands, COUNT(respondent_id) AS total_respondents
	FROM
	    fact_survey_responses
	GROUP BY current_brands
	ORDER BY total_respondents DESC
	LIMIT 5;

***b. What are the primary reasons consumers prefer those brands over ours?***

	WITH cte1 AS (
	SELECT DISTINCT current_brands, count(respondent_id) 
		FROM fact_survey_responses s
	    GROUP BY current_brands
	    ORDER BY count(respondent_id) DESC
	    LIMIT 3)
	SELECT c.current_brands, s.reasons_for_choosing_brands, 
        count(s.reasons_for_choosing_brands) AS number_of_responses 
	FROM cte1 AS c JOIN fact_survey_responses s
	USING(current_brands)
	GROUP BY c.current_brands, s.reasons_for_choosing_brands
	HAVING s.reasons_for_choosing_brands <> "other"
	ORDER BY current_brands, s.reasons_for_choosing_brands DESC;


**4. Marketing Channels and Brand Awareness:**

***a. Which marketing channel can be used to reach more customers?***

	SELECT c.marketing_channels AS `Marketing Channel`
	FROM (
	    SELECT marketing_channels, COUNT(respondent_id) AS count
	    FROM fact_survey_responses
	    GROUP BY marketing_channels
	    ORDER BY count DESC
	) AS c
	JOIN (
	    SELECT MAX(count) AS max_count
	    FROM (
	        SELECT marketing_channels, COUNT(respondent_id) AS count
	        FROM fact_survey_responses
	        GROUP BY marketing_channels
	    ) AS subquery
	) AS c1 ON c.count = c1.max_count;

***b. How effective are different marketing strategies and channels in reaching our 
customers?***

	SELECT 
	    cte.marketing_channels,
	    ROUND(100 * cte.count / (SELECT 
	                    SUM(count)
	                FROM
	                    (SELECT 
	                        marketing_channels, COUNT(respondent_id) AS count
	                    FROM
	                        fact_survey_responses
	                    GROUP BY marketing_channels) AS subquery),
	            2) AS `Effectiveness %`
	FROM
	    (SELECT 
	        marketing_channels, COUNT(respondent_id) AS count
	    FROM
	        fact_survey_responses
	    GROUP BY marketing_channels) AS cte
	GROUP BY cte.marketing_channels;


**5. Brand Penetration:**

***a. What do people think about our brand? (overall rating)***

	WITH cte AS (
	SELECT 
	current_brands,
	Brand_perception,
	COUNT(Brand_perception) AS Total_respondents,
	ROUND(100* COUNT(Brand_perception)/SUM(COUNT(Brand_perception)) OVER(),2) AS Respondents_Percentage
	FROM
	fact_survey_responses
	WHERE
	current_brands='Codex'
	GROUP BY
	Current_brands,
	Brand_perception
	)
	SELECT
	current_brands,
	Brand_perception, 
	Total_Respondents,
	Respondents_Percentage
	FROM
	cte
	ORDER BY
	Total_Respondents DESC;

***b. Which cities do we need to focus more on?***

- Reasons for choosing brand

		SELECT 
		    Reasons_for_choosing_brands,
		    (COUNT(Respondent_ID)/SUM(COUNT(Respondent_ID)) OVER()) * 100 AS respondents_percentage
		FROM
		    fact_survey_responses
		WHERE
		    current_brands = 'Codex'
		GROUP BY Reasons_for_choosing_brands
		ORDER BY respondents_percentage DESC;

- Ingredients Expected: CodeX

		SELECT Ingredients_expected, (count(Respondent_ID)/SUM(COUNT(Respondent_ID)) OVER()) * 100  AS respondents_percentage
		FROM fact_survey_responses
		WHERE current_brands = "Codex"
		GROUP BY Ingredients_expected
		ORDER BY respondents_percentage DESC;

- Average taste rating: CodeX

		SELECT 
		    c.city,
		    (SELECT 
		            ROUND(AVG(Taste_experience), 1)
		        FROM
		            fact_survey_responses s
		                INNER JOIN
		            dim_repondents r ON s.Respondent_ID = r.Respondent_ID
		        WHERE
		            s.current_brands = 'CodeX'
		                AND r.city_ID = c.CIty_ID) AS avg_taste_experience
		FROM
		    dim_cities c


**6. Purchase Behavior:**

***a. Where do respondents prefer to purchase energy drinks?***

	SELECT 
	    purchase_location AS 'Purchase Location',
	    ROUND(100 * customer_count / (SELECT 
	                    SUM(customer_count)
	                FROM
	                    (SELECT 
	                        purchase_location, COUNT(respondent_id) AS customer_count
	                    FROM
	                        fact_survey_responses
	                    GROUP BY purchase_location) AS subquery),
	            2) AS `Respondents%`
	FROM
	    (SELECT 
	        purchase_location, COUNT(respondent_id) AS customer_count
	    FROM
	        fact_survey_responses
	    GROUP BY purchase_location) AS cte
	GROUP BY purchase_location;

***b. What are the typical consumption situations for energy drinks among 
respondents?***

	WITH cte1 as(
	SELECT DISTINCT Typical_consumption_situations, COUNT(Respondent_ID) AS total_respondents
	FROM fact_survey_responses
	GROUP BY Typical_consumption_situations
	ORDER BY COUNT(Respondent_ID) DESC
	)
	SELECT Typical_consumption_situations, ROUND(100*total_respondents/SUM(total_respondents)
	OVER(),2) AS '% respondents'
	FROM cte1;

***c. What factors influence respondents' purchase decisions, such as price range and 
limited edition packaging?***

- Purchase behaviour: Price range

		SELECT DISTINCT
		    Price_range, COUNT(Respondent_ID) AS total_respondents
		FROM
		    fact_survey_responses
		GROUP BY Price_range
		ORDER BY total_respondents DESC;

- Purchase behaviour: limited edition packaging 

		SELECT DISTINCT
		    Limited_edition_packaging,
		    COUNT(Respondent_ID) AS total_respondents
		FROM
		    fact_survey_responses
		GROUP BY Limited_edition_packaging
		ORDER BY total_respondents DESC;

- Purchase behaviour: health concerns

		SELECT DISTINCT
		    Health_concerns, COUNT(Respondent_ID) AS total_respondents
		FROM
		    fact_survey_responses
		GROUP BY Health_concerns
		ORDER BY total_respondents DESC

**7. Product Development**

***a. Which area of business should we focus more on our product development? 
(Branding/taste/availability)***

- Reasons for choosing brand: CodeX

		SELECT Reasons_for_choosing_brands, 
		(COUNT(Respondent_ID)/SUM(COUNT(Respondent_ID)) OVER()) * 100 AS respondents_percentage
		FROM fact_survey_responses
		WHERE current_brands = "Codex"
		GROUP BY Reasons_for_choosing_brands
		ORDER BY respondents_percentage DESC;

- Ingredients expected: CodeX

		SELECT Ingredients_expected, 
		(COUNT(Respondent_ID)/SUM(COUNT(Respondent_ID)) OVER()) * 100 AS '% respondents'
		FROM fact_survey_responses
		WHERE current_brands = "Codex"
		GROUP BY Ingredients_expected
		ORDER BY '% respondents' DESC;

- Average taste_rating: CodeX

		SELECT 
		    c.city,
		    (SELECT 
		            ROUND(AVG(taste_experience), 1)
		        FROM
		            fact_survey_responses s
		                INNER JOIN
		            dim_repondents r ON s.respondent_ID = r.Respondent_ID
		        WHERE
		            s.current_brands = 'CodeX'
		                AND r.city_ID = c.city_ID) AS avg_taste_experience
		FROM
		    dim_cities c



















   
