---
layout: post
title:  "Olympics Telegram Bot"
date:   2021-08-12 12:00:00 -0300
image:  telegram_olympics.jpg
tags:   Telegram Bot
---
## Toy Project | Bot no Telegram para notificar sobre eventos das Olimpíadas

This project started when I realized Olympics schedule will be super confusing for Brazilians this year, since our timezone is completely different from Japan's. So I decided to create a Twitter Bot to post when any event is about to start and, by turning notifications on, I could receivve an alert.

There are two main parts here:
- Scraping the schedule
- Creating the actual bot

All of that is made using Python3, Requests and Tweepy.

## Scraping the schedule

To scrape the schedule, first I found a page with a proper table with all the data. [Sporting News](https://www.sportingnews.com/us/athletics/news/olympics-2021-start-schedule-opening-ceremony/9z5omct2mqe211c0ajna5tyj1) has it all, and with that we can go straight to the code.

#### Step 1 - Import libs

We need to import all the modules we'll use:

```python
import os
import pytz
import tweepy
import requests
import env_file
import pandas as pd
import dateutil.parser

import pandas as pd

from lxml import html
from time import sleep
from datetime import datetime, timedelta
```

#### Step 2 - Get the data

In order to get the data, we'll use the Requests module to make the requests and lxml to parse the response. To make the requests and parse the response we simply do:

```python
url = 'https://www.sportingnews.com/us/athletics/news/olympics-2021-start-schedule-opening-ceremony/9z5omct2mqe211c0ajna5tyj1'
response = requests.get(url)
tree = html.fromstring(response.text)
```

By inspecting the page we can find the table elements and their xpaths *//div[@class="content-element__table-container"]*. Now for each table we can iterate over the rows and columns to get the data:

```python
for table in tables:                   # iterate over the tables
    rows = table[0].xpath('.//tr')     # get all rows

    for row in rows[1:]:               # iterate over the rows
        cols = row.xpath('.//td')      # get all colunms for the current row

        sport = cols[0].text           
        event = cols[1].text
        time = cols[2].text

        new_row = {                    # create dict with the current row
            'Sport': sport,
            'Event': event,
            'Time': time,
            'Post': False
        }

        data.append(new_row)            # add new dict to the data

df = pd.DataFrame(data)                 # create dataframe from list
```

We can make one improvement when it comes to dealing with the dates. Not only they have an weird format, they're also based in ET timezone. For that, we'll create a new function that receives the text we got from the table, the amount of days since the start of the olympics and the olympics start date.

```python
def parse_time(time, days_since, start_day):
    day = start_day + timedelta(days=days_since)                                                   # calculate day
    
    start_time, end_time = time.replace('.', '').replace(u'\xa0', ' ').replace(' ', '').split('-') # clean text
    
    if ':' in start_time:                                                                          # parses text according to the minutes format
        start_time = datetime.strptime(start_time, '%I:%M%p')
    else:
        start_time = datetime.strptime(start_time, '%I%p')
        
    start_time = start_time.replace(year=day.year, month=day.month, day=day.day)                   # add date info to the time
    
    if ':' in end_time:
        end_time = datetime.strptime(end_time, '%I:%M%p')
    else:
        end_time = datetime.strptime(end_time, '%I%p')
    
        
    end_time = end_time.replace(year=day.year, month=day.month, day=day.day)
    
    if end_time < start_time:                                                                      # check if the end is in the following day 
        end_time += timedelta(days=1)
    
    start_time = pytz.timezone('US/Eastern').localize(start_time)                                  # add timezone
    end_time = pytz.timezone('US/Eastern').localize(end_time)
    return start_time, end_time
```

And the last thing is to change a bit our *get* function.

```python
def get(self):
    if not os.path.isdir('data'):
        os.mkdir('data')

    url = 'https://www.sportingnews.com/us/athletics/news/olympics-2021-start-schedule-opening-ceremony/9z5omct2mqe211c0ajna5tyj1'
    response = requests.get(url)
    tree = html.fromstring(response.text)
    tables = tree.xpath('//div[@class="content-element__table-container"]')
    data = []
    start_day = datetime(day=20, month=7, year=2021)

    for i, table in enumerate(tables[1:]):
        rows = table[0].xpath('.//tr')

        for row in rows[1:]:
            cols = row.xpath('.//td')

            sport = cols[0].text
            event = cols[1].text
            start_time, end_time = self.parse_time(cols[2].text, i, start_day)

            new_row = {
                'Sport': sport,
                'Event': event,
                'Start Time': start_time,
                'End Time': end_time,
                'Post': False
            }

            data.append(new_row)

    df = pd.DataFrame(data)
    return df
```

#### Step 3 - Create Bot

Now that we have the schedule, we can go ahead and create a bot to post it. The first thing I usually do is create a Bot class, in order to authenticate and create useful wrappers:


```python
class TwitterBot:
    def __init__(self, creds_path=None):
        self.api = self.get_api(creds_path)
    
    # assumes you have a .env file with valid credentials
    def get_api(self, creds_path):
        creds = env_file.get('.env')
        api_key = creds['API_KEY']
        api_secret = creds['API_SECRET']
        access_token = creds['ACCESS_TOKEN']
        access_token_secret = creds['ACCESS_TOKEN_SECRET']

        auth = tweepy.OAuthHandler(api_key, api_secret) 
        auth.set_access_token(access_token, access_token_secret)
        api = tweepy.API(auth)
        return api
    
    def post_tweet(self, tweet):
        self.api.update_status(tweet)
```

We can now create a function to check if there's any event happening soon that should be posted:

```python
def check_post(schedule, bot):
    ET = pytz.timezone('US/Eastern')

    # iterate over the schedule checking for events happening in a 20-second window from now that has not yet been posted
    for i, row in schedule.iterrows():
        now = datetime.now(ET)
        
        if abs(row['Start Time'] - now) < timedelta(seconds=20) and not row['Post']:
            schedule.loc[i, 'Post'] = True
            post_text = get_post_text(row)
            bot.post_tweet(post_text)
```

The last thing is to create a main function to keep a loop on that:

```python
if __name__ == '__main__':
    bot = TwitterBot()

    while True:
        check_post(schedule, bot)
        sleep(5)
```

## The end

The project is not that complex, but yet uses a lot of different technologies and modules. It was nice doing it to practice and to have some fun =D