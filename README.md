#  European Football SQL Analysis (Top 5 Leagues)

##  Project Overview
**Project Title:** European Football
**Level:** Advanced
**Database:** `european_soccer`

This project is a comprehensive **SQL-based exploratory data analysis** of European soccer matches, focusing on the **Top 5 European leagues**:

- English Premier League
- La Liga
- Bundesliga
- Serie A
- Ligue 1

Using match-level data across multiple seasons, the project explores **league competitiveness, team dominance, home advantage, scoring trends, and seasonal dynamics**.  
The goal is to demonstrate **advanced SQL skills** including joins, CTEs, window functions, aggregations, and analytical reasoning.

---

##  Objectives
- Analyze competitiveness across European leagues
- Identify historically dominant teams
- Measure home vs away performance
- Understand seasonal and stage-based competitiveness
- Detect goal-scoring trends over time
- Showcase real-world SQL problem-solving skills

---

##  Project Structure

- **Database Setup**: Created a database named `european_soccer` and creation tables for countries, leagues, matches and teams
- **Database Cleaning & Exploration**: Handling, data type, null values and ensuring data integrity
- **Virtual Table**: Extracted the top 5 leagues from the main table which includes EPL, Bundesliga, Ligue 1, , Seria A and La liga
- **Data Analysis**: Each SQL query answers a specific analytical question related to football performance and competitiveness.
- **Key Findings**: Identified differences in goal-scoring trends, home advantage, and competitive balance across Europe’s top leagues.
- **Insights**: Revealed how tactical styles, home advantage, and team dominance vary by league, supporting comparative football performance analysis.


### Database Setup

```sql
DROP DATABASE IF EXISTS european_soccer; 
CREATE DATABASE european_Soccer;

-- Create table "Countries"
CREATE TABLE countries (
		id INT PRIMARY KEY,
        name VARCHAR(50));

-- Create table "Leagues" 
CREATE TABLE leagues (
		id INT PRIMARY KEY,
        FOREIGN KEY (country_id) REFERENCES countries(country_id),
        name VARCHAR(50));

-- Create table "matches"
CREATE TABLE matches (
		id INT PRIMARY KEY,
        FOREIGN KEY (country_id) REFERENCES countries(country_id),
        season VARCHAR(10),
        stage INT,
        date CHAR(10),
        hometeam_id INT,
        awayteam_id INT,
        home_goal INT,
        away_goal INT);

-- Create table "Teams"
CREATE TABLE teams (
		id INT PRIMARY KEY,
        team_api_id INT,
        team_long_name VARCHAR(50),
        team_short_name VARCHAR(3));   
```

### Data Cleaning 
```sql
-- Convert date column to date data type
ALTER TABLE matches
MODIFY COLUMN date DATE;  


-- Check for Null Values
SELECT * FROM matches
WHERE id IS NULL OR country_id IS NULL OR season IS NULL 
	OR stage IS NULL OR date IS NULL OR hometeam_id IS NULL 
    OR awayteam_id IS NULL OR home_goal IS NULL OR away_goal IS NULL;
```

### Data Exploration
```sql
-- How many leagues we have
SELECT COUNT(DISTINCT name)
FROM leagues; 

-- How many Season we have
SELECT DISTINCT season
FROM matches;

-- How many matches per league
SELECT name AS league,
		COUNT(*) As total_matches
FROM matches
LEFT JOIN leagues 
USING(country_id)
GROUP BY name;
```

### Virtual Table
```sql
-- Create virtual Table for top 5 league matches
CREATE VIEW top_5_league AS (
SELECT *
FROM matches
WHERE country_id IN(1729, 4769, 7809, 10257, 21518));
```

### Data Analysis

#### 1. Question: How many matches were played in each league, how many total goals were scored, and what is the average number of goals per match?
**Returns:** One row per league showing total matches, total goals scored, and average goals per match, ordered from highest to lowest scoring leagues.
```sql
SELECT name AS league,
		COUNT(*) As total_matches,
		SUM(home_goal + away_goal) As total_goals,
        ROUND(AVG(home_goal + away_goal),2) AS avg_goals_per_match
FROM top_5_league
LEFT JOIN leagues 
USING(country_id)
GROUP BY name
ORDER BY avg_goals_per_match DESC;
```

