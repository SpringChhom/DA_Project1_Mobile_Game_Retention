# Data Analytics Bootcamp - Juno College - Project 1 - 30-Day Player Retention in a Mobile Game App

## Welcome to my first Data Analysis Project on Github!

For this project, two teammates and I were tasked with using SQL and Google Sheets to analyze player retention for a mobile game in which players could have matches against one another and purchase items in the game store. Upon the one year anniversary of this game, the mobile game company asked us to report on the rolling 30-day retention of players as well as an additional metric related to retention. The company provided us with a dataset consisting of four tables which included data on the players based on their player ID: when they joined, their age and location, what they purchased and when, and what matches they played and when.

For the purpose of this report, the word “player(s)” is referring to the unique player IDs under the assumption that real life players did not share their player IDs with others.

## Rolling 30-Day Retention

Rolling 30-day retention refers to whether a player has engaged with the game 30 days after having joined, and if they have, they are considered retained. For this company, they were more specifically interested in whether the players had participated in a one on one match 30 days after having joined. To answer this question, we calculated the rolling 30 day retention using several nested SQL queries which I will now break down into steps.

Our first step, which is the innermost SQL query in the nested queries, was to divide the players into two groups, those who had played a match 30 days after joining (retained) and those who hadn’t (not retained.) We started this by using the JOIN clause to link the matchs_info and the player_info tables.  We also used the MAX function to find the last match played by each player, then calculating whether that last match was more than or equal to 30 days after the player’s join day. We use the IF function to label retained players with the integer 1 and non-retained players with a 0 and we named that column retention_status. The JOIN clause chosen was LEFT JOIN in order to include players who had joined but not played in any matches.

```
{
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
}
```

