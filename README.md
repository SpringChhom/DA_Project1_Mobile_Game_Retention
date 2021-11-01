# Data Analytics Bootcamp - Juno College - Project 1 - 30-Day Player Retention in a Mobile Game App

## Welcome to my first Data Analysis Project on Github!

For this project, two teammates and I were tasked with using SQL and Google Sheets to analyze player retention for a mobile game in which players could have matches against one another and purchase items in the game store. Upon the one year anniversary of this game, the mobile game company asked us to report on the rolling 30-day retention of players as well as an additional metric related to retention. The company provided us with a dataset consisting of four tables which included data on the players based on their player ID: when they joined, their age and location, what they purchased and when, and what matches they played and when.

For the purpose of this report, the word “player(s)” is referring to the unique player IDs under the assumption that real life players did not share their player IDs with others.

## Rolling 30-Day Retention

Rolling 30-day retention refers to whether a player has engaged with the game 30 days after having joined, and if they have, they are considered retained. For this company, they were more specifically interested in whether the players had participated in a one on one match 30 days after having joined. To answer this question, we calculated the rolling 30 day retention using several nested SQL queries which I will now break down into steps.

Our first step, which is the innermost SQL query in the nested queries, was to divide the players into two groups, those who had played a match 30 days after joining (retained) and those who hadn’t (not retained.) We started this by using the JOIN clause to link the matchs_info and the player_info tables.  We also used the MAX function to find the last match played by each player, then calculating whether that last match was more than or equal to 30 days after the player’s join day. We use the IF function to label retained players with the integer 1 and non-retained players with a 0 and we named that column retention_status. The JOIN clause chosen was LEFT JOIN in order to include players who had joined but not played in any matches.