#### 2. Question: Which European leagues are the most competitive based on match outcomes?
**Returns:** Win, draw, and loss distribution for each league, including percentages. Higher draw percentages indicate higher competitiveness.
```sql
SELECT
    l.name AS league,
    COUNT(*) AS total_matches,
    SUM(CASE WHEN m.home_goal > m.away_goal THEN 1 ELSE 0 END) AS home_wins,
    SUM(CASE WHEN m.away_goal > m.home_goal THEN 1 ELSE 0 END) AS away_wins,
    SUM(CASE WHEN m.home_goal = m.away_goal THEN 1 ELSE 0 END) AS draws,
    ROUND(SUM(CASE WHEN m.home_goal > m.away_goal THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS home_win_pct,
    ROUND(SUM(CASE WHEN m.away_goal > m.home_goal THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS away_win_pct,
    ROUND(SUM(CASE WHEN m.home_goal = m.away_goal THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS draw_pct
FROM top_5_league m
JOIN leagues l
    ON m.country_id = l.country_id
GROUP BY l.name
ORDER BY draw_pct DESC;
```

#### 3. Question: Which teams have been historically dominant across all seasons in each league?
**Returns:** Top 5 teams per league ranked by total points and goal difference, including goals scored and conceded.
```sql
-- Create CTE
WITH match_results AS (
	-- Home team matches and points
    SELECT l.name AS league,
			team_long_name AS team,
            CASE WHEN home_goal > away_goal THEN 3
				WHEN home_goal = away_goal THEN 1
                ELSE 0 END AS points,
			home_goal AS goals_for,
            away_goal AS goals_against
	FROM top_5_league 
    LEFT JOIN leagues l
    USING(country_id)
    LEFT JOIN teams t
    ON hometeam_id = t.team_api_id
    
    UNION ALL -- join both the home teams and away teams vertically
    -- Away team matches and points
    SELECT l.name AS league,
			team_long_name AS team,
            CASE WHEN away_goal > home_goal THEN 3
				WHEN away_goal = home_goal THEN 1
                ELSE 0 END AS points,
			away_goal AS goals_for,
            home_goal AS goals_against
	FROM top_5_league
    LEFT JOIN leagues l
    USING(country_id)
    LEFT JOIN teams
    ON awayteam_id = team_api_id),

-- Sum up the points and goals for each team
teams_total AS (
	SELECT league,
			team,
            SUM(points) AS total_points,
            SUM(goals_for) AS goals_scored,
            SUM(goals_against) AS goals_conceded,
            SUM(goals_for) - SUM(goals_against) AS goal_difference
	FROM match_results
    GROUP BY league, team),

-- Rank teams by points and goal difference
teams_rank AS (
	SELECT league,
			team,
            total_points,
            goals_scored,
            goals_conceded,
            goal_difference,
            RANK() OVER(
						PARTITION BY league
						ORDER BY total_points DESC, 
								goal_difference DESC) AS league_rank
	FROM teams_total) 
    -- CTE End
    
SELECT league,
		team,
        total_points,
        goal_difference,
        goals_scored,
        goals_conceded,
        league_rank
FROM teams_rank
WHERE league_rank <= 5
ORDER BY league, league_rank;
```


