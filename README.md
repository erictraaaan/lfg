<img src="img/lfg_logo.png" width="200">

# LFG
We often say that a fight in the NHL helps "fire up the team", or "electrifies the building", but what does that really mean? Is there a way to quantify this phenomenon?

LFG is an attempt to measure the "Impact" that a fight has on an NHL hockey game, by comparing the pace of play before and after a fight occurs.

## Summary
Pace of play is measured with 3 metrics:
* Shots per 60 minutes of play (SH/60)
* Hits per 60 minutes of play (HIT/60)
* Penalties per 60 minutes of play (PEN/60)

Using a dataset containing game data for every NHL regular season game between 2015-2021, these metrics are computed pre- and post- fight for both the home and away team.  The metrics are computed as "per 60" values in order to adjust for different lengths of game time before and after a fight occurs.  For a detailed look at how the metrics are computed, see "Data Prep & Analysis".

## Results

The following results have been found using data from all NHL regular season games between 2015-2021.  The percentage shown shows how often the metric _increases_ after a fight.

| Metric        | Home Team          | Away Team  | Combined  |
| ------------- |:------------------:| ----------:| ---------:|
| SH/60         | 41.33%             | 42.32%     | 42.26%    |
| HIT/60        | 23.54%             | 22.12%     | 18.09%    |
| PEN/60        | 37.36%             | 40.15%     | 51.98%    |

_Example interpretation: The total PEN/60 rate for both teams increases after a fight 51.98% of the time._

### Interpreting the Results
Each rate metric is compared before and after a fight occurs.  Following this, the total number of times the post-fight rate is _higher_ than the pre-fight metrics is computed, as a percentage of the total dataset.  For example, the data shows that following a fight, the home team ends up shooting and a __higher rate__ than before the fight _41.33%_ of the time.

---

## Data Prep & Analysis

This section will walk through the Python code used to compute the results above.

### Getting the Raw Game Data

_Code for this section is found in 1-GetFightGameData.ipynb_

In order to get the data required, we start by querying the NHL API.  We start with a list of all regular season games that took place during a given season.  

Define a helper function to make NHL API calls as such: 

```python
    import requests
    import json
    import pandas as pd
    import numpy as np

    nhl_url = 'https://statsapi.web.nhl.com/api/v1/'

    def nhl_request(path, **kwargs):
        res = requests.get(nhl_url+path, **kwargs)
        if res.status_code == 200:
            return res.json()
```

Then, for a given season, we create a function that extracts a list of GameIDs for every regular season game that occurred that year:

```python
    def getSeasonGameIDs(season):
        params= {}
        params['season'] = season
        params['gameType'] = 'R'
        response = nhl_request(f'schedule', params=params)
        gameIDArray = []
        for dateObj in response['dates']:
            date = dateObj['date']
            dateArray.append(date)
            for gameObj in dateObj['games']:
                gameIDArray.append(gameObj['gamePk'])
        return gameIDArray
```

Lastly, we create a function to extract all necessary game play data associated with a given gameID.  For each play, we extract:
* The players involved
* The event type
* The period and period time when the event took place
* Other metadata such as team names/IDs, player names/IDs, and penalty times

We also restrict this function to only extract data if the give game contains a fight: 