```
SELECT
    joined,
    pl.player_id,
IF ((MAX(day)>=joined+30),1,0) AS retention_status
FROM
    `my-project-46686-1st-try.game_company_data.player_info` AS pl
LEFT JOIN
    `my-project-46686-1st-try.game_company_data.matches_info` AS m
ON
    pl.player_id = m.player_id
GROUP BY
    joined,
    pl.player_id
```
![alt text](https://github.com/SpringChhom/Data_Analytics_Project_1/blob/main/images_and_graphs/Img1-retention_status_table.png)

The next step was to calculate the percentage of players retained for 30 days out of the total players who had joined for each day of the first year of the game. We did this using the COUNT function to count the total players (as players_joined) and the COUNTIF function to count the number of players who were retained 30 days or more (as players_retained) and then dividing players_retained by total_players to find the fraction of players retained (as pct_retention) for each day of the year. We did not round the results or multiply by 100 to get the percent, instead leaving that to be done later in Google Sheets where we would do the visualization.

```
SELECT 
    joined,
    COUNT (player_id) AS players_joined,
    COUNTIF (retention_status = 1) AS players_retained,
    (COUNTIF (retention_status = 1)) / (COUNT (DISTINCT (player_id))) AS pct_retention
FROM (
query above
)
GROUP BY 
    joined
```
![alt text](https://github.com/SpringChhom/Data_Analytics_Project_1/blob/main/images_and_graphs/Img2-retention_pct_table-.png)

In the last two steps of this query (the two outermost queries of the whole), we calculated the 30-day retention growth rate for players throughout the year. We used the LAG function with OVER and ORDER BY to create an additional column called prev_pct_retention for the fractional retention of each previous day. We then subtracted the previous day’s fractional retention from that of each day of the year and divided the results by the previous day’s fractional retention to get the 30-day retention growth rate. Once again, we left the rounding and the multiplying by 100 to get the percentage to be done in Google Sheets.

```
SELECT 
    joined,
    pct_retention,
    prev_pct_retention,
    ROUND(SAFE_DIVIDE((pct_retention - prev_pct_retention), prev_pct_retention),4) AS retention_growth_rate
FROM (
    SELECT
        joined,
        pct_retention,
        LAG(pct_retention, 1) OVER (ORDER BY joined) AS prev_pct_retention
    FROM (queries above)
```
![alt text](https://github.com/SpringChhom/Data_Analytics_Project_1/blob/main/images_and_graphs/img3b-30-day-retention-growthrate.png)

We then imported the results into Google Sheets in order to visualize the 30-day retention by creating a chart. The percent retention and the retention growth rate were both plotted by day across the year and the horizontal trend lines for these scatterplots indicated that there was virtually no change throughout the year. We did not plot the last 30 days of the year since we did not have match information beyond 29 or fewer days for the players who joined during that time.
![alt text](https://github.com/SpringChhom/Data_Analytics_Project_1/blob/main/images_and_graphs/img4-30-day-rolling-retention-graph.png)
We also calculated the averages for our two retention metrics and found that the average for the retention growth rate was virtually 0% (0.24%), confirming that there was very little change from day to day throughout the year. The average for the daily percent retention was 73%, which by industry standards was quite high.  After researching the 30-day retention for mobile gaming apps in general, we found that it averaged anywhere from 5 to 27% with those numbers being acceptable and expected.  So with a 73% 30-day rolling retention, this company’s game seemed to be doing well. 


## The Effect of Win Streaks and Losing Streaks on 30-Day Player Retention

For the second part of the project, we decided to look at win streaks and losing streaks and how they affected 30-day retention. To accomplish this, we used a combination of WITH and nested queries in SQL.  I will only go through the win streak query since the losing streak query was the  same sequence but with the opposite win/loss terminology.

The first step was to identify win streaks by using the CASE statement with the LAG function in order to label each game played as a 1 or a 0 where a 1 indicated that the game was the beginning of a streak.  The games were also linked to the player IDs using the JOIN clause.

```
--identify win streaks by player_id
WITH new_win_streaks AS 
    (SELECT
        player_id,
        outcome,
        day,
    CASE 
        WHEN outcome = "win" AND
        LAG (outcome) OVER (PARTITION BY player_id order by day) = "loss"
        THEN 1
        WHEN outcome = "win" AND
        LAG (outcome) OVER (PARTITION BY player_id order by day) IS NULL
        THEN 1
        ELSE 0
END AS new_win_streak
FROM (
    SELECT 
        m.player_id,
        m.outcome,
        m.day
FROM 
     `my-project-46686-1st-try.game_company_data.matches_info` AS m
JOIN 
    `my-project-46686-1st-try.game_company_data.retention_status` AS  r
ON m.player_id=r.player_id
WHERE 
    m.day <= r.joined+30)),
```
In the next two steps, a unique number was assigned to each streak and the lengths of the streaks were calculated. A unique number was assigned to each streak, partitioning the streaks by player and day, so that we would be able to count the games per streak in the second step here. 

```
--assign unique number to each new streak
streak_nums AS
(SELECT 
    day,
    player_id,
    SUM(new_win_streak) OVER (PARTITION BY player_id ORDER BY day) AS streak_num
FROM
    new_win_streaks 
WHERE
   outcome = "win"),
--length of streak / how many wins
streak_length AS 
(SELECT 
    COUNT (*) AS counter,
    player_id,
    streak_num
FROM 
    streak_nums
GROUP BY
    player_id,
    streak_num)
```
In the last step, we used the MAX function to select the longest streak per player and we used the JOIN clause to link this information to the retention status of each player which was in a table created in the original 30-day retention status query. 

```
--longest streak per player plus link to retention
SELECT
    s.player_id,
    MAX (counter) AS longest_streak,
    r.retention_status
FROM
    streak_length AS s
JOIN
   `my-project-46686-1st-try.game_company_data.matches_info` AS m
ON  m.player_id=s.player_id
JOIN
    `my-project-46686-1st-try.game_company_data.retention_status` AS r
ON m.player_id=r.player_id
WHERE 
    m.day <= r.joined+30
GROUP BY
    s.player_id,
    r.retention_status
```
![alt text](https://github.com/SpringChhom/Data_Analytics_Project_1/blob/main/images_and_graphs/img3-streak_retention_table.png)

Once again, we imported our findings into Google Sheets for the visualization process. Here we’ll look at the relationship between losing streaks and 30-day retention first. 

In the graph below, we can see that as losing streaks got longer, the number of players retained increased until the 3 day losing streak point, where the number retained then rapidly declined. Those players not retained followed a similar pattern but this time, they started to decline after just a two loss streak.  Overall, for each length of losing streak, more players were retained than not retained, as seen previously in our average retention rate of 73%.
![alt text](https://github.com/SpringChhom/Data_Analytics_Project_1/blob/main/images_and_graphs/img5-losing_streak_line_graph.png)
For the next graph, we only looked at winning streaks up to a length of 10 games since the number of players for streaks higher than that was so low that the margin of error and variability would lead to meaningless results.

Here, we can see that only with a one game longest losing streak is the percent retention higher for those not retained than for those retained. It seems as if once players stuck it out into a 2 or more game losing streak, they were more likely to be retained the longer the losing streak.  
![alt text](https://github.com/SpringChhom/Data_Analytics_Project_1/blob/main/images_and_graphs/img6-losing_streak_bar_graph.png)
Next, we’ll look at the relationship between win streaks and 30-day retention. 

In this graph, we can see that as win streaks got longer, the number of players retained increased until the win streak was 3 days long, and then once again, the number retained rapidly declined. Much like the losing streak pattern, those players not retained started to decline after just a two win streak.  And again, 0verall, for each length of win streak, more players were retained than not retained, following our average retention rate of 73%.
![alt text](https://github.com/SpringChhom/Data_Analytics_Project_1/blob/main/images_and_graphs/img7-win_streak_line_graph.png)
Again, in the following graph, we only looked at win streaks up to a length of 10 games due to the low sample size of higher streak lengths.

Here, we can see that once again, only with a one game longest win streak is the percent retention higher for those not retained than for those retained. Then, as the players’ longest win streaks increased, they continued to be more likely to be retained.
![alt text](https://github.com/SpringChhom/Data_Analytics_Project_1/blob/main/images_and_graphs/img8-win_streak_bar_graph.png)

## Conclusions and Next Steps

In conclusion, we were able to report to the mobile game company that their game’s 30-day retention had remained steady throughout the entire year and that their 73% retention rate indicated that their game was doing quite well by industry standards. In addition, we were able to inform them that win streaks and losing streaks did not seem to affect 30-day retention and that in both circumstances, the players retained remained higher than those not retained over all. 

Further analysis could be done in the area of win streaks and losing streaks such as their effect on player purchases and the effect of the number of win streaks and losing streaks per player on 30-day retention. We could also look at the 30-day retention rate by age and location, and purchasing patterns based on those metrics as well.