#### 4. Question: Which teams are the most dominant across all leagues based on win rate?
**Returns:** Top 10 teams with the highest win percentage, including total matches, wins, draws, and losses.
```sql
with total_matches AS (
-- Home team
	SELECT l.name AS league,
			team_long_name AS team,
            CASE WHEN home_goal > away_goal THEN 1 ELSE 0 END AS win,
            CASE WHEN home_goal = away_goal THEN 1 ELSE 0 END AS draw,
            CASE WHEN home_goal < away_goal THEN 1 ELSE 0 END AS loss
	FROM top_5_league
    LEFT JOIN leagues l
    USING(country_id)
    LEFT JOIN teams
    ON hometeam_id = team_api_id
    UNION ALL
    -- Away team
    SELECT l.name AS league,
			team_long_name AS team,
            CASE WHEN away_goal > home_goal THEN 1 ELSE 0 END AS win,
            CASE WHEN away_goal = home_goal THEN 1 ELSE 0 END AS draw,
            CASE WHEN away_goal < home_goal THEN 1 ELSE 0 END AS loss
	FROM top_5_league
    LEFT JOIN leagues l
    USING(country_id)
    LEFT JOIN teams
    ON awayteam_id = team_api_id
)
SELECT league,
		team,
        COUNT(*) AS total_matches,
        SUM(win) AS total_wins,
        SUM(draw) AS total_draws,
        SUM(loss) AS total_loss,
        ROUND((SUM(win)/COUNT(*)) * 100, 2) AS win_rate_pct
FROM total_matches
GROUP BY league, team
ORDER BY win_rate_pct DESC
LIMIT 10;
```

#### 5. Question: Which teams are the least efficient at winning matches?
**Returns:** Bottom 10 teams with the lowest win rates across all leagues.
```sql
with total_matches AS (
-- Home team
	SELECT l.name AS league,
			team_long_name AS team,
            CASE WHEN home_goal > away_goal THEN 1 ELSE 0 END AS win,
            CASE WHEN home_goal = away_goal THEN 1 ELSE 0 END AS draw,
            CASE WHEN home_goal < away_goal THEN 1 ELSE 0 END AS loss
	FROM top_5_league
    LEFT JOIN leagues l
    USING(country_id)
    LEFT JOIN teams
    ON hometeam_id = team_api_id
    
    UNION ALL
    -- Away team
    SELECT l.name AS league,
			team_long_name AS team,
            CASE WHEN away_goal > home_goal THEN 1 ELSE 0 END AS win,
            CASE WHEN away_goal = home_goal THEN 1 ELSE 0 END AS draw,
            CASE WHEN away_goal < home_goal THEN 1 ELSE 0 END AS loss
	FROM top_5_league
    LEFT JOIN leagues l
    USING(country_id)
    LEFT JOIN teams
    ON awayteam_id = team_api_id
)
SELECT league,
		team,
        COUNT(*) AS total_matches,
        SUM(win) AS total_wins,
        SUM(draw) AS total_draws,
        SUM(loss) AS total_loss,
        ROUND((SUM(win)/COUNT(*)) * 100, 2) AS win_rate_pct
FROM total_matches
GROUP BY league, team
ORDER BY win_rate_pct
LIMIT 10;
```

#### 6. Question: How strong is home advantage across different leagues?
**Returns:** Home win percentage for each league, showing how often home teams win their matches.
```sql
SELECT l.name AS league,
		COUNT(*) as total_matches,
        SUM(CASE WHEN home_goal > away_goal THEN 1 ELSE 0 END) AS home_win,
        ROUND((SUM(CASE WHEN home_goal > away_goal THEN 1 ELSE 0 END)/COUNT(*)) * 100, 2) AS home_win_pct
FROM top_5_league
LEFT JOIN leagues l
USING(country_id)
GROUP BY league
ORDER BY home_win_pct DESC;
```

#### 7. Question: Which teams benefit the most from playing at home?
**Returns:** Top 5 teams with the highest home win percentages across all leagues.
```sql
SELECT l.name AS league,
		team_long_name As team,
        COUNT(*) AS home_matches,
        SUM(CASE WHEN home_goal > away_goal THEN 1 ELSE 0 END) AS home_wins,
        SUM(CASE WHEN home_goal = away_goal THEN 1 ELSE 0 END) AS home_draws,
        SUM(CASE WHEN home_goal < away_goal THEN 1 ELSE 0 END) AS home_losses,
        ROUND((SUM(CASE WHEN home_goal > away_goal THEN 1 ELSE 0 END)/COUNT(*)) * 100.0, 2) AS home_win_pct
FROM top_5_league
LEFT JOIN leagues l
USING(country_id)
LEFT JOIN teams
ON hometeam_id = team_api_id
GROUP BY team, league
ORDER BY home_win_pct DESC
LIMIT 5;
```

