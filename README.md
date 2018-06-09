

```python
# Import dependencies
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
import tweepy
import time
import json
import datetime
from config import consumer_key, consumer_secret, access_token, access_token_secret
from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer

# Define sentiment analyzer
analyzer = SentimentIntensityAnalyzer()
```


```python
# Twitter API Keys
consumer_key = consumer_key
consumer_secret = consumer_secret
access_token = access_token
access_token_secret = access_token_secret
```


```python
# Twitter Credentials
auth = tweepy.OAuthHandler(consumer_key, consumer_secret)
auth.set_access_token(access_token, access_token_secret)
api = tweepy.API(auth, parser=tweepy.parsers.JSONParser())
```


```python
# Define a function to plot the sentiment analysis
# Inputs are the lists for the x-axis (tweets ago), y-axis (tweet scores) and user name of the account analyzed
def plot_sentiments(tweets_ago, tweet_scores, user_name):

    # Use the Seaborn's preset Notebook appearance
    sns.set()

    # Pull current date
    now = datetime.datetime.now()

    fig = plt.figure(figsize=(9,6))
    ax = plt.subplot(111)

    ax.plot(tweets_ago, tweet_scores, linestyle = 'solid', marker = 'o', linewidth = .45, label = f'@{user_name}')

    # Add labeling
    plt.title(f'Sentiment Analysis of Tweets ({now.strftime("%m/%d/%Y")})')
    plt.xlabel("Tweets Ago")
    plt.ylabel("Tweet Polarity")

    # Set x and y limits
    plt.xlim(min(tweets_ago) - 5, 5)
    plt.ylim(-1.05,1.05)

    # Shrink current axis by 20%
    box = ax.get_position()
    #ax.set_position([box.x0, box.y0, box.width * 0.8, box.height])

    # Put a legend to the right of the current axis
    ax.legend(loc='upper left', bbox_to_anchor=(1, 1), title = 'Tweets')

    # Save the figure in our current directory, indicate 'tight' so we don't cut off the legend
    plt.savefig('sentiment.png', bbox_inches = 'tight')
```


```python
# Define the function to actually tweet the sentinment analysis given a user name
def tweet_sentiments(tweet_from, user_name):
    
    # Gather last 500 tweets from the user's timeline
    oldest_tweet = None
    
    # Initialize lists to track the scores and number of tweets ago
    tweet_scores = []
    tweets_ago = []
    
    # Use this loop to grab 100 tweets 5 times
    for x in range(5):
        
        # Gather last 100 tweets from the user's timeline
        tweets = api.user_timeline(user_name, count = 100, max_id = oldest_tweet)
    
        # Loop through each tweet and perform a sentiment analysis
        for tweet in tweets:
            # Perform sentiment analysis on the tweet text
            score = analyzer.polarity_scores(tweet['text'])

            # Capture the overall polarity in the list
            tweet_scores.append(score['compound'])
            
            # Increment oldest_tweet
            oldest_tweet = tweet["id"] - 1

    # Create a list for the x-axis based on the number of scores (in case there are <500 tweets)
    for x in range(len(tweet_scores)):
        tweets_ago.append(-1*(x+1))
    
    # Create the plot in the current directory
    plot_sentiments(tweets_ago, tweet_scores, user_name)
    
    # Tweet out the plot
    api.update_with_media("sentiment.png", f"New Tweet Analysis: @{user_name} (Thanks @{tweet_from}!)") 
```


```python
def mentions_plot():
    # Pull all mentions for the bot
    mentions = api.mentions_timeline()

    # Initialize counter to loop through mentions
    counter = 0

    # Loop through all mentions to find any new visualizations to plot
    while counter < len(mentions):

        # Store the user mentions data tweet by tweet starting from the most recent
        # We assume the 2nd mention corresponds to the requests since the first mention is the bot itself
        from_user = mentions[counter]['user']['screen_name']
        user = mentions[counter]['entities']['user_mentions'][1]['screen_name']

        if user not in users_list:

            # Call the tweet_sentiments function to produce the sentiment analysis graph
            tweet_sentiments(from_user, user)

            # Add username to already plotted list
            users_list.append(user)

            # Set the counter high enough to break the while loop
            counter = len(mentions)

        # Else increment the counter and go to the next mention
        else:
            counter += 1
```


```python
# Initialize the users list which tracks which accounts have already been analyzed
users_list = []

while True:
    # Run the mentions plot function
    mentions_plot()
    
    # Restart after 300 seconds (5 minutes)
    time.sleep(300)
```


```python
# Delete all of the account's tweets
tweets = api.home_timeline()

for tweet in tweets:
    api.destroy_status(tweet['id'])
```
