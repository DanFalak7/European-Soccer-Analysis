#  European Soccer SQL Analysis (Top 5 Leagues)

##  Project Overview
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

- **Database Setup**: Creation of the 'european_soccer_db' and the required tables
- **Table Definitions**: Creation of the tables and data import
- **Database Exploration & Cleaning**: Handlimng null values, data tipe corrections and ensuring data integrity
- **Views Creation**: The top 5 leagues extraction from the main table
- **Data Analysis & Findings**: Each SQL query answers a specific analytical question related to football performance and competitiveness.


## Database Setup

sql
DROP DATABASE IF EXISTS european_soccer; 
CREATE DATABASE european_Soccer;
USE european_soccer;