#### 8. Question: Which seasons were unusually high or low scoring compared to league averages?
**Returns:** Seasons with the biggest positive difference between season average goals and historical league average goals.
```sql
WITH league_historical_avg AS (
    SELECT
        l.name AS league_name,
        AVG(m.home_goal + m.away_goal) AS league_avg_goals_per_match
    FROM top_5_league m
    JOIN leagues l
        ON m.country_id = l.country_id
    GROUP BY l.name
),
season_goals AS (
    SELECT
        l.name AS league_name,
        m.season,
        COUNT(*) AS match_count,
        SUM(m.home_goal + m.away_goal) AS total_goals,
        SUM(m.home_goal + m.away_goal) * 1.0 / COUNT(*) AS season_avg_goals_per_match
    FROM top_5_league m
    JOIN leagues l
        ON m.country_id = l.country_id
    GROUP BY l.name, m.season
)
SELECT
    s.league_name,
    s.season,
    s.match_count,
    s.total_goals,
    ROUND(s.season_avg_goals_per_match - h.league_avg_goals_per_match,
		2
    ) AS goal_diff_from_league_avg
FROM season_goals s
JOIN league_historical_avg h
    ON s.league_name = h.league_name
ORDER BY goal_diff_from_league_avg DESC
LIMIT 5;
```

#### 9. Question: Which match stages (weeks) are the most competitive in each league?
**Returns:** Top 2 stages per league with the highest draw percentages.
```sql
WITH stage_competitiveness AS (
    SELECT
        l.name AS league,
        m.stage,
        COUNT(*) AS total_matches,
        SUM(CASE WHEN m.home_goal > m.away_goal THEN 1 ELSE 0 END) AS home_wins,
        SUM(CASE WHEN m.away_goal > m.home_goal THEN 1 ELSE 0 END) AS away_wins,
        SUM(CASE WHEN m.home_goal = m.away_goal THEN 1 ELSE 0 END) AS draws,
        ROUND(
            SUM(CASE WHEN m.home_goal = m.away_goal THEN 1 ELSE 0 END) * 100.0 / COUNT(*),
            2
        ) AS draw_pct
    FROM top_5_league m
    JOIN leagues l
        ON m.country_id = l.country_id
    GROUP BY l.name, m.stage
),
ranked_stages AS (
    SELECT
        league,
        stage,
        total_matches,
        home_wins,
        away_wins,
        draws,
        draw_pct,
        RANK() OVER (
            PARTITION BY league
            ORDER BY draw_pct DESC
        ) AS competitiveness_rank
    FROM stage_competitiveness
)
SELECT *
FROM ranked_stages
WHERE competitiveness_rank <= 2
ORDER BY league, competitiveness_rank;
```

#### 10. Question:Which part of the season is generally more competitive?
**Returns:** Counts of competitive stages grouped into early, mid, and late season.
```sql
WITH stage_competitiveness AS (
    SELECT
        l.name AS league,
        m.stage,
        COUNT(*) AS total_matches,
        SUM(CASE WHEN m.home_goal > m.away_goal THEN 1 ELSE 0 END) AS home_wins,
        SUM(CASE WHEN m.away_goal > m.home_goal THEN 1 ELSE 0 END) AS away_wins,
        SUM(CASE WHEN m.home_goal = m.away_goal THEN 1 ELSE 0 END) AS draws,
        ROUND(
            SUM(CASE WHEN m.home_goal = m.away_goal THEN 1 ELSE 0 END) * 100.0 / COUNT(*),
            2
        ) AS draw_pct
    FROM top_5_league m
    JOIN leagues l
        ON m.country_id = l.country_id
    GROUP BY l.name, m.stage
),
ranked_stages AS (
    SELECT
        league,
        stage,
        total_matches,
        home_wins,
        away_wins,
        draws,
        draw_pct,
        RANK() OVER (
            PARTITION BY league
            ORDER BY draw_pct DESC
        ) AS competitiveness_rank
    FROM stage_competitiveness
),
top_2 AS (
SELECT *
FROM ranked_stages
WHERE competitiveness_rank <= 2
ORDER BY league, competitiveness_rank
)
SELECT CASE WHEN stage <= 10 THEN 'Early Season'
			WHEN stage BETWEEN 11 AND 25 THEN 'Mid Season'
			ELSE 'Late Season' END AS stages,
            COUNT(*)
FROM top_2
GROUP BY stages;
```

