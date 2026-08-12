# GSP374, Perform Predictive Data Analysis in BigQuery challenge lab

Soccer event data. Five tasks: load two tables, a penalty kick query, a shot distance query, two user defined functions plus a logistic regression model, and a prediction over the 2018 World Cup. About twenty minutes, all from Cloud Shell with `bq`.

Task 1 says to load through the console but the scorer only checks the tables exist, so `bq load` is fine and much quicker.

## The pitch constants change per lab run

This is the thing to watch. One run used goal mouth `(110, 45)` and field `112 x 57`, not the usual `(100, 50)` and `105 x 68`. Read them off the function code blocks in your own lab text. Wrong constants fail tasks 3, 4 and 5 silently, since the queries still run and still return rows.

## Task 1, loading

The lab names two tables. Four more are needed because tasks 2 to 5 join against them.

```
bq mk -d soccer

bq load --autodetect --source_format=NEWLINE_DELIMITED_JSON soccer.EVENTS_TABLE gs://spls/bq-soccer-analytics/events.json
bq load --autodetect --source_format=CSV soccer.TAGS_TABLE gs://spls/bq-soccer-analytics/tags2name.csv
bq load --autodetect --source_format=NEWLINE_DELIMITED_JSON soccer.competitions gs://spls/bq-soccer-analytics/competitions.json
bq load --autodetect --source_format=NEWLINE_DELIMITED_JSON soccer.matches gs://spls/bq-soccer-analytics/matches.json
bq load --autodetect --source_format=NEWLINE_DELIMITED_JSON soccer.teams gs://spls/bq-soccer-analytics/teams.json
bq load --autodetect --source_format=NEWLINE_DELIMITED_JSON soccer.players gs://spls/bq-soccer-analytics/players.json
```

## Task 2, penalty success rate

Join events to players, filter on free kicks with sub event Penalty, count goals with `SUM(IF(101 IN UNNEST(tags.id), 1, 0))`, group by player id and name, keep players with five or more attempts, order by success rate then attempts. Tag 101 means goal.

## Task 3, shot distance

Build a Shots cte that adds an `isGoal` field from the tags array and computes distance with the lab's own constants, filter to shots plus free kick shots and penalties, then aggregate by distance rounded to the nearest meter.

The lab text says distance under fifty while the course reference query uses less than or equal to fifty. The reference version passed.

## Task 4, functions and model

Create both functions exactly as the lab prints them, with its constants. The model:

```
CREATE MODEL `soccer.MODEL_NAME`
OPTIONS(model_type = 'LOGISTIC_REG', input_label_cols = ['isGoal']) AS
SELECT
  Events.subEventName AS shotType,
  (101 IN UNNEST(Events.tags.id)) AS isGoal,
  `soccer.DISTANCE_FN`(Events.positions[ORDINAL(1)].x, Events.positions[ORDINAL(1)].y) AS shotDistance,
  `soccer.ANGLE_FN`(Events.positions[ORDINAL(1)].x, Events.positions[ORDINAL(1)].y) AS shotAngle
FROM `soccer.EVENTS_TABLE` Events
LEFT JOIN `soccer.matches` Matches ON Events.matchId = Matches.wyId
LEFT JOIN `soccer.competitions` Competitions ON Matches.competitionId = Competitions.wyId
WHERE Competitions.name != 'World Cup'
  AND (eventName = 'Shot' OR (eventName = 'Free Kick' AND subEventName IN ('Free kick shot', 'Penalty')))
  AND `soccer.ANGLE_FN`(Events.positions[ORDINAL(1)].x, Events.positions[ORDINAL(1)].y) IS NOT NULL
```

Training took three to five minutes.

The other BQML challenge lab is `gsp327-bigquery-ml-fare-prediction-challenge.md`, where the trap is the same in shape: randomised per run values, and everything else straightforward.

Qualify the function names the same way everywhere. One community script calls them bare in the model query and dataset qualified in the prediction query, so one of the two always fails.

## Task 5, prediction

`ML.PREDICT` over the same features, joined to players and teams for names, filtered to the World Cup competition. Include all shots, not only goals. Some walkthroughs add a goals only filter to rank impressive goals, which answers a different question and narrows the result set past what the check expects.