```python
    def getFightGameData(gameID):
        events = []
        gameRes = nhl_request(f'game/{gameID}/feed/live')
        hasFight = False
        for penaltyPlay in gameRes['liveData']['plays']['penaltyPlays']:
            play = gameRes['liveData']['plays']['allPlays'][penaltyPlay]
            if play['result']['secondaryType'] == 'Fighting':
                hasFight = True
        if not hasFight:
            # No fight exists in this game.  Return an empty list.
            return events
        else:
            # A fight happened in this game! Store all the data.
            homeTeam = gameRes['gameData']['teams']['home']['name']
            homeId = gameRes['gameData']['teams']['home']['id']
            awayTeam = gameRes['gameData']['teams']['away']['name']
            awayId = gameRes['gameData']['teams']['away']['id']
            for play in gameRes['liveData']['plays']['allPlays']:
                # Extract the necessary data and store it into a dictionary.
                # Full implementation can be found in the source code.
                ...

                ...
                dict1 = {
                    "gameId" : gameID,
                    "homeTeam" : homeTeam,
                    "homeTeamId" : homeId,
                    "awayTeam" : awayTeam,
                    "awayTeamId" : awayId,
                    "event" : event,
                    "eventTypeId" : eventTypeId,
                    "eventIDx" : eventIDx,
                    "eventID" : eventID,
                    "secondaryType" : secondaryType,
                    "penaltySeverity" : penaltySeverity,
                    "penaltyMinutes" : penaltyMinutes,
                    "period" : period,
                    "periodTime" : periodTime,
                    "player1ID" : player1ID,
                    "player1FullName" : player1FullName,
                    "player1Type" : player1Type,
                    "player2ID" : player2ID,
                    "player2FullName" : player2FullName,
                    "player2Type" : player2Type,
                    "player3ID" : player3ID,
                    "player3FullName" : player3FullName,
                    "player3Type" : player3Type,
                    "player4ID" : player4ID,
                    "player4FullName" : player4FullName,
                    "player4Type" : player4Type,
                    "teamID" : teamID,
                    "teamName" : teamName
                }
                events.append(dict1)
            # Return the list of events containing all play data for the game.
            return events
    
```

Lastly, we create a wrapper function to gather all the data for a given season and save it to a CSV file:

```python 
    def getFightGameDataCSV(season):
        gameIDs = getSeasonGameIDs(season)
        data = []
        for gameID in gameIDs:
            data = data + getFightGameData(gameID)
        df = pd.DataFrame(data)
        df.to_csv(f'AllFightGameData-{season}.csv', index=False)
        print(f'Data gathering for {season} complete')
```

Simply executing the function for a number of seasons outputs multiple CSV files containing all the data.  This can be found in the `notebooks` folder of this repository: 

```python 
    # Get fight game data for each year
    getFightGameDataCSV(20152016)
    getFightGameDataCSV(20162017)
    getFightGameDataCSV(20172018)
    getFightGameDataCSV(20182019)
    getFightGameDataCSV(20192020)
    getFightGameDataCSV(20202021)
```
For ease of computation, all the files are then combined into one, named `AllFightGameDataCombined.csv`.  This can also be found in the `notebooks` folder.

### Computing Fight Related Metrics

_Code for this section is found in 3-GetFightanalyticsData.ipynb_

Before diving into the implementation, let's first start by defining the "end-goal" data model, i.e., the final data frame that we want to use for analysis.  We want to end up with a data set where each row in the data represent a fight that occurred.  We want to capture pre- and post- rate metrics for the fight for both teams as well as a combined rate.  As such, the final 'columns' of this data set are as follows:

```
GAMEID
HOMETEAM
HOMEID
AWAYTEAM
AWAYID
PRE HOME SH/60
PRE HOME HIT/60
PRE HOME PENALTY/60
PRE AWAY SH/60
PRE AWAY HIT/60
PRE AWAY PENALTY/60
PRE TOTAL SH/60
PRE TOTAL HIT/60
PRE TOTAL PENALTY/60
FIGHT TIME
FIGHT ID1
FIGHT NAME1
FIGHT ID2
FIGHT NAME2
POST HOME SH/60
POST HOME HIT/60
POST HOME PENALTY/60
POST AWAY SH/60
POST AWAY HIT/60
POST AWAY PENALTY/60
POST TOTAL SH/60
POST TOTAL HIT/60
POST TOTAL PENALTY/60
```

With that in mind, we can start building out our data set.

We start by building out a number of helper functions to help us with things like getting time stamps, total elapsed time, and per sixty metrics.  The implementations of the functions can be found in the notebook: 

```python
    import numpy as np
    import pandas as pd
    import math
    from datetime import datetime

    def getAbsolutePeriodTime(period, time):
        ...
    def getSplitTimeLength(df):
        ...
    def getDateTimeStamp(time):
        ...
    def getElapsedTime(start, end):
        ...
    def getPerSixtyValue(events, elapsedTime):
        ...
```

Next, we create a function to extract the rows of the raw data that contain fighting penalties for a given game (gameID).  Then, we create a subset of the raw data containing only the specified game.  For each fight that occurs in the game, we run the `getFightInfo()` function to extract all the elements needed to create a row in the final dataset, as specified above.