#### 11. Question: Which month has historically produced the highest number of goals in each league?
**Returns:** For each league, the season-month combination with the highest total goals scored.
```sql
WITH monthly_goals AS (
	SELECT l.name AS league,
			season,
			EXTRACT(MONTH FROM date) AS month,
			SUM(home_goal + away_goal) AS total_goals
	FROM top_5_league
    LEFT JOIN leagues l
    USING(country_id)
    GROUP BY league, season, EXTRACT(MONTH FROM date)	),
ranked_months AS (
	SELECT league,
			season, month,
			total_goals,
            RANK() OVER(
						PARTITION BY league 
                        ORDER BY total_goals DESC) AS ranks
	FROM monthly_goals)
SELECT league, season, month, 
        total_goals
FROM ranked_months
WHERE ranks = 1
```

## Key Findings:

1.	**League Scoring Trends:** Bundesliga highest scoring (2.93 goals/match), Ligue 1 lowest (2.50).
2.	**League Competitiveness:** Serie A & Ligue 1 most competitive; La Liga less competitive.
3.	**Dominant Teams:** Manchester City, PSG, Bayern Munich, Juventus, Barcelona consistently lead.
4.	**Top Win Rates:** Bayern Munich 77.94%, Barcelona 76.97%, Juventus 72.48%.
5.	**Lowest Win Rates:** Cardoba CF, Racing Santander, Cesena <17% win rate.
6.	**Home Advantage:** Strong across all leagues; top teams benefit significantly.
7.	**High-Scoring Seasons:** Bundesliga 2013/2014 and La Liga 2012/2013 notable.
8.	**Competitive Match Stages:** Early and mid-season stages more unpredictable.
9.	**Seasonal Competitiveness:** Early season slightly more volatile.
10.	**Goal Patterns by Month:** Peak months vary per league (Dec, Mar, May).

## Insights

- Elite clubs dominate consistently across leagues.
- Home advantage remains critical for success.
- Competitiveness and scoring trends vary by league, season, and match stage.
- Useful for analysts, scouts, or anyone studying European soccer trends.

##  Skills Demonstrated

#### SQL Skills
-	Data Cleaning & Transformation: ALTER TABLE for type conversion, null checks, and ensuring data integrity.
-	Joins: INNER JOIN, LEFT JOIN, USING for combining multiple tables (matches, leagues, teams, countries).
-	Aggregations: COUNT(), SUM(), AVG(), for calculating matches, goals, and win/draw/loss statistics.
-	Conditional Logic: CASE WHEN statements to compute points, wins, draws, and losses.
-	Window Functions: RANK() OVER(PARTITION BY ...) to rank teams within leagues and stages.
-	Common Table Expressions (CTEs): Organizing complex queries with WITH for clarity and modularity.
-	Views: CREATE VIEW to create reusable virtual tables for the Top 5 leagues.
  
#### Data Analysis Skills
-	Exploratory Data Analysis (EDA): Identifying trends, distributions, and patterns across leagues and seasons.
-	Performance Analysis: Evaluating team dominance, win rates, home advantage, and competitiveness.
-	Time Series & Seasonal Analysis: Analyzing scoring trends across months, stages, and seasons.
-	Comparative Analysis: Comparing leagues, teams, and seasonal trends to draw meaningful insights.

#### Analytical Thinking
-	Breaking down a large dataset into digestible parts
-	Translating football domain questions into SQL queries
-	Interpreting results to generate actionable insights
### 
