
# Near Real Time Prediction On Sentiment Of Tweets By Injecting Data Into Elasticsearch And Displaying On Kibana

A simple demo of near realtime sentiment analysis on Tweets using Elasticsearch, Kibana and Twitter API in Python 3

Interested in o president Trump's perfomance ten month after his inauguration, I use Twitter public stream to analyze people's sentiment on topics mentioned his name in a near rea-time way. In this demo, I will show how to pipe data from Twitter, predict the sentiment with Textblob, inject **Hot Data** to ElasticSeach using Twitter APIs and, finally, present the output on Kibana Dashboard.

The following screenshot is one example of showing poeple's sentiment to president Trump's role in Topics (Twitter Hashtag) on the Kibana dashboard. To access the the animation, please visit the [interaction with the Dashboard](https://www.dropbox.com/s/f8vhacnusdwghgb/RecordDashboard.webm?dl=0)

![screencapture](/Visualization/screencapture-localhost-5601-app-kibana-without%20Geo.png)

For simplicity, sentiment of Tweets are predicted by ![model](https://textblob.readthedocs.io/en/dev) trained, and the output are injected into the locally installed Elasticsearch.

## Prerequisites

### Install Elasticsearch

For Mac users who'd like to install manually, please check ![How to install Elasticsearch on mac os x](https://chartio.com/resources/tutorials/how-to-install-elasticsearch-on-mac-os-x/)
You may add the enviroment variable to your bash profile as described in the webpage above or change directory to unzipped file and then, launch Elasticsearch by typing in 'bin/elasticsearch'in Terminal.

### Install Kibana

To install Kibana to get the "Hot Data" visualized, please check ![Hot Data Visualized](https://www.youtube.com/watch?v=psNH33pcGBo)

## The Python Script to Pipe Tweets To Elasticsearch With Sentiment Prediction


```python
import json
from tweepy.streaming import StreamListener
from tweepy import OAuthHandler
from tweepy import Stream
from textblob import TextBlob #predict the sentiment of Tweet, see 'https://textblob.readthedocs.io/en/dev/'
from elasticsearch import Elasticsearch #pip install Elasticsearch if not intalled yet
from datetime import datetime
import calendar
import numpy as np
from http.client import IncompleteRead

#log in to your Twitter Application Management to create an App, url: 'https://apps.twitter.com'
consumer_key = '<Twitter_Consumer_Key>'
consumer_secret = '<Twitter_Consumer_Secret>'
access_token = '<Twitter_Access_Token>'
access_token_secret = '<Twitter_Access_Token_Secret>'

# create instance of elasticsearch
es = Elasticsearch()


class TweetStreamListener(StreamListener):
    
    # re-write the on_data function in the TweetStreamListener
    # This function enables more functions than 'on_status', see 'https://stackoverflow.com/questions/31054656/what-is-the-difference-between-on-data-and-on-status-in-the-tweepy-library'        
    def on_data(self, data):
        # To understand the key-values pulled from Twitter, see 'https://dev.twitter.com/overview/api/tweets'
        dict_data = json.loads(data)
        # pass Tweet into TextBlob to predict the sentiment
        tweet = TextBlob(dict_data["text"]) if "text" in dict_data.keys() else None
        
        # if the object contains Tweet
        if tweet:
            # determine if sentiment is positive, negative, or neutral
            if tweet.sentiment.polarity < 0:
                sentiment = "negative"
            elif tweet.sentiment.polarity == 0:
                sentiment = "neutral"
            else:
                sentiment = "positive"
            
            # print the predicted sentiment with the Tweets
            print(sentiment, tweet.sentiment.polarity, dict_data["text"])
            
    
            # extract the first hashtag from the object
            # transform the Hashtags into proper case
            if len(dict_data["entities"]["hashtags"])>0:
                hashtags=dict_data["entities"]["hashtags"][0]["text"].title()
            else:
                #Elasticeach does not take None object
                hashtags="None"
                      
            # add text and sentiment info to elasticsearch
            es.index(index="logstash-a",
                     # create/inject data into the cluster with index as 'logstash-a'
                     # create the naming pattern in Management/Kinaba later in order to push the data to a dashboard
                     doc_type="test-type",
                     body={"author": dict_data["user"]["screen_name"],
                           "followers":dict_data["user"]["followers_count"],
                           #parse the milliscond since epoch to elasticsearch and reformat into datatime stamp in Kibana later
                           "date": datetime.strptime(dict_data["created_at"], '%a %b %d %H:%M:%S %z %Y'),
                           "message": dict_data["text"]  if "text" in dict_data.keys() else " ",
                           "hashtags":hashtags,
                           "polarity": tweet.sentiment.polarity,
                           "subjectivity": tweet.sentiment.subjectivity,
                           "sentiment": sentiment})
        return True
        
    # on failure, print the error code and do not disconnect
    def on_error(self, status):
        print(status)

if __name__ == '__main__':
    # create instance of the tweepy tweet stream listener
    listener = TweetStreamListener()
    # set twitter keys/tokens
    auth = OAuthHandler(consumer_key, consumer_secret)
    auth.set_access_token(access_token, access_token_secret)    
    # The most exception break up the kernel in my test is ImcompleteRead. This exception handler ensures
    # the stream to resume when breaking up by ImcompleteRead
    while True:
        try:
            # create instance of the tweepy stream
            stream = Stream(auth, listener)
            # search twitter for keyword "trump"
            stream.filter(track=['trump'])
        except IncompleteRead:
            continue
        except KeyboardInterrupt:
            # or however you want to exit this loop
            stream.disconnect()
            break

```

## Pipe Data From Twitter To Kibana Dashboard

1. Launch Elasticsearch
   - type in *Elasticsearch* in Terminal if the environment variable has already been added to the bash profile
   - or change the directory to the unzipped Elasticsearch folder and type *./bin/elasticsearch* to launch<br>
   <br/>
2. Run the python script<br>
    <br/>
3. In a few seconds after the python script starts to run, launch Kibanada by changing the directory to the unzipped Kibana folder and type _./bin/kibana_
    - go to http://localhost:5601, move to __Management__ --> __Index__ -->__+ Create Index Pattern__ to create an index name or pattern
    - check for the ![naming convention](https://www.elastic.co/guide/en/kibana/current/tutorial-define-index) of Kibana index pattern. In this example, the index name in the Python script is __logstash-a__, so I set the naming pattern in as **logstash-***. 'date' is the millisecond since epoch attribute I set in the Python script, so I choose *date* from the drag down box 'Time Filter field name'
    - click 'Create' to create the index pattern
    ![screencapture](/Visualization/screencapture-localhost-5601-app-kibana-naming%20pattern%20and%20datetimestamp.png)<br>
    <br/>
4. To transform the **date** from millisecond to readable datetimestamp, select **+ Add Scripted Field** in **Index Patterns**
    ![screencapture](/Visualization/screencapture-localhost-5601-app-kibana-add%20DateTimeStamp.png)<br>

5. Select **Language**, **Type**, **Format** as the following graph and add _doc['date'].value_ to the field of **script**
    ![screencapture](/Visualization/screencapture-localhost-5601-app-kibana-datetime-transform-detail.png)<br>

6. See ![Kibana User Guide](https://www.elastic.co/guide/en/kibana/current/getting-started) about how to build up the customized graphs in **Visualize** and put them together in **Dashboard**<br>
<br/>

7. In case that I'd like to clear out the data in the elasticsearch index, run _DELETE /logstash-a_ in the console of Dev Tools
     ![screencapture](/Visualization/screencapture-localhost-5601-app-kibana-clear%20out%20data.png)
