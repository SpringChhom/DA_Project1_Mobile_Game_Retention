## Rolling 30-Day Player Retention Query
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
    FROM (
        SELECT 
            joined,
            COUNT (player_id) AS players_joined,
            COUNTIF (retention_status = 1) AS players_retained,
            (COUNTIF (retention_status = 1)) / (COUNT (DISTINCT (player_id))) AS pct_retention
        FROM (
            SELECT
                joined,
                pl.player_id,
                IF((MAX(day)>=joined+30),1,0) AS retention_status
            FROM
                `my-project-46686-1st-try.game_company_data.player_info` AS pl
            LEFT JOIN 
                `my-project-46686-1st-try.game_company_data.matches_info` AS m
                ON pl.player_id = m.player_id
            GROUP BY
                joined,
                pl.player_id)
    GROUP BY 
    joined))
ORDER BY
    joined
```

## Win Streaks and 30-Day Player Retention Query
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
--assign number to each new streak
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
ORDER BY
    longest_streak DESC
```