```python
    def getFightDataForGame(gameId, df):
        game_of_interest = df['gameId'] == gameId
        game_df = df[game_of_interest]
        game_df = game_df.reset_index()
        fight_rows = game_df.index[game_df['secondaryType'] == "Fighting"].tolist()[::2]
        output = []
        for fight in fight_rows:
            output.append(getFightInfo(fight, game_df))
        return output
```

In `getFightInto()`, we split the game's data frame pre- and post-fight subsets.  From there, we count the number of shots, hits, and penalties that occur for each team during each subset.  Then, using the timestamp of the fight, we can compute the rate at which each metric occurs, on a per/60 basis.  We then store all this data into a dictionary.  The sample code below shows the implementation for just the home team, but the full code can be found in the notebook.

```python
    def getFightInfo(fight_row, df):
        pre_fight_df = df.iloc[:fight_row,:]
        post_fight_df = df.iloc[fight_row+1:,:]
        pre_fight_df = pre_fight_df.reset_index()
        post_fight_df = post_fight_df.reset_index()
        pre_home_shots = pre_fight_df.index[ (pre_fight_df['eventTypeId'] == "SHOT") 
            & (pre_fight_df['teamName'] == pre_fight_df['homeTeam'])].tolist()
        pre_home_hits = pre_fight_df.index[ (pre_fight_df['eventTypeId'] == "HIT") 
            & (pre_fight_df['teamName'] == pre_fight_df['homeTeam'])].tolist()
        pre_home_penalties = pre_fight_df.index[ (pre_fight_df['eventTypeId'] == "PENALTY") 
            & (pre_fight_df['teamName'] == pre_fight_df['homeTeam'])].tolist()
        post_home_shots = post_fight_df.index[ (post_fight_df['eventTypeId'] == "SHOT") 
            & (post_fight_df['teamName'] == post_fight_df['homeTeam'])].tolist()
        post_home_hits = post_fight_df.index[ (post_fight_df['eventTypeId'] == "HIT") 
            & (post_fight_df['teamName'] == post_fight_df['homeTeam'])].tolist()
        post_away_hits = post_fight_df.index[ (post_fight_df['eventTypeId'] == "HIT") 
            & (post_fight_df['teamName'] == post_fight_df['awayTeam'])].tolist()
        post_home_penalties = post_fight_df.index[ (post_fight_df['eventTypeId'] == "PENALTY") 
            & (post_fight_df['teamName'] == post_fight_df['homeTeam'])].tolist()
        post_away_penalties = post_fight_df.index[ (post_fight_df['eventTypeId'] == "PENALTY") 
            & (post_fight_df['teamName'] == post_fight_df['awayTeam'])].tolist()
        fightData = df.iloc[fight_row]
        pre_fight_duration = getSplitTimeLength(pre_fight_df)
        post_fight_duration = getSplitTimeLength(post_fight_df)
        fightRowDict = {
            'gameid' : df['gameId'].min(),
            'homeTeam' : df['homeTeam'].min(),
            'homeId' : df['homeTeamId'].min(),
            'preShotsHome' : getPerSixtyValue(pre_home_shots,pre_fight_duration),
            'preHitsHome' : getPerSixtyValue(pre_home_hits,pre_fight_duration),
            'prePenaltiesHome' : getPerSixtyValue(pre_home_penalties,pre_fight_duration),
            'fightTime' : getAbsolutePeriodTime(fightData['period'],fightData['periodTime']),
            'player1Id' : fightData['player1ID'],
            'player1Name' : fightData['player1FullName'],
            'player2Id' : fightData['player2ID'],
            'player2Name' : fightData['player2FullName'],
            'postShotsHome' : getPerSixtyValue(post_home_shots,post_fight_duration),
            'postHitsHome' : getPerSixtyValue(post_home_hits,post_fight_duration),
            'postPenaltiesHome' : getPerSixtyValue(post_home_penalties,post_fight_duration),
        }
        return fightRowDict
```

Lastly, we iterate through the entire raw data set and compute the fight metrics for each fight.  We store the results in `FightAnalytics.csv`.

```python
    df = pd.read_csv('AllFightGameDataCombined.csv')
    gameIds = df['gameId'].unique()
    outputData = []
    for gameId in gameIds:
        outputData = outputData +  getFightDataForGame(gameId,df)
    output_df = pd.DataFrame.from_dict(outputData)
    output_df.to_csv(f'FightAnalytics.csv', index=False)
```

### Interpreting a Line from the Dataset

A line from the final data set is shown below.  Here is an example of what we can draw from the data:

After the fight between Robert Bortuzzo and Derek Dorsett:
* _Home team SH/60 went from 28.85 to 33.33_
* _Total PEN/60 for the game went from 7.69 to 10.42_

| Column             | Value                |
| :-----------------:| :-------------------:|
| gameid             | 2015020062           |
| homeTeam	         | Vancouver Canucks    |
| homeId             | 23                   |
| awayTeam           | St. Louis Blues      |
| awayId             | 19                   |
| preShotsHome       | 28.85                |
| preHitsHome        | 13.46                |
| prePenaltiesHome   | 3.85                 |
| preShotsAway       | 28.85                |
| preHitsAway        | 15.38                |
| prePenaltiesAway   | 3.85                 | 
| preShotsTotal      | 57.69                |
| preHitsTotal       | 28.85                |
| prePenaltiesTotal  | 7.69                 |
| fightTime          | 31:12                |
| player1Id          | 8474145              |
| player1Name        | Robert Bortuzzo      |
| player2Id          | 8473432              |
| player2Name        | Derek Dorsett        |
| postShotsHome      | 33.33                |
| postHitsHome       | 16.67                |
| postPenaltiesHome  | 6.25                 |
| postShotsAway      | 27.08                |
| postHitsAway       | 6.25                 |
| postPenaltiesAway  | 4.17                 |
| postShotsTotal     | 60.42                |
| postHitsTotal      | 22.92                |
| postPenaltiesTotal | 10.42                |

### Analyzing the Results

_Code for this section is found in 4-FightAnalytics.ipynb_

Here, we can do some very simple analysis of the results.  We can determine the number of rows in the data set where a given _post-_ metric is **higher** than it's corresponding _pre-_ metric.  This is done as follows (home team only shown in the example)

```python
    '''
    Import the data
    '''
    df = pd.read_csv('FightAnalytics.csv')

    more_home_hits = df.index[ (df['postHitsHome'] > df['preHitsHome']) 
        & (df['preHitsHome'] > 0) & (df['postHitsHome'] > 0)].tolist()
    more_home_penalties = df.index[ (df['postPenaltiesHome'] > df['prePenaltiesHome']) 
        & (df['postPenaltiesHome'] > 0) & (df['prePenaltiesHome'] > 0)].tolist()
    more_home_shots = df.index[ (df['postShotsHome'] > df['preShotsHome']) 
        & (df['postShotsHome'] > 0) & (df['preShotsHome'] > 0)].tolist()
    home_hit_percent = (len(more_home_hits) / df.shape[0]) * 100
    home_pen_percent = (len(more_home_penalties) / df.shape[0]) * 100
    home_shot_percent = (len(more_home_shots) / df.shape[0]) * 100
```

We can then round the data to 2 decimal places and print it out:

```python
    print( "More home hits " + "%.2f" % round(home_hit_percent, 2) + "% of the time")
    print( "More home penalties " + "%.2f" % round(home_pen_percent, 2) + "% of the time") 
    print( "More home shots " + "%.2f" % round(home_shot_percent, 2) + "% of the time") 
```

```
    More home hits 23.54% of the time
    More home penalties 37.36% of the time
    More home shots 41.33% of the time
```

## Next Steps

This project was a fun way to practice data aggregation and processing in Python using the Pandas library.  There is also a lot of potential to improve on this project.  Some examples of this include:

* Creating a statistic (LFG) that represents a  player's individual contribution to the game pace change after a fight
    * Eg. a player with an LFG statistic of 108 means that the game pace increases by 8% after this player fights
    * Likewise, a player with an LFG of 95 means that the game pace decreases by 5% after this player fights

This could be a fun future project.  Stay tuned!
