
# A short insight into Brandwatch as found on Twitter


```python
from IPython.display import HTML
HTML('''<script>
code_show=true; 
function code_toggle() {
 if (code_show){
 $('div.input').hide();
 } else {
 $('div.input').show();
 }
 code_show = !code_show
} 
$( document ).ready(code_toggle);
</script>
The raw code for this IPython notebook is by default hidden for easier reading.
To toggle on/off the raw code, click <a href="javascript:code_toggle()">here</a>.''')
```




<script>
code_show=true; 
function code_toggle() {
 if (code_show){
 $('div.input').hide();
 } else {
 $('div.input').show();
 }
 code_show = !code_show
} 
$( document ).ready(code_toggle);
</script>
The raw code for this IPython notebook is by default hidden for easier reading.
To toggle on/off the raw code, click <a href="javascript:code_toggle()">here</a>.




```python
HTML(""" <style>
div.text_cell_render h1 { /* Main titles bigger, centered */
font-size: 2.2em;
line-height:1.4em;
text-align:center;
}
div.text_cell_render { /* Customize text cells */
font-size:1.15em;
}</style>""")
```




 <style>
div.text_cell_render h1 { /* Main titles bigger, centered */
font-size: 2.2em;
line-height:1.4em;
text-align:center;
}
div.text_cell_render { /* Customize text cells */
font-size:1.15em;
}</style>



Twitter recently celebrated its 10th anniversary. Simplicity is perhaps one of the keys to the success of the 3rd most popular social media that often ranks in the Top 10 of most visited websites worldwide (Alexa). I remember setting up my Twitter account because after Dominique Strauss Kahn was arrested in New-York, some journalists were inside the courthouse where he was being heard for the first time, and they were live tweeting the event. This was 5 years ago, and probably when I realised that Twitter had also become a tool for journalists, the last one in the race for being the first to bring the news. Since then, Twitter has kept growing and is now used for many purposes. From connecting people who share an interest to bringing the news and advertise jobs openings.

Here we have a look into the kind of information we can derive from Twitter activities when accounts and tweets are sampled from two populations, one that is employed by Brandwatch when the other is only known to mention the company within their posts or summaries. Of course both populations are likely to interesect, and it would thus be hard to obtain something such as a classifier to distinguish employees from other observations.

There are however several interesting leads to follow, and we will start with trying to make sense of the data at hand in terms of its geography: where in the world were located the people whose tweets we are looking at? We will then move forward with some quantitative prediction analysis, see if we may hope to classify tweets (is it an original post or a share?) as well as the number of retweets. Obviously, the key information contained in a tweet lies within the text of a post and perhaps its author's summary. We will therefore eventually take a look at these texts and for instance try to use it to distinguish what different topics are people talking about in different parts of the world.

But first of all, we need to read the data. It was organized in a tar archive which we must untar before sifting through the various JSONs contained in zipped files.

## Unlock the data

For now, we choose to keep track of our two sources (employees or simply people mentioning Brandwatch) by separating the data. We will eventually concatenate the two for applications. Some zipped files contained data that didn't seem valid JSONs, but that became so after they were split. These were apparently JSONs that had been put end to end. The data is then first obtained as two separate lists, each being made of either dictionaries or lists of dictionaries.


```python
# -*- coding: UTF-8 -*-
### Libraries
import json
import gzip
import sys
import tarfile

### separate the data sources
authors_name, mentions_name = [], []
authors, mentions = [], []
### untar the files
tar = tarfile.open("ds-test.tar", "r")
tar.extractall()
        
for name in tar.getnames():
    ### sift through the .gz files only, first for Brandwatch employees
    if "authors" in name and "_part" not in name and name.endswith('.gz'):
        authors_name.append(name)
        temp = gzip.open(name, "rb")
        rtemp = temp.read()
        ### try to directly read the content
        try:
            d = json.loads(rtemp)
            #d = json.loads(temp.read())#.decode("utf-8"))
            authors.append(d)
        ### some files need a little pre-processing, some JSON strings are laid end to end
        except:
            dtemp = []
            stemp = rtemp.decode('latin-1').split("\n")
            ### loop over all the resulting JSON strings
            for ttemp in stemp:
                try:
                    dtemp.append(json.loads(ttemp))
                ### some still require some fixing with special characters (might be due
                ### to my MAC OS Swiss German laptop's configuration)
                except:
                    copy, first = "", True
                    for t in ttemp.split('"'):
                        if first:
                            ts = str(t)
                            first = False
                        else:
                            ts = '"'+str(t)
                        copy += ts.decode('utf-8', 'ignore').replace("\'", "")\
                                                            .replace("\\n", "")
                    if copy == "":
                        copy = "{}"
                    dtemp.append(json.loads(copy))
            
            authors.append(dtemp)
    ### sift through the tweets that mention Brandwatch
    elif "mentions" in name and "_part" not in name and name.endswith('.gz'):
        mentions_name.append(name)
        temp = gzip.open(name, "rb")
        rtemp = temp.read()
        
        try:
            d = json.loads(rtemp)
            mentions.append(d)
        except:
            dtemp = []
            stemp = rtemp.decode('latin-1').split("\n")
            for ttemp in stemp:
                try:
                    dtemp.append(json.loads(ttemp))
                except:
                    copy, first = "", True
                    for t in ttemp.split('"'):
                        if first:
                            ts = str(t)
                            first = False
                        else:
                            ts = '"'+str(t)
                        copy += ts.decode('utf-8', 'ignore').replace("\'", "")\
                                                            .replace("\\n", "")
                    if copy == "":
                        copy = "{}"
                    dtemp.append(json.loads(copy))
            
            mentions.append(dtemp)
### close the tar file
tar.close()
```

Before we move on, let us first check that we successfully opened all the data with the following quick test.


```python
import codecs
import random
### checking the results, try to see if something is missing
indA = [(i, authors_name[i]) for i in range(len(authors_name)) if authors[i] == [{}]]
indM = [(i, mentions_name[i]) for i in range(len(mentions_name)) if mentions[i] == [{}]]

print random.sample(indA, 3)
print random.sample(indM, 3)
### the files left empty seem to be only from empty .gz elements (checked on my disk)
```

    [(892, 'ds-test/brandwatch-authors/dt=20160130/part-00002.gz'), (794, 'ds-test/brandwatch-authors/dt=20160210/part-00004.gz'), (147, 'ds-test/brandwatch-authors/dt=20160416/part-00007.gz')]
    [(280, 'ds-test/brandwatch-mentions/dt=20160402/part-00000.gz'), (312, 'ds-test/brandwatch-mentions/dt=20160329/part-00002.gz'), (304, 'ds-test/brandwatch-mentions/dt=20160330/part-00004.gz')]


Now that we have obtained the data as lists of JSONs, let us modify and clean the data a little. The lists of dictionaries are also now unnested.


```python
### Data cleaning: get rid of some empty dictionaries and unnested dictionaries put end to end
aut_dic = []
for i in range(len(authors)):
    if type(authors[i]) == dict:
        aut_dic.append(authors[i])
    else:
        for dic in authors[i]:
            if dic != {}:
                aut_dic.append(dic)
            
print "Number of listed tweet roots for Brandwatch employees: {}".format(len(aut_dic))
            
men_dic = []
for i in range(len(mentions)):
    if type(mentions[i]) == dict:
        men_dic.append(mentions[i])
    else:
        for dic in mentions[i]:
            if dic != {}:
                men_dic.append(dic)
            
print "Number of listed tweet roots mentioning Brandwatch: {}".format(len(men_dic))

### list of all tweet bodies from both sources
tweets_aut, tweets_men = [], []
for dic in aut_dic:
    tweets_aut.append(dic['body'])
    
for dic in men_dic:
    tweets_men.append(dic['body'])
```

    Number of listed tweet roots for Brandwatch employees: 2976
    Number of listed tweet roots mentioning Brandwatch: 2371


In the following chunks of code, we look more closely at the data: the JSONs are like dictionaries with different set of keys which will later be the features we may use. We look at all the available features, and compare those between our two different sources.


```python
import numpy as np
### looking at the different dictionaries lengths
autLen = np.unique([len(dic) for dic in aut_dic])
menLen = np.unique([len(dic) for dic in men_dic])
### and splitting them accordingly
aut_split = [[dic for dic in aut_dic if len(dic) == l] for l in autLen]
men_split = [[dic for dic in men_dic if len(dic) == l] for l in menLen]

### list of unique dictionaries entries split per size
aut_split_keys = [[dic.keys() for dic in aut_dic if len(dic.keys()) == l]\
                                for l in np.unique([len(dic) for dic in aut_dic])]
men_split_keys = [[dic.keys() for dic in men_dic if len(dic.keys()) == l]\
                                for l in np.unique([len(dic) for dic in men_dic])]
### looking at the different lists of dictionariy entries
aut_diff_keys, aut_keys_list, men_diff_keys, men_keys_list = [], [], [], []
for lkey in aut_split_keys:
    temp = []
    for key in lkey:
        if key not in temp:
            temp.append(key)
        for entry in key:
            if entry not in aut_keys_list:
                aut_keys_list.append(entry)
    aut_diff_keys.append(temp)
    
aut_diff = [[(i, j, key) for i in range(len(ldic)) for j in range(len(ldic))\
                   for key in ldic[j] if key not in ldic[i]] for ldic in aut_diff_keys]

for lkey in men_split_keys:
    temp = []
    for key in lkey:
        if key not in temp:
            temp.append(key)
        for entry in key:
            if entry not in men_keys_list:
                men_keys_list.append(entry)
    men_diff_keys.append(temp)
    
men_diff = [[(i, j, key) for i in range(len(ldic)) for j in range(i+1, len(ldic))\
                   for key in ldic[j] if key not in ldic[i]] for ldic in men_diff_keys]
### some dictionaries differ in the two sources, a little care is needed to build a Data Frame
print [set(aut) for aut in aut_diff] == [set(men) for men in men_diff]
### but the total list of dictionary entries is the same
print set(aut_keys_list) == set(men_keys_list)
```

    False
    True



```python
### list of all dictionary keys, to have a look at all available features
col_aut_list, col_men_list = [], []
for dic in aut_dic:
    for key in dic.keys():
        if key not in col_aut_list:
            if type(dic[key]) != dict:
                col_aut_list.append(key)
            else:
                for kkey in dic[key].keys():
                    if kkey not in col_aut_list:
                        col_aut_list.append(kkey)                        
for dic in men_dic:
    for key in dic.keys():
        if key not in col_men_list:
            if type(dic[key]) != dict:
                col_men_list.append(key)
            else:
                for kkey in dic[key].keys():
                    if kkey not in col_men_list:
                        col_men_list.append(kkey)
```

## Converting the data to DataFrame

We use the pandas library to organize our data into dataframes. To do so, we select some of the features that seemed the most promising for future use. Once again, we keep the two sources separated, pdA will contain all the observations for Employees only while pdM will contatin the other observations. The two sources having dictionaries of different length, we also separate the loops to sift through both databases.

We start with pdA:


```python
### import pandas for Data Frames
import pandas as pd
### list of features present in aut_dic but not in men_dic and vice versa
inAnM, inMnA = [], []
for key1, key2 in zip(col_aut_list, col_men_list):
    if key1 not in col_men_list:
        inAnM.append(key1)
    if key2 not in col_aut_list:
        inMnA.append(key2)

### DataFrame of various features for tweets authored by Brandwatch employees
pdA = pd.DataFrame()
count = 0
for dic in aut_dic:
#    print "{} steps left".format(len(aut_dic)-count)
    temp = {}
    temp['name'] = dic['actor']['displayName']
    temp['favoritesCount'] = dic['actor']['favoritesCount']
    temp['followersCount'] = dic['actor']['followersCount']
    temp['friendsCount'] = dic['actor']['friendsCount']
    temp['id'] = dic['actor']['id'].split("id:twitter.com:")[1]
    temp['image'] = dic['actor']['image']
    temp['languages'] = dic['actor']['languages'][0]
    temp['listedCount'] = dic['actor']['listedCount']
    temp['location'] = dic['actor']['location']['displayName'] if dic['actor'] \
                                                     and 'location' in dic['actor'].keys()\
                                                     else ""
    temp['postedTime'] = dic['actor']['postedTime']
    temp['preferredUsername'] = dic['actor']['preferredUsername']
    temp['statusesCount'] = dic['actor']['statusesCount']
    temp['summary'] = dic['actor']['summary']
    temp['twitterTimeZone'] = dic['actor']['twitterTimeZone']
    temp['body'] = dic['body']#.encode('latin-1').decode('utf-8')
    temp['favoritesCount'] = dic['favoritesCount']
    temp['generator'] = dic['generator']['displayName']
    temp['klout_score'] = dic['gnip']['klout_score'] if dic['gnip'] \
                                                     and 'klout_score' in dic['gnip'].keys()\
                                                     else ""
    
    temp['postedTime'] = dic['postedTime']
    temp['provider'] = dic['provider']['displayName']
    temp['retweetCount'] = dic['retweetCount']
    temp['verb'] = dic['verb']
    temp['user_mentions'] = "-".join([str(t["id"])\
                                      for t in dic['twitter_entities']['user_mentions']])\
            if dic['twitter_entities']['user_mentions']\
                and dic['twitter_entities']['user_mentions'] != []\
            else ""
    
    
    temp['url'] = dic['gnip']['urls'][0]['expanded_url']\
                        if 'urls' in dic['gnip'].keys() else ""
    temp["sharetype"] = dic['twitter_extended_entities']['media'][0]['type']\
                        if 'twitter_extended_entities' in dic.keys()\
                            and 'media' in dic['twitter_extended_entities'].keys()\
                            and dic['twitter_extended_entities']['media'][0]\
                            and 'type' in dic['twitter_extended_entities']['media'][0].keys()\
                        else ""
    
    temp['fol_name'] = dic['object']['actor']['displayName']\
                        if 'actor' in dic['object'].keys()\
                            and 'displayName' in dic['object']['actor'].keys()\
                        else ""
    temp['fol_folCount'] = dic['object']['followersCount']\
                        if dic['object'] and 'followersCount' in dic['object'].keys() else ""
    temp['fol_id'] = dic['object']['id'].split("2005:")[1] if dic['object'] else ""
    temp['fol_location'] = dic['object']['location']\
                        if dic['object'] and 'location' in dic['object'].keys() else ""
    temp['fol_summary'] = dic['object']['summary']\
                        if dic['object'] and 'summary' in dic['object'].keys() else ""
    temp['fol_body'] = dic['object']['body']\
                        if dic['object'] and 'body' in dic['object'].keys() else ""
    
    temp['fol_hashtags'] = dic['twitter_entities']['hashtags'][0]\
                        if dic['twitter_entities']['hashtags'] else ""
    temp['fol_type'] = dic['twitter_entities']['media'][0]['type']\
                        if 'twitter_entities' in dic.keys()\
                            and 'media' in dic['twitter_entities'].keys()\
                        else ""
                
    temp['geo'] = str(dic['geo']['coordinates']) if 'geo' in dic.keys() else ""

    pdA = pd.concat([pdA, pd.DataFrame(temp, index=[count])], axis=0)
    count += 1
```

and repeat the same process for pdM:


```python
### DataFrame of various features for tweets authored by random people who happened to mention
### Brandwatch within their tweets
pdM = pd.DataFrame()
count = 0
for dic in men_dic:
    temp = {}
    temp['name'] = dic['actor']['displayName']
    temp['favoritesCount'] = dic['actor']['favoritesCount']
    temp['followersCount'] = dic['actor']['followersCount']
    temp['friendsCount'] = dic['actor']['friendsCount']
    temp['id'] = dic['actor']['id'].split("id:twitter.com:")[1]
    temp['image'] = dic['actor']['image']
    temp['languages'] = dic['actor']['languages'][0]
    temp['listedCount'] = dic['actor']['listedCount']
    temp['location'] = dic['actor']['location']['displayName'] if dic['actor'] \
                                                     and 'location' in dic['actor'].keys()\
                                                     else ""
    temp['postedTime'] = dic['actor']['postedTime']
    temp['preferredUsername'] = dic['actor']['preferredUsername']
    temp['statusesCount'] = dic['actor']['statusesCount']
    temp['summary'] = dic['actor']['summary']
    temp['twitterTimeZone'] = dic['actor']['twitterTimeZone']
    temp['body'] = dic['body']
    temp['favoritesCount'] = dic['favoritesCount']
    temp['generator'] = dic['generator']['displayName']
    temp['klout_score'] = dic['gnip']['klout_score'] if dic['gnip'] \
                                                     and 'klout_score' in dic['gnip'].keys()\
                                                     else ""
    
    temp['postedTime'] = dic['postedTime']
    temp['provider'] = dic['provider']['displayName']
    temp['retweetCount'] = dic['retweetCount']
    temp['verb'] = dic['verb']
    temp['user_mentions'] = "-".join([str(t["id"])\
                                      for t in dic['twitter_entities']['user_mentions']])\
            if dic['twitter_entities']['user_mentions']\
                and dic['twitter_entities']['user_mentions'] != []\
            else ""
    
    
    temp['url'] = dic['gnip']['urls'][0]['expanded_url']\
                        if 'urls' in dic['gnip'].keys() else ""
    temp["sharetype"] = dic['twitter_extended_entities']['media'][0]['type']\
                        if 'twitter_extended_entities' in dic.keys()\
                            and 'media' in dic['twitter_extended_entities'].keys()\
                            and dic['twitter_extended_entities']['media'][0]\
                            and 'type' in dic['twitter_extended_entities']['media'][0].keys()\
                        else ""
    
    temp['fol_name'] = dic['object']['actor']['displayName']\
                        if 'actor' in dic['object'].keys()\
                            and 'displayName' in dic['object']['actor'].keys()\
                        else ""
    temp['fol_folCount'] = dic['object']['followersCount']\
                        if dic['object'] and 'followersCount' in dic['object'].keys() else ""
    temp['fol_id'] = dic['object']['id'].split("2005:")[1] if dic['object'] else ""
    temp['fol_location'] = dic['object']['location']\
                        if dic['object'] and 'location' in dic['object'].keys() else ""
    temp['fol_summary'] = dic['object']['summary']\
                        if dic['object'] and 'summary' in dic['object'].keys() else ""
    temp['fol_body'] = dic['object']['body']\
                        if dic['object'] and 'body' in dic['object'].keys() else ""
    
    temp['fol_hashtags'] = dic['twitter_entities']['hashtags'][0]\
                        if dic['twitter_entities']['hashtags'] else ""
    temp['fol_type'] = dic['twitter_entities']['media'][0]['type']\
                        if 'twitter_entities' in dic.keys()\
                            and 'media' in dic['twitter_entities'].keys()\
                        else ""
                
    temp['geo'] = str(dic['geo']['coordinates']) if 'geo' in dic.keys() else ""

    pdM = pd.concat([pdM, pd.DataFrame(temp, index=[count])], axis=0)
    count += 1
```

Let us have a look at the result:


```python
pd.set_option('display.max_columns', 50)
pdM.iloc[:3, :]
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>body</th>
      <th>favoritesCount</th>
      <th>fol_body</th>
      <th>fol_folCount</th>
      <th>fol_hashtags</th>
      <th>fol_id</th>
      <th>fol_location</th>
      <th>fol_name</th>
      <th>fol_summary</th>
      <th>fol_type</th>
      <th>followersCount</th>
      <th>friendsCount</th>
      <th>generator</th>
      <th>geo</th>
      <th>id</th>
      <th>image</th>
      <th>klout_score</th>
      <th>languages</th>
      <th>listedCount</th>
      <th>location</th>
      <th>name</th>
      <th>postedTime</th>
      <th>preferredUsername</th>
      <th>provider</th>
      <th>retweetCount</th>
      <th>sharetype</th>
      <th>statusesCount</th>
      <th>summary</th>
      <th>twitterTimeZone</th>
      <th>url</th>
      <th>user_mentions</th>
      <th>verb</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>RT @Brandwatch: How to use social data to perf...</td>
      <td>0</td>
      <td>How to use social data to perfect and personal...</td>
      <td></td>
      <td></td>
      <td>726352169780178946</td>
      <td></td>
      <td>Brandwatch</td>
      <td></td>
      <td>photo</td>
      <td>47272</td>
      <td>40188</td>
      <td>RoundTeam</td>
      <td></td>
      <td>1169760512</td>
      <td>https://pbs.twimg.com/profile_images/564276653...</td>
      <td>65</td>
      <td>en</td>
      <td>2686</td>
      <td>Seattle, WA</td>
      <td>Evan Dunn</td>
      <td>2016-04-30T10:14:21.000Z</td>
      <td>evanpdunn</td>
      <td>Twitter</td>
      <td>1</td>
      <td>photo</td>
      <td>49165</td>
      <td>Slam Poet | Quant Mktg @TransformDGTL | Latest...</td>
      <td>Pacific Time (US &amp; Canada)</td>
      <td>http://www.business2community.com/social-data/...</td>
      <td>86107125</td>
      <td>share</td>
    </tr>
    <tr>
      <th>1</th>
      <td>RT @DrJillianNey: Generating actionable market...</td>
      <td>0</td>
      <td>Generating actionable marketing intelligence w...</td>
      <td></td>
      <td></td>
      <td>725995038401019905</td>
      <td></td>
      <td>Dr Jillian Ney</td>
      <td></td>
      <td></td>
      <td>2733</td>
      <td>1369</td>
      <td>Top Event Retweets</td>
      <td></td>
      <td>3395767516</td>
      <td>https://pbs.twimg.com/profile_images/626782661...</td>
      <td>56</td>
      <td>en</td>
      <td>4066</td>
      <td>San Francisco, CA</td>
      <td>SolidRecommendations</td>
      <td>2016-04-30T17:29:06.000Z</td>
      <td>SolidRecs</td>
      <td>Twitter</td>
      <td>2</td>
      <td></td>
      <td>73920</td>
      <td>Showcasing the best online tools to the people...</td>
      <td>Pacific Time (US &amp; Canada)</td>
      <td>https://www.brandwatch.com/2016/04/5-hacks-gen...</td>
      <td>145177177-86107125</td>
      <td>share</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Brandwatch is looking for: Freelance Social Me...</td>
      <td>0</td>
      <td></td>
      <td></td>
      <td>NaN</td>
      <td>726446148211367936</td>
      <td></td>
      <td></td>
      <td>Brandwatch is looking for: Freelance Social Me...</td>
      <td></td>
      <td>198</td>
      <td>522</td>
      <td>Jobvite</td>
      <td></td>
      <td>102946757</td>
      <td>https://pbs.twimg.com/profile_images/376404277...</td>
      <td>29</td>
      <td>en</td>
      <td>9</td>
      <td>Brighton, UK</td>
      <td>Dan Leonard</td>
      <td>2016-04-30T16:20:33.000Z</td>
      <td>danleonard85</td>
      <td>Twitter</td>
      <td>0</td>
      <td></td>
      <td>1157</td>
      <td>-  Head of Customer Support with @Brandwatch (...</td>
      <td>London</td>
      <td>https://app.jobvite.com/Jobvite/jobvite.aspx?b...</td>
      <td></td>
      <td>post</td>
    </tr>
  </tbody>
</table>
</div>



Note that the data is not perfect and some entries remain emtpy.

# Visualization of tweets' origin

At least to me, it matters to know where people are located. It helps understand how broad an audience we are facing. Is Brandwatch mostly a European thing? With offices in America, it seems fair to assume the brand raises interest in other distant countries. But how large exactly is that interest?

It is not exactly as straightforward to proceed here. There are indeed far too many missing data in the 'geo' feature that contains longitude and latitude coordinates to base our judgment on that only. Instead we could use either the 'location' or the 'twitterTimeZone' features. The latter needs a little pre-processing work but is quite easy to use and might seem more reliable an information. The 'location' data is indeed set by the user and has more than often nothing to do with a real location but ranges from various jokes (see the 'exclude' list below) to political messages. As the biggest source of geographical information, we start by trying to use the 'location' feature nonetheless to obtain an image of the worlwide distribution.

## Using a Google API to recover coordinates

After setting up the Google API we may sift through GoogleMaps Database with the location names we have. And for that, we now merge the two dataframes pdA and pdM into a third one, pdTot.

### Worldwide distribution 


```python
### Use Google Maps API to retrieve coordinates associated to the entries of the field
### twitterTimeZone
import googlemaps
from googlemaps import client, convert

gmaps = client.Client(key="AIzaSyAQfwuO_dZx2kam_uK4xq439rEYhkNgwm8")

### Form a dataframe totalizing all the observations
pdTot = pd.concat([pdA, pdM], axis=0).reset_index()
pdTot = pdTot.iloc[:, 1:]
### check whether a character is written in ASCII format
def is_ascii(s):
    return all(ord(c) < 128 for c in s)
### list of characters to exclude from the list of locations
exclude = ["#muslim", "%", u"\xc3", u'|-/', "iPhone", u'twd, ahs, got, htgawm', "@",\
           "BieberLand", "Asteroid B612", "Bi-coastal", "Bin dort", "I-75", "I.E.",\
          "Ici ou ailleurs", "Internet", "Local", "On the MY WORLD TOUR!!!",\
          "On the go", "On your Timeline", "One Image All Networks", "french libertarian",\
          "http://Bit.ly/BounceVideo", "im Wald", "international", "j follows", "linlithgow",\
          "Snapchat", "tra le braccia di Niall", "twitterverse"]
Location = [x for x in np.unique(np.array(pdTot['location'])) if x != "" and is_ascii(x) \
                                   and all([t not in x for t in exclude])]

typeLoc, boundsLoc, locLoc = {}, {}, {}
for loc in Location:
    if loc:
        if "/" in loc:
            loc = loc.split("/")[0]
        if "#" in loc:
            loc = loc.split("#")[0]
        try:
            dloc = gmaps.geocode(loc)
            locLoc[loc] = [dloc[0]['geometry']['location']['lat'],\
                          dloc[0]['geometry']['location']['lng']]
            if 'types' in dloc[0].keys() and len(dloc[0]['types']) > 0:
                typeLoc[loc] = dloc[0]['types'][0]
            if 'bounds' in dloc[0].keys():
                boundsLoc[loc] = [[dloc[0]['geometry']['bounds']['northeast']['lat'],\
                                 dloc[0]['geometry']['bounds']['northeast']['lng']],\
                                [dloc[0]['geometry']['bounds']['southwest']['lat'],\
                                 dloc[0]['geometry']['bounds']['southwest']['lng']]]
        except:
            pass
```

To make a plot, we can now use the following dictionary:


```python
locLoc
```




    {u' Bromley, London BR3 1JF': [51.4113658, -0.0456431],
     u'128.0.0.1': [44.108341, 24.368537],
     u'26215 Wiefelstede': [53.2549953, 8.1175835],
     u'48.943494,8.670127': [48.94361749999999, 8.670328999999999],
     u'50.829733,-0.148918': [50.8297866, -0.1490137],
     u'805, California': [34.6050598, -118.1528395],
     u'85004': [33.4528292, -112.0685027],
     u'A Guarda, Spain 36780': [41.9041915, -8.8734975],
     u'ATL - DC - LA - NASHVILLE ': [36.1626638, -86.7816016],
     u'Aachen': [50.7753455, 6.083886800000001],
     u'Aarau ': [47.390434, 8.0457015],
     u'Adirondacks & NYC ': [40.3974376, -74.39557049999999],
     u'Ahmedabad, India': [23.022505, 72.5713621],
     u'Aix en provence, Fr': [43.529742, 5.447426999999999],
     u'Akumal': [20.3959736, -87.315901],
     u'Alabama': [32.3182314, -86.902298],
     u'Albany, New Jersey': [42.622791, -73.73069029999999],
     u'Albuquerque NM': [35.0853336, -106.6055534],
     u'Alexandria, VA': [38.8048355, -77.0469214],
     u'Amsterdam & Eindhoven': [51.44164199999999, 5.4697225],
     u'Amsterdam, Netherlands': [52.3702157, 4.895167900000001],
     u'Amsterdam, the Netherlands ': [52.3702157, 4.895167900000001],
     u'Anna Maria, FL': [27.5311474, -82.7334334],
     u'Argentina': [-38.416097, -63.61667199999999],
     u'Arizona': [34.0489281, -111.0937311],
     u'Arnhem, The Netherlands': [51.9851034, 5.898729599999999],
     u'Athens, Thessaloniki': [37.9714263, 23.7108959],
     u'Atherton, California': [37.4613272, -122.197743],
     u'Atlanta, GA': [33.7489954, -84.3879824],
     u'Atlanta, GA, USA': [33.7489954, -84.3879824],
     u'Austin TX': [30.267153, -97.7430608],
     u'Austin, TX': [30.267153, -97.7430608],
     u'Australia': [-25.274398, 133.775136],
     u'BELGIUM': [50.503887, 4.469936],
     u'BERLIN': [52.52000659999999, 13.404954],
     u'Bangalore': [12.9715987, 77.5945627],
     u'Bangalore, India': [12.9715987, 77.5945627],
     u'Bangalore, India.': [12.9715987, 77.5945627],
     u'Bangkok, Thailand': [13.7563309, 100.5017651],
     u'Barcellona, Catalogna': [41.3850639, 2.1734035],
     u'Barcelona, Catalonia': [41.3850639, 2.1734035],
     u'Barcelona, Catalunya': [41.3850639, 2.1734035],
     u'Barcelona, Spain': [41.3850639, 2.1734035],
     u'Barrie, Ontario': [44.38935559999999, -79.6903316],
     u'Basel': [47.5595986, 7.5885761],
     u'Bay Area': [37.8271784, -122.2913078],
     u'Bayern': [48.7904472, 11.4978895],
     u'Belgium': [50.503887, 4.469936],
     u'Bengaluru, Karnataka': [12.9715987, 77.5945627],
     u'Benoni, South Africa': [-26.1510854, 28.3695699],
     u'Benztown Stuttgart': [48.7758459, 9.1829321],
     u'Berkeley, CA': [37.8715926, -122.272747],
     u'Berlin': [52.52000659999999, 13.404954],
     u'Berlin ': [52.52000659999999, 13.404954],
     u'Berlin & Stuttgart': [48.7758459, 9.1829321],
     u'Berlin - Potsdam': [52.3905689, 13.0644729],
     u'Berlin I Kiel': [54.3232927, 10.1227652],
     u'Berlin, Berlin DE': [52.52000659999999, 13.404954],
     u'Berlin, Deutschland': [52.52000659999999, 13.404954],
     u'Berlin, Germany': [52.52000659999999, 13.404954],
     u'Billericay, England': [51.627903, 0.418397],
     u'Birmingham, UK': [52.48624299999999, -1.890401],
     u'Blackheath, London SE3, UK': [51.466677, 0.0035],
     u'Blackwood, South Wales': [51.668935, -3.195787],
     u'Blue Springs, Mo': [39.0169509, -94.2816148],
     u'Bookham': [51.2783826, -0.3733532],
     u'Bordeaux, Aquitaine': [44.837789, -0.57918],
     u'Boston Austin SanFrancisco USA': [42.3600825, -71.0588801],
     u'Boston, MA': [42.3600825, -71.0588801],
     u'Boulder, CO': [40.0149856, -105.2705456],
     u'Brasil': [-14.235004, -51.92528],
     u'Brazil': [-14.235004, -51.92528],
     u'Brighton': [50.82253000000001, -0.137163],
     u'Brighton ': [50.82253000000001, -0.137163],
     u'Brighton & Hove': [50.82534520000001, -0.1551347],
     u'Brighton & Hove ': [50.82534520000001, -0.1551347],
     u'Brighton & Hove, UK': [50.82534520000001, -0.1551347],
     u'Brighton & London, UK': [51.4956075, -0.08309599999999999],
     u'Brighton UK': [50.82253000000001, -0.137163],
     u'Brighton and Hove': [50.82253000000001, -0.137163],
     u'Brighton and Hove, UK': [50.82253000000001, -0.137163],
     u'Brighton, East Sussex, England': [50.8663889, -0.1147526],
     u'Brighton, East Sussex, UK': [50.8663889, -0.1147526],
     u'Brighton, England': [50.82253000000001, -0.137163],
     u'Brighton, England.': [50.82253000000001, -0.137163],
     u'Brighton, UK': [50.82253000000001, -0.137163],
     u'Brighton, UK ': [50.82253000000001, -0.137163],
     u'Brighton, UK.': [50.82253000000001, -0.137163],
     u'Brighton, United Kingdom': [50.82253000000001, -0.137163],
     u'Brisbane': [-27.4710107, 153.0234489],
     u'Brisbane Australia': [-27.4710107, 153.0234489],
     u'Brisbane, Queensland': [-27.4710107, 153.0234489],
     u'British Columbia, Canada': [53.7266683, -127.6476206],
     u'Brixton': [51.4612794, -0.1156148],
     u'Brooklyn, NY': [40.6781784, -73.9441579],
     u'Brooklyn, New York': [40.6781784, -73.9441579],
     u'Brooklyn, USA': [40.6781784, -73.9441579],
     u'Brooklynite via Miami': [25.7616798, -80.1917902],
     u'Brussels': [50.8503396, 4.3517103],
     u'Brussels & Prague': [48.8549431, 2.7622488],
     u'Bruxelles': [50.8503396, 4.3517103],
     u'Budapest': [47.497912, 19.040235],
     u'Buffalo': [42.88644679999999, -78.8783689],
     u'Bussy Saint Geoges (77600)': [48.842205, 2.69802],
     u'CA': [36.778261, -119.4179324],
     u'CITY-ZEN': [14.6138146, 121.1069995],
     u'Calgary, AB': [51.0486151, -114.0708459],
     u'Calgary, Alberta, Canada': [51.0486151, -114.0708459],
     u'Cali, Valle del Cauca': [3.4516467, -76.5319854],
     u'California': [36.778261, -119.4179324],
     u'California, USA': [36.778261, -119.4179324],
     u'Cambodia': [12.565679, 104.990963],
     u'Cambridge, England': [52.205337, 0.121817],
     u'Cambridge, MA': [42.3736158, -71.10973349999999],
     u'Canada': [56.130366, -106.346771],
     u'Cape Breton, Nova Scotia': [46.2486851, -60.851817],
     u'Carbonia': [39.16442809999999, 8.522885300000002],
     u'Carson City, NV': [39.1637984, -119.7674034],
     u'Cernay, Alsace': [47.80712399999999, 7.176532000000001],
     u'Chamonix et Lausanne': [46.5196535, 6.6322734],
     u'Charlotte NC': [35.2270869, -80.8431267],
     u'Charlotte, NC': [35.2270869, -80.8431267],
     u'Chemnitz': [50.827845, 12.9213697],
     u'Chennai': [13.0826802, 80.2707184],
     u'Cheshire, UK': [53.203376, -2.927308],
     u'Chester - North West, England': [53.193392, -2.893075],
     u'Chester, England': [53.193392, -2.893075],
     u"Cheul'o Block 105": [47.799377, -3.050758],
     u'Chicago': [41.8781136, -87.6297982],
     u'Chicago + Highland Park, IL': [42.1960819, -87.806805],
     u'Chicago via Lansing': [41.573384, -87.54419399999999],
     u'Chicago, IL': [41.8781136, -87.6297982],
     u'Chichester, England': [50.83761000000001, -0.774936],
     u'Cholargos, Greece': [38.0026476, 23.7986048],
     u'Cincinnati, OH': [39.1031182, -84.5120196],
     u'Cleveland': [41.49932, -81.6943605],
     u'Cleveland, OH': [41.49932, -81.6943605],
     u'Clinton, Iowa 52732': [41.8444735, -90.1887379],
     u'Coimbatore, Bangalore, Chennai': [13.0826802, 80.2707184],
     u'Cologne': [50.937531, 6.9602786],
     u'Colombia': [4.570868, -74.297333],
     u'Columbus, OH': [39.9611755, -82.99879419999999],
     u'Coulsdon, England': [51.319941, -0.141267],
     u'Covington, LA': [30.4754702, -90.1009108],
     u'Curitiba - PR': [-25.4289541, -49.267137],
     u'Dallas TX': [32.7766642, -96.79698789999999],
     u'Dallas, TX': [32.7766642, -96.79698789999999],
     u'Dallas, TX - Global ': [32.7435217, -96.72529589999999],
     u'Dallas, TX USA': [32.7766642, -96.79698789999999],
     u'Dallas, Texas': [32.7766642, -96.79698789999999],
     u'Damascus - Syria': [33.5138073, 36.2765279],
     u'Daresbury Science Park, UK': [53.344017, -2.635447],
     u'Denton, Texas': [33.2148412, -97.13306829999999],
     u'Denver Area': [14.3395086, 120.9330599],
     u'Denver, CO': [39.7392358, -104.990251],
     u'Derby UK': [52.9225301, -1.4746186],
     u'Detroit, MI': [42.331427, -83.0457538],
     u'Deutschland': [51.165691, 10.451526],
     u'Dhaka, Bangladesh': [23.810332, 90.4125181],
     u'Dortmund, Deutschland': [51.5135872, 7.465298100000001],
     u'Dubai': [25.2048493, 55.2707828],
     u'Dubai, UAE': [25.2048493, 55.2707828],
     u'Dubai, United Arab Emirates': [25.2048493, 55.2707828],
     u'Dublin City, Ireland': [53.3498053, -6.2603097],
     u'Dublin, Copenhagen': [55.64074309999999, 12.6033037],
     u'Dublin, Ireland': [53.3498053, -6.2603097],
     u'Duesseldorf, Germany': [51.2277411, 6.7734556],
     u'Durban, South Africa': [-29.85868039999999, 31.0218404],
     u'EDGWARE NW LONDON': [51.615786, -0.262034],
     u'EDGWARE. NW LONDON': [51.615786, -0.262034],
     u'East Coast, Virginia': [51.7780813, -8.3031706],
     u'East Midlands, England': [52.82937399999999, -1.332134],
     u'East Nashville': [36.1860795, -86.7315785],
     u'East Sussex': [50.908596, 0.2494235],
     u'East Sussex, UK.': [50.908596, 0.2494235],
     u'Eastvale, CA': [33.952463, -117.5848025],
     u'Edinburgh': [55.953252, -3.188267],
     u'Elche': [38.2699329, -0.7125608],
     u'England': [52.3555177, -1.1743197],
     u'England ': [52.3555177, -1.1743197],
     u'England, United Kingdom': [52.3555177, -1.1743197],
     u'Essen': [51.4556432, 7.0115552],
     u'Essen-Steele': [51.44856480000001, 7.0792857],
     u'Eugene, OR': [44.0520691, -123.0867536],
     u'Europa': [54.5259614, 15.2551187],
     u'Europe': [54.5259614, 15.2551187],
     u'Farnham, UK': [51.214321, -0.798802],
     u'Finland': [61.92410999999999, 25.748151],
     u'Flushing, NY': [40.7674987, -73.833079],
     u'Fort Collins, CO': [40.5852602, -105.084423],
     u'Fort Wayne, IN': [41.079273, -85.1393513],
     u'Fort Worth TX': [32.7554883, -97.3307658],
     u'Foster City, CA': [37.5585465, -122.2710788],
     u'Framingham, Mass': [42.279286, -71.4161565],
     u'France': [46.227638, 2.213749],
     u'France-Provence-Toulon': [43.1334953, 5.9662083],
     u'Frankfurt am Main': [50.1109221, 8.6821267],
     u'Frankfurt am Main, Hessen': [50.1109221, 8.6821267],
     u'Frankfurt, Germany': [50.1109221, 8.6821267],
     u'Freiburg': [47.9990077, 7.842104299999999],
     u'Freiburg i. Br., Germany': [48.8435527, 9.5452234],
     u'GREECE': [39.074208, 21.824312],
     u'Gauteng': [-26.2707593, 28.1122679],
     u'Geneva, Switzerland': [46.2043907, 6.1431577],
     u'Germany': [51.165691, 10.451526],
     u'Germany ': [51.165691, 10.451526],
     u'Germany, Meerbusch': [51.2503512, 6.690717299999999],
     u'Germany, baby!': [49.075131, 14.452735],
     u'Germany, near Hamburg': [53.5510846, 9.9936818],
     u'Germany; Stuttgart': [48.7758459, 9.1829321],
     u'Glasgow': [55.864237, -4.251806],
     u'Glasgow, Scotland': [55.864237, -4.251806],
     u'Global': [31.9524799, -102.1732882],
     u'Globe Trotter': [46.2149266, -73.4546866],
     u'Goodyear, AZ': [33.4353394, -112.3576567],
     u'Gotham ': [40.7127837, -74.0059413],
     u'Grand Rapids, MI': [42.9633599, -85.6680863],
     u'Graz, Austria': [47.070714, 15.439504],
     u'Greater Austin Area': [-33.9565129, 151.0729093],
     u'Greater Seattle Area': [47.606139, -122.342528],
     u'Greater Vancouver, British Columbia': [49.2501243, -123.0824217],
     u'Greece': [39.074208, 21.824312],
     u'Greensboro NC': [36.0726354, -79.7919754],
     u'Grenoble': [45.188529, 5.724524],
     u'Guatemala': [15.783471, -90.23075899999999],
     u'Gurgaon, Haryana': [28.4594965, 77.0266383],
     u'Gutenbergstr. 77A, Stuttgart': [48.77071730000001, 9.157009],
     u'Habitando en Madrid': [40.4167754, -3.7037902],
     u'Halifax, Nova Scotia': [44.6488625, -63.5753196],
     u'Hamburg': [53.5510846, 9.9936818],
     u'Hamburg ': [53.5510846, 9.9936818],
     u'Hamburg, Deutschland': [53.5510846, 9.9936818],
     u'Hamburg, Germany': [53.5510846, 9.9936818],
     u'Harrisburg, PA': [40.2731911, -76.8867008],
     u'Hasselroth': [50.1631979, 9.095912199999999],
     u'Havana, Cuba': [23.1135925, -82.3665956],
     u'High Wycombe ': [51.628611, -0.7482289999999999],
     u'Highlands Ranch, CO': [39.5480789, -104.9739333],
     u'Houston, Texas': [29.7604267, -95.3698028],
     u'Houston, Texas ': [29.7604267, -95.3698028],
     u'Hove': [50.8279319, -0.168749],
     u'Hove, England': [50.8279319, -0.168749],
     u'Huddersfield': [53.645792, -1.785035],
     u'Huelva': [37.261421, -6.9447224],
     u'Hyattsville, MD': [38.9559442, -76.9455301],
     u'Hyderabad India': [17.385044, 78.486671],
     u'IND': [40.2671941, -86.1349019],
     u'INDIA': [20.593684, 78.96288],
     u'Iligan City': [8.228020899999999, 124.2452422],
     u'In Poland & CEE since 1995': [51.919438, 19.145136],
     u'India': [20.593684, 78.96288],
     u'Indianapolis, IN': [39.768403, -86.158068],
     u'Indonesia': [-0.789275, 113.921327],
     u'Inkster, Michigan; virtual USA': [42.2942045, -83.30993029999999],
     u'Israel': [31.046051, 34.851612],
     u'Istanbul, Turkey': [41.0082376, 28.9783589],
     u'JAKARTA': [-6.2087634, 106.845599],
     u'Jamaica, KN': [42.3392014, -71.09085189999999],
     u'Johannesburg, South Africa': [-26.2041028, 28.0473051],
     u'KL': [3.139003, 101.686855],
     u'KL ': [3.139003, 101.686855],
     u'Kansas City': [39.0997265, -94.5785667],
     u'Kansas City, MO': [39.0997265, -94.5785667],
     u'Keller, TX': [32.9341893, -97.229298],
     u'Kenner,La': [29.9940924, -90.2417434],
     u'Kisii, Kenya': [-0.677334, 34.779603],
     u'Kita, Tokyo': [35.7528042, 139.7334805],
     u'Kota Palembang - NgayogyaKarta': [-2.9760735, 104.7754307],
     u'LOUISVILLE': [38.2526647, -85.7584557],
     u'Lahore, Pakistan': [31.55460609999999, 74.3571581],
     u'Lake Geneva, WI': [42.5916836, -88.4334301],
     u'Lake Geneva, Wi': [42.5916836, -88.4334301],
     u'Langenfeld (Rheinland)': [51.1191898, 6.951266299999999],
     u'Las Vegas, NV': [36.1699412, -115.1398296],
     u'Leatherhead': [51.29640699999999, -0.33112],
     u'Leeds': [53.8007554, -1.5490774],
     u'Leeds and London': [53.7730148, -1.2496171],
     u'Levittown, PA, USA': [40.1551096, -74.8287747],
     u'Lewes, England': [50.87387200000001, 0.00878],
     u'Lille': [50.62925, 3.057256],
     u'Limassol, Cyprus': [34.7071301, 33.0226174],
     u'Lisboa, Portugal': [38.7222524, -9.1393366],
     u'Lisbon, Portugal': [38.7222524, -9.1393366],
     u'Little Rock, AR': [34.7464809, -92.28959479999999],
     u'LittleHouseOnThePrairie': [53.4350341, -6.6019025],
     u'Littoral, Cameroon': [4.1682138, 10.0807298],
     u'Liverpool': [53.4083714, -2.9915726],
     u'London': [51.5073509, -0.1277583],
     u'London ': [51.5073509, -0.1277583],
     u'London UK': [51.5073509, -0.1277583],
     u'London UK ': [51.5073509, -0.1277583],
     u'London and Kent': [51.5073509, -0.1277583],
     u'London, Brighton & Hove, UK': [51.5797281, -0.0279909],
     u'London, England': [51.5073509, -0.1277583],
     u'London, Leeds, Grasse.': [51.5073509, -0.1277583],
     u'London, Ontario': [42.9869502, -81.243177],
     u'London, UK': [51.5073509, -0.1277583],
     u'London, UK.': [51.5073509, -0.1277583],
     u'London, United Kingdom': [51.5073509, -0.1277583],
     u'Los Angeles': [34.0522342, -118.2436849],
     u'Los Angeles ': [34.0522342, -118.2436849],
     u'Los Angeles, CA': [34.0522342, -118.2436849],
     u'Los Angeles, California': [34.0522342, -118.2436849],
     u'Los Angels, California .USA': [34.0522342, -118.2436849],
     u'Louisville, KY': [38.2526647, -85.7584557],
     u'Lucknow, Uttar Pradesh': [26.8466937, 80.94616599999999],
     u'Luzern': [47.05016819999999, 8.3093072],
     u'Lyon': [45.764043, 4.835659],
     u'MA': [42.4072107, -71.3824374],
     u'MARSEILLE - France': [43.296482, 5.36978],
     u'Made in Iran': [37.4033333, 48.7863889],
     u'Madison, WI': [43.0730517, -89.4012302],
     u'Madrid': [40.4167754, -3.7037902],
     u'Madrid (Spain)': [40.4167754, -3.7037902],
     u'Madrid, Spain': [40.4167754, -3.7037902],
     u'Maine': [45.253783, -69.4454689],
     u'Makuhari Messe, Tokyo, Japan': [35.7090259, 139.7319925],
     u'Malaysia': [4.210484, 101.975766],
     u'Manchester': [53.4807593, -2.2426305],
     u'Manchester ': [53.4807593, -2.2426305],
     u'Manchester, UK': [53.4807593, -2.2426305],
     u'Manhattan, NY': [40.7830603, -73.9712488],
     u'Manhattan, New York': [40.7830603, -73.9712488],
     u'Manila, Philippines': [14.5995124, 120.9842195],
     u'Mannheim, Germany, Europe': [49.4874592, 8.466039499999999],
     u'Mantes-La-Jolie (France)': [48.989323, 1.714958],
     u'Marseille': [43.296482, 5.36978],
     u'Marseille et France': [43.997343, 0.102542],
     u'Marshfield, Wisconsin': [44.6688524, -90.1717987],
     u'Massachusetts': [42.4072107, -71.3824374],
     u'McKinney Texas ': [33.1972465, -96.6397822],
     u'Memphis, TN': [35.1495343, -90.0489801],
     u'Metropolregion Hamburg': [53.5510846, 9.9936818],
     u'Mexico': [23.634501, -102.552784],
     u'Mexico City': [19.4326077, -99.133208],
     u'Miami': [25.7616798, -80.1917902],
     u'Miami Beach Florida': [25.790654, -80.1300455],
     u'Miami, FL': [25.7616798, -80.1917902],
     u'Minneapolis': [44.977753, -93.2650108],
     u'Minnesota': [46.729553, -94.6858998],
     u'Monmouth, Wales': [51.8116533, -2.7163045],
     u'Montreal, Quebec, Canada': [45.5016889, -73.567256],
     u'Much, Nordrhein-Westfalen': [50.90397309999999, 7.4022887],
     u'Mumbai': [19.0759837, 72.8776559],
     u'Mumbai - India': [19.0759837, 72.8776559],
     u'Munich': [48.1351253, 11.5819806],
     u'Munich & Hamburg': [25.9590554, -97.5332486],
     u'Munich, Germany': [48.1351253, 11.5819806],
     u'NYC': [40.7127837, -74.0059413],
     u'Nantes': [47.218371, -1.553621],
     u'Naples, Italy': [40.8517746, 14.2681244],
     u'New Delhi (India)': [28.6139391, 77.2090212],
     u'New Jersey': [40.0583238, -74.4056612],
     u'New Jersey, USA': [40.0583238, -74.4056612],
     u'New York': [40.7127837, -74.0059413],
     u'New York .': [40.7127837, -74.0059413],
     u'New York City': [40.7127837, -74.0059413],
     u'New York City or a plane': [40.7127837, -74.0059413],
     u'New York NY & Lexington KY': [40.7127837, -74.0059413],
     u'New York sometimes': [40.7127837, -74.0059413],
     u'New York, NY': [40.7127837, -74.0059413],
     u'New York, New York': [40.7127837, -74.0059413],
     u'New York, USA': [40.7127837, -74.0059413],
     u'New York, United States': [40.7127837, -74.0059413],
     u'NewYork ': [40.7127837, -74.0059413],
     u'Newark, Delaware': [39.6837226, -75.74965720000002],
     u'Newark, NJ': [40.735657, -74.1723667],
     u'Newbury': [51.401409, -1.3231139],
     u'Newton, MA': [42.3370413, -71.20922139999999],
     u'Niagara Region, Canada': [49.106313, -118.466454],
     u'Nigeria': [9.081999, 8.675277],
     u'Nivelles, Belgium': [50.59787, 4.32851],
     u'Noida, Uttar Pradesh': [28.5355161, 77.3910265],
     u'Normal, IL': [40.5142026, -88.9906312],
     u'North Carolina, USA': [35.7595731, -79.01929969999999],
     u'Northern California ': [38.8375215, -120.8958242],
     u'Norwich, UK': [52.6308859, 1.297355],
     u'Nottingham': [52.95478319999999, -1.1581086],
     u'Oberderdingen': [49.0643457, 8.8014843],
     u'Ohio, USA': [40.4172871, -82.90712300000001],
     u'Oklahoma': [35.0077519, -97.092877],
     u'Oldenburg in Oldenburg': [54.2957397, 10.8919582],
     u'Oldsmar, FL': [28.0341847, -82.6650992],
     u'Omaha, NE': [41.2523634, -95.99798829999999],
     u'Orlando': [28.5383355, -81.3792365],
     u'Orlando, FL': [28.5383355, -81.3792365],
     u'Oxford': [51.7520209, -1.2577263],
     u'Oxford, England': [51.7520209, -1.2577263],
     u'Oxfordshire, England': [51.7612056, -1.2464674],
     u'Pakistan': [30.375321, 69.34511599999999],
     u'Palo Alto': [37.4418834, -122.1430195],
     u'Palo Alto, CA': [37.4418834, -122.1430195],
     u'Pandora': [40.9481078, -83.9610538],
     u'Paris': [48.856614, 2.3522219],
     u'Paris & San Francisco & Berlin': [22.1409279, -100.9345312],
     u'Paris 13': [48.8278352, 2.2935372],
     u'Paris 75 (France) U.E': [48.856614, 2.3522219],
     u'Paris, France': [48.856614, 2.3522219],
     u'Paris, Ile-de-France': [48.856614, 2.3522219],
     u'Philadelphia, PA': [39.9525839, -75.1652215],
     u'Philly, PA': [39.9525839, -75.1652215],
     u'Pittsburgh, Pa': [40.44062479999999, -79.9958864],
     u'Portland Oregon USA': [45.5230622, -122.6764816],
     u'Portland, OR': [45.5230622, -122.6764816],
     u'Portland, Oregon, USA': [45.5230622, -122.6764816],
     u'Pozo-Lorente, Spain 02154': [39.076351, -1.5069455],
     u'Princeton, NJ': [40.3572976, -74.6672226],
     u'Providence RI': [41.8239891, -71.4128343],
     u'Providence and Boston': [42.2637706, -71.1229378],
     u'Pune, India': [18.5204303, 73.8567437],
     u'Pune, Maharashtra': [18.5204303, 73.8567437],
     u'Punjab & H.P. India': [31.1596072, 76.15402569999999],
     u'Quito': [-0.1806532, -78.4678382],
     u'Quito, Ecuador': [-0.1806532, -78.4678382],
     u'Radolfzell, Germany': [47.745237, 8.9669098],
     u'Raleigh, North Carolina': [35.7795897, -78.6381787],
     u'Reading & London, UK': [51.454344, -0.9435855999999999],
     u'Reading, UK': [51.4542645, -0.9781303],
     u'Republic of Derry': [39.5000849, -74.4993396],
     u'River Forest, IL': [41.8978091, -87.81394829999999],
     u'Rome, Salerno, London, Lisbon': [38.7222524, -9.1393366],
     u'Rotterdam': [51.9244201, 4.4777325],
     u'Rotterdam or at home in Breda': [51.5719149, 4.768323],
     u'Rotterdam, Netherlands': [51.9244201, 4.4777325],
     u'Rouen': [49.44323199999999, 1.099971],
     u'Ruhrgebiet ': [51.3659097, 7.6282792],
     u'Runcorn': [53.342078, -2.729673],
     u'STL': [40.453868, -105.0629797],
     u'SUISSE': [46.818188, 8.227511999999999],
     u'Sacramento, CA': [38.5815719, -121.4943996],
     u'Sale, Manchester': [53.42556099999999, -2.323702],
     u'Salerno': [40.68244079999999, 14.7680961],
     u'Salt Lake City, Utah': [40.7607793, -111.8910474],
     u'San Diego': [32.715738, -117.1610838],
     u'San Diego, California ': [32.715738, -117.1610838],
     u'San Francisco': [37.7749295, -122.4194155],
     u'San Francisco ': [37.7749295, -122.4194155],
     u'San Francisco | New York': [14.2491625, 120.9009946],
     u'San Francisco, CA': [37.7749295, -122.4194155],
     u'San Jose, Costa Rica': [9.9280694, -84.0907246],
     u'San Luis Obispo, Ca': [35.2827524, -120.6596156],
     u'Santiago, Chile': [-33.4488897, -70.6692655],
     u'Sao Paulo, Brazil': [-23.5505199, -46.63330939999999],
     u'Schweiz': [46.818188, 8.227511999999999],
     u'Seaford, East Sussex, England': [50.7734669, 0.101108],
     u'Seattle': [47.6062095, -122.3320708],
     u'Seattle | The Farm': [42.3605071, -71.1252543],
     u'Seattle, WA': [47.6062095, -122.3320708],
     u'Seattle. Before that, London.': [51.5073509, -0.1277583],
     u'Sevilla': [37.3890924, -5.9844589],
     u'Sheffield': [53.38112899999999, -1.470085],
     u'Sheffield, South Yorkshire, UK': [53.38112899999999, -1.470085],
     u'Singapore': [1.352083, 103.819836],
     u'Sitges': [41.2371851, 1.805886],
     u'Slough, England': [51.51053839999999, -0.5950405999999999],
     u'Solothurn, Switzerland': [47.2088348, 7.532291],
     u'Somewhere near Brighton': [50.82253000000001, -0.137163],
     u'South Africa': [-30.559482, 22.937506],
     u'South Africa - Johannesburg': [-26.2041028, 28.0473051],
     u'South East': [-24.9366094, 25.8048521],
     u'South West, England': [52.3555177, -1.1743197],
     u'Southampton': [50.90970040000001, -1.4043509],
     u'Southern California': [34.9592083, -116.419389],
     u'Spain': [40.46366700000001, -3.74922],
     u'St Louis, MO': [38.6270025, -90.19940419999999],
     u'St Paul, Mn': [44.9537029, -93.0899578],
     u'St. Louis, MO': [38.6270025, -90.19940419999999],
     u'Stafford': [52.806693, -2.12066],
     u'Stafford, VA': [38.4220687, -77.4083086],
     u'Strasbourg': [48.5734053, 7.752111299999999],
     u'Strasbourg, Alsace': [48.5734053, 7.752111299999999],
     u'Straubing': [48.8777333, 12.5801538],
     u'Stuttgart': [48.7758459, 9.1829321],
     u'Stuttgart, Germany': [48.7758459, 9.1829321],
     u'Sunbury-On-Thames': [51.4191935, -0.4191503],
     u'Surbiton & Brighton': [50.82253000000001, -0.137163],
     u'Surigao City, Philippines': [9.7571312, 125.5137674],
     u'Surrey, UK': [51.31475930000001, -0.5599501],
     u'Surrey,UK': [51.31475930000001, -0.5599501],
     u'Sussex': [43.1338972, -88.2220372],
     u'Sussex, England': [50.8936205, -0.2068074],
     u'Swiss': [46.818188, 8.227511999999999],
     u'Switzerland': [46.818188, 8.227511999999999],
     u'Sydney, Australia': [-33.8674869, 151.2069902],
     u'Sydney, New South Wales': [-33.8674869, 151.2069902],
     u'Syracuse, NY': [43.0481221, -76.14742439999999],
     u'Tampa Bay, Florida': [27.763383, -82.5436722],
     u'Tampa, FL': [27.950575, -82.4571776],
     u'Tanger': [35.7594651, -5.833954299999999],
     u'Tel Aviv - Israel': [32.0852999, 34.78176759999999],
     u'The Cloud, CT, NY, IL': [40.6331249, -89.3985283],
     u'The Netherlands': [52.132633, 5.291265999999999],
     u'The Pelican State': [30.9842977, -91.96233269999999],
     u'The wilds of Mid Sussex': [51.015496, -0.031553],
     u'Tirana,Albania ': [41.3275459, 19.8186982],
     u'Toronto': [43.653226, -79.3831843],
     u'Toronto  Canada': [43.653226, -79.3831843],
     u'Toronto, Canada': [43.653226, -79.3831843],
     u'Toulouse': [43.604652, 1.444209],
     u'Toulouse Haute-Garonne France': [43.604652, 1.444209],
     u'Toulouse, France': [43.604652, 1.444209],
     u'Tours ': [47.394144, 0.68484],
     u'Tri-Cities, WA': [46.2349999, -119.2233014],
     u'Trujillo - Peru.': [-8.1090524, -79.0215336],
     u'Tunbridge Wells, UK': [51.132377, 0.263695],
     u'U.S. & Europe': [41.10818949999999, 28.86700609999999],
     u'UK': [55.378051, -3.435973],
     u'UK ': [55.378051, -3.435973],
     u'UK & Saudi Arabia, Jeddah': [21.2854067, 39.2375507],
     u'USA': [37.09024, -95.712891],
     u'USA ': [37.09024, -95.712891],
     u'USA: NJ LA CHINA JAPAN VIETNAM': [40.0583238, -74.4056612],
     u'Ubiquity + NYC': [40.7127837, -74.0059413],
     u'Ukiah & San Francisco Bay Area': [37.8271784, -122.2913078],
     u'United Kingdom': [55.378051, -3.435973],
     u'United Kingdom,united states ': [30.1786603, -97.8084318],
     u'United States': [37.09024, -95.712891],
     u'United States of America': [37.09024, -95.712891],
     u'Utah': [39.3209801, -111.0937311],
     u'Vancouver': [45.6387281, -122.6614861],
     u'Vancouver | New York | Kenya': [45.6387281, -122.6614861],
     u'Vancouver, BC': [49.2827291, -123.1207375],
     u'Vancouver, British Columbia': [49.2827291, -123.1207375],
     u'Vancouver, Canada': [49.2827291, -123.1207375],
     u'Vannes - Morbihan - Bretagne': [47.658236, -2.760847],
     u'Varel, Germany': [53.3961959, 8.1368242],
     u'Veenendaal, Netherlands': [52.0263009, 5.5544309],
     u'Venezuela': [6.42375, -66.58973],
     u'Venus': [32.4334742, -97.10250769999999],
     u'Victoria, BC': [48.4284207, -123.3656444],
     u'Victoria, BC, Canada': [48.4284207, -123.3656444],
     u'Vienna': [48.2081743, 16.3738189],
     u'Virginia Beach': [36.8529263, -75.97798499999999],
     u'Vleuten, Utrecht': [52.1092881, 5.0241355],
     u'Walla Walla Wine Country': [46.0645809, -118.3430209],
     u'Waltham, MA': [42.3764852, -71.2356113],
     u'Warstein, Deutschland': [51.4491749, 8.3557715],
     u'Washington': [38.9071923, -77.0368707],
     u'Washington DC': [38.9071923, -77.0368707],
     u'Washington, D.C': [38.9071923, -77.0368707],
     u'Washington, DC': [38.9071923, -77.0368707],
     u'Waupun Correctional Prison': [43.6333219, -88.72955189999999],
     u'West Sussex': [50.9280143, -0.4617075],
     u'Westerville, Ohio': [40.1261743, -82.92906959999999],
     u'Wien': [48.2081743, 16.3738189],
     u'Winnipeg, Canada': [49.8997541, -97.1374937],
     u'Wisconsin - Minnesota': [43.6280145, -94.9143239],
     u'Wisconsin, USA': [43.7844397, -88.7878678],
     u'Woodmont, CT': [41.228153, -72.9914925],
     u'Worldwide': [52.57083129999999, -0.3006158],
     u'Worthing, West Sussex': [50.81787, -0.372882],
     u'Wroclaw Baja Silesia - Polonia': [51.1078852, 17.0385376],
     u'Zapopan, Jalisco': [20.6719563, -103.416501],
     u'Zaragoza': [41.6488226, -0.8890853],
     u'Zimbabwe': [-19.015438, 29.154857],
     u'anderson ': [34.5034394, -82.6501332],
     u'brighton': [50.82253000000001, -0.137163],
     u'brighton | london': [51.4956075, -0.08309599999999999],
     u'london': [51.5073509, -0.1277583],
     u'matheus': [-39.1039403, -67.07811629999999],
     u'missouri USA': [37.9642529, -91.8318334],
     u'montreal': [45.5016889, -73.567256],
     u'mumbhi': [19.0759837, 72.8776559],
     u'nyc': [40.7127837, -74.0059413],
     u'oak lawn , illinois , usa': [41.719978, -87.7479528],
     u'paris': [48.856614, 2.3522219],
     u'sd x berkeley': [37.8950207, -122.2724044],
     u'singapore, london, 35,000 ft': [1.352083, 103.819836],
     u'small town Ohio': [40.4172871, -82.90712300000001],
     u'stark tower, new york.': [40.7127837, -74.0059413],
     u'tennessee': [35.5174913, -86.5804473],
     u'texas': [31.9685988, -99.9018131],
     u'third world country': [40.82438500000001, -73.810236],
     u'toronto via nyc from sydney': [-33.8674869, 151.2069902],
     u'~ Shanghai': [31.230416, 121.473701]}



that we also turn into the following DataFrame:


```python
lats, lons, source = [], [], []
for loc in locLoc.keys():
    lats.append(locLoc[loc][0])
    lons.append(locLoc[loc][1])
    if loc in np.array(pdA['location']) and loc in np.array(pdM['location']):
        source.append("Both")
    elif loc in np.array(pdA['location']):
        source.append("Employee")
    else:
        source.append("Mention")
        
CoordLoc = pd.DataFrame({'Latitude': lats, 'Longitude': lons, 'Source': source})
CoordLoc.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Latitude</th>
      <th>Longitude</th>
      <th>Source</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>56.130366</td>
      <td>-106.346771</td>
      <td>Mention</td>
    </tr>
    <tr>
      <th>1</th>
      <td>44.648862</td>
      <td>-63.575320</td>
      <td>Mention</td>
    </tr>
    <tr>
      <th>2</th>
      <td>34.503439</td>
      <td>-82.650133</td>
      <td>Mention</td>
    </tr>
    <tr>
      <th>3</th>
      <td>47.559599</td>
      <td>7.588576</td>
      <td>Mention</td>
    </tr>
    <tr>
      <th>4</th>
      <td>52.829374</td>
      <td>-1.332134</td>
      <td>Mention</td>
    </tr>
  </tbody>
</table>
</div>



We use the package 'Basemap' to picture the worlwide spread of users in our dataset.


```python
import numpy as np
import matplotlib.pyplot as plt
from mpl_toolkits.basemap import Basemap
%matplotlib inline

plt.rcParams['figure.figsize'] = 30, 15
id_color = {'Employee': "g", "Mention": "m", "Both": "r"}

### we define coordinate coorners to hide Antartica
m = Basemap(projection='merc', resolution = 'l', area_thresh = 1000, lat_0=0, lon_0=0,\
            llcrnrlat=-60, llcrnrlon=-140, urcrnrlat=75, urcrnrlon=180)

m.drawcoastlines()
m.drawcountries()
m.fillcontinents(color = 'gainsboro', lake_color='steelblue')
m.drawmapboundary(fill_color='steelblue')

        
for idx, row in CoordLoc.iterrows():
    x, y = row[['Longitude','Latitude']]
    xcoord, ycoord = m([x], [y])
    source = row['Source']
    color = id_color[source]
    m.plot(xcoord, ycoord, 'o', markersize=10, zorder=6, markerfacecolor=color, 
                alpha=0.75, markeredgecolor="none")
markers = [plt.Line2D([0,0],[0,0],color=color, marker='o', markersize=15, linestyle='')\
                   for color in id_color.values()]
plt.legend(markers, id_color.keys(), numpoints=1, fontsize=20, markerscale=0.75)
plt.title("Users' Worldwide distribution", fontsize=30)
plt.show()
```


![png](output_33_0.png)


We observe many dots in both Europe and the US. The green and red colors here represent people working for Brandwatch, and we spot the offices in Brighton and Stuttgart (New-York is barely visible, surrounded by many magenta dots and Chicago presumably hidden the same way). There are other red and green dots in Houston, Scotland and India that don't seem to fit. Probably employees who gave personal locations or tweeted while traveling.

At any rate, regardless of whether every point is to be trusted or not there is evidence for a large distribution in the whole world. There is also a fair amount of observations in India around South Africa and Indonesia, which shouldn't be too surprising given the interest and large number of IT professionals in general and Data Science in particular who live in or simply come from these countries.

###  Observations count and Europe's focus

Let us now keep track of how many times we get an observation for a given location. This time we use the more reliable 'twitterTimeZone' feature to find the locations. But first, we need to use the GoogleMaps API once more.


```python
### Use Google Maps API to retrieve coordinates associated to the entries of the field
### twitterTimeZone
import googlemaps
from googlemaps import client, convert

gmaps = client.Client(key="AIzaSyAQfwuO_dZx2kam_uK4xq439rEYhkNgwm8")

### Form a dataframe totalizing all the observations
pdTot = pd.concat([pdA, pdM], axis=0).reset_index()
pdTot = pdTot.iloc[:, 1:]

typeT, boundsT, locT = {}, {}, {}
for loc in np.unique(pdTot['twitterTimeZone']):
    if loc:
        dloc = gmaps.geocode(loc)
        locT[loc] = [dloc[0]['geometry']['location']['lat'],\
                          dloc[0]['geometry']['location']['lng']]
        if 'types' in dloc[0].keys() and len(dloc[0]['types']) > 0:
            typeT[loc] = dloc[0]['types'][0]
        else:
            typeT[loc] = 'unknown'
        if 'bounds' in dloc[0].keys():
            boundsT[loc] = [[dloc[0]['geometry']['bounds']['northeast']['lat'],\
                             dloc[0]['geometry']['bounds']['northeast']['lng']],\
                            [dloc[0]['geometry']['bounds']['southwest']['lat'],\
                             dloc[0]['geometry']['bounds']['southwest']['lng']]]
        else:
            boundsT[loc] = [[], []]
    else:
        locT[loc] = []
        typeT[loc] = 'unknown'
        boundsT[loc] = [[], []]
```

Again, we store the results in a DataFrame:


```python
lats, lons = [], []
for loc in pdTot['twitterTimeZone']:
    if locT[loc] != []:
        lats.append(locT[loc][0])
        lons.append(locT[loc][1])
        
Coord = [tuple([lat, lon]) for lat, lon in zip(lats, lons)]
NbTweet = {}
for coord in Coord:
    if not coord in NbTweet.keys():
        NbTweet[coord] = 1
    else:
        NbTweet[coord] += 1
CoordT = pd.DataFrame()
ct = 0
for coord in NbTweet.keys():
    Name = [nam for nam in locT.keys() if locT[nam] == [coord[0], coord[1]]]
    nName = [nam for nam in Name if "/" not in nam]
    if len(nName) > 0:
        name = nName[0]
    else:
        name = Name[0]
    CoordT = pd.concat([CoordT, pd.DataFrame({'Latitude': coord[0], 'Longitude': coord[1],
                                             'Count': NbTweet[coord],
                                              'Name': name}, index=[ct])], axis=0)
    ct += 1
CoordT.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Count</th>
      <th>Latitude</th>
      <th>Longitude</th>
      <th>Name</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>7</td>
      <td>-33.867487</td>
      <td>151.206990</td>
      <td>Sydney</td>
    </tr>
    <tr>
      <th>1</th>
      <td>25</td>
      <td>46.947974</td>
      <td>7.447447</td>
      <td>Bern</td>
    </tr>
    <tr>
      <th>2</th>
      <td>9</td>
      <td>41.902783</td>
      <td>12.496365</td>
      <td>Rome</td>
    </tr>
    <tr>
      <th>3</th>
      <td>44</td>
      <td>33.573110</td>
      <td>-7.589843</td>
      <td>Casablanca</td>
    </tr>
    <tr>
      <th>4</th>
      <td>4</td>
      <td>-6.208763</td>
      <td>106.845599</td>
      <td>Jakarta</td>
    </tr>
  </tbody>
</table>
</div>




```python
import numpy as np
import matplotlib.pyplot as plt
from mpl_toolkits.basemap import Basemap
%matplotlib inline

plt.rcParams['figure.figsize'] = 20, 10
id_color = {'Employee': "g", "Mentioning": "m", "Both": "r"}


m = Basemap(projection='robin', resolution = 'l', area_thresh = 1000.0,
              lat_0=0, lon_0=0)

m.drawcoastlines()
m.drawcountries()
m.fillcontinents(color = 'gainsboro', lake_color='steelblue')
m.drawmapboundary(fill_color='steelblue')

locA = [pdA.iloc[i, :]['twitterTimeZone'] for i in range(len(pdA.index))]
locM = [pdM.iloc[i, :]['twitterTimeZone'] for i in range(len(pdM.index))]
        
for idx, row in CoordT.iterrows():
    x, y = row[['Longitude','Latitude']]
    xcoord, ycoord = m([x], [y])
    loc = row['Name']
    if loc in locA and loc in locM:
        color = id_color["Both"]
    elif loc in locA:
        color = id_color["Employee"]
    else:
        color = id_color["Mentioning"]
    m.plot(xcoord, ycoord, 'o', markersize=15, zorder=6, markerfacecolor=color, 
                alpha=0.75, markeredgecolor="none")
    plt.annotate(
        str(row['Count']), 
        xy = (xcoord[0], ycoord[0]), xytext = (-20, 20),
        textcoords = 'offset points', ha = 'right', va = 'bottom',
        bbox = dict(boxstyle = 'round,pad=0.5', fc = 'yellow', alpha = 0.75),
        arrowprops = dict(arrowstyle = '->', connectionstyle = 'arc3,rad=0'))
### fake lines used as legend entries    
markers = [plt.Line2D([0,0],[0,0],color=color, marker='o', markersize=15, linestyle='')\
                   for color in id_color.values()]
plt.legend(markers, id_color.keys(), numpoints=1, fontsize=20, markerscale=0.75)

plt.show()
```


![png](output_40_0.png)


The world distribution is still as impressive, and now we see more clearly Employees traveling as we map here locations based on the users time zones as provided by Twitter. Brandwatch appears mentioned in every corner of the world, the largest empty zones even agree with Siberia, the Gobi, Sahara or Australian desert. There are even signs of mentions in the Pacific Ocean with Hawaï and Tahiti.

There are a few oddities though, such as 13 observations in Greenland that would be worth take a look into. We check that the locations have indeed nothing to do with the recorded 'twitterTimeZone':


```python
pdGreenA = pdA[pdA.twitterTimeZone == 'Greenland']
pdGreenM = pdM[pdM.twitterTimeZone == 'Greenland']
for i in range(len(pdGreenM.index)):
    print "###################"
    print pdGreenM.iloc[i, :]['location']
    print pdGreenM.iloc[i, :]['body']
```

    ###################
    Germany; Stuttgart
    @BrandwatchDE @be_gr33n Gehaltsdetails sollten wir natÃ¼rlich schon noch regeln. Nur Luft und Liebe funktioniert bei mir bisher nicht. ;-)
    ###################
    Paris
    RT @BrandwatchFR: Marketing d'#influence : Comment tirer profit de la stratégie #marketing la plus rentable https://t.co/t84yyfYiTA https:/…
    ###################
    Aachen
    Infographic: How to Get More From Social Intelligence in 8 Steps https://t.co/6uwqqhKq3A via @brandwatch #socialintelligence #infographic
    ###################
    Rotterdam
    RT @SMMarketing_EU: Brandwatch:From strength to strength: See how the biggest brands compare on social media -  â¦ https://t.co/D0grxoeDup
    ###################
    Habitando en Madrid
    RT @diegocoquillat: Un placer colaborar con @BrandwatchES  https://t.co/el0yB9ffuA
    ###################
    Hamburg
    @BrandwatchDE @wuv @50hz find ich gut. muss nur bei der liste jedoch gestehen: "recht aktiv" ist wirklich seeehr dehnbar ;-)
    ###################
    Berlin
    Hier kÃ¶nnt Ihr jetzt Euer eigenes Command Center bauen #Workshop #BrandwatchDE #SMWHH https://t.co/5x0vckymYE
    ###################
    Metropolregion Hamburg
    FrauenPower ã· @BrandwatchDE: Gleich im #SMWHH #Lifestream: The Invisible Hand â #Technologie-Trends im @BVDW Panel: https://t.co/sWk2WSQHA4
    ###################
    Berlin
    "unser" Thomas mit @TwitterDE auf der #SMWHH. Vortrag Ã¼ber die Telco Banche auf Twitter beginnt. @BrandwatchDE https://t.co/3IL1jz51l9
    ###################
    Salerno
    The Hidden Value of a #SocialStrategy by @brandwatch #brandwatch #evangelism &gt; https://t.co/xmyblKmyW8 via @SlideShare
    ###################
    Frankfurt am Main
    RT @Brandwatch: 12 months of Twitter data revealed everything you need to know about runners - https://t.co/K50Zhid38E https://t.co/Dpgn1h9â¦
    ###################
    Paris, Ile-de-France
    Test de Brandwatch : l'outil de social listening ultime ? https://t.co/oXWI1Yj0Ux via @leptidigital #socialmedia


Unfortunately the information contained in those tweets apparently emitted from Greenland such as set up locations or tweet bodies do not give away any clue. The most reasonable explanation is that these people were working on social networking while traveling to Greenland. After a little search on the internet to try to validate this asumption, I could link Ms Melissa Lecoanet, a fellow French and a Brandwatch employee showing interest for Nicolas Vanier and his adventurer's camp, validating that at least she might be one to attempt the journey.

Now that our fears of incorrect data are appeased, let us turn to Europe and see what the map looks like at close range:


```python
import csv
import copy
import numpy as np
from mpl_toolkits.basemap import Basemap
import matplotlib.pyplot as plt
from matplotlib.colors import LinearSegmentedColormap

id_color = {'Hi': u'red', 'MidHi': u'darkviolet', 'MidLo': u'turquoise', 'Lo': u'darkcyan'}
id_size = [10,18,25,35]

def ColorSize(val, colors=id_color, size=id_size):
    if val < np.percentile(CoordT['Count'], 25):
        return [colors['Lo'], size[0]]
    elif val < np.percentile(CoordT['Count'], 50):
        return [colors['MidLo'], size[1]]
    elif val < np.percentile(CoordT['Count'], 75):
        return [colors['MidHi'], size[2]]
    else:
        return [colors['Hi'], size[3]]
    
CoordT_eu = copy.deepcopy(CoordT)
CoordT_eu = CoordT_eu[(CoordT.Latitude > 30) & (CoordT.Latitude < 70)]
CoordT_eu = CoordT_eu[(CoordT.Longitude > -25) & (CoordT.Longitude < 50)]


### Use the stereographic projection centered on England with corners
### defined to hide vast regions with no data such as Antartica
plt.rcParams['figure.figsize'] = 25, 25
m  = Basemap(projection='gall',resolution='l',lat_0=0, lon_0=0,
            llcrnrlat=30,urcrnrlat=70,\
            llcrnrlon=-25,urcrnrlon=50)

m.drawcoastlines()
m.drawcountries()
m.drawstates()
m.fillcontinents(color = 'gainsboro', lake_color='steelblue')
m.drawmapboundary(fill_color='steelblue')


for idx, row in CoordT_eu.iterrows():
    x, y = row[['Longitude','Latitude']]
    xcoord, ycoord = m([x], [y])
    color = ColorSize(row['Count'])[0]
    size = ColorSize(row['Count'])[1]
    name = row['Name']
    m.plot(xcoord, ycoord, 'o', markersize=size, zorder=6, markerfacecolor=color, 
                alpha=0.5, markeredgecolor="none")
    plt.annotate(
        str(row['Name']), 
        xy = (xcoord[0], ycoord[0]), xytext = (-20, 20),
        textcoords = 'offset points', ha = 'right', va = 'bottom',
        bbox = dict(boxstyle = 'round,pad=0.5', fc = 'yellow', alpha = 0.75),
        arrowprops = dict(arrowstyle = '->', connectionstyle = 'arc3,rad=0'))
### fake lines used as legend entries    
markers = [plt.Line2D([0,0],[0,0],color=color, marker='o', markersize=size, linestyle='')\
                   for color, size in zip(id_color.values(), id_size)]
plt.legend(markers, id_color.keys(), numpoints=1, fontsize=20, markerscale=0.75)
    
### make image bigger:
#plt.gcf().set_size_inches(25,25)

m.fillcontinents(color = 'gainsboro', lake_color='steelblue')
m.drawmapboundary(fill_color='steelblue')
plt.title("Europe's focus", fontsize=30)
plt.show()
```


![png](output_44_0.png)


With the focus on Europe we observe an important activity in many large cities. Not very surprisingly we find very Tech-friendly cities such as London, Dublin, Amsterdan or Stockolm.

## Create a dummify function to quantify the categorical features and make them usable by ML classification algorithms

Now that we visualize the data a little better, let us try a few Machine Learning techniques to work on classifying or predicting some of the features.

First of all, we define a slew of functions that will help us deal with the categorical features. First a function 'dummify' to transform a categorical variable (typically made of string observations) into a quantitative variable (along with the dictionary to interpret the transformation). Secondly a function 'timesplit' that classifies all recorded zulu times (including dates) into a unique number made of the month, the day and the time by splits of one hour long (so that for instance '2016-03-11T23:51:13.000Z' becomes 031123).

Note that we make our dictionaries for interpretation re-usable with the variable legend, so that we may first create a dictionary for pdA variables and then re-use that same dictionary for the pdM variables. This way, given for instance a feature with binary values, it will not be translated into $0/1$ for pdA and $1/0$ for pdM.


```python
import re
import ast
### label is the list of quantitative observarions and dic is the dictionary translating
### the original data into the quantitative values
def dummify(array, legend=None, start=0):
    if not legend:
        lab, label, dic = start, [], {}
        for x in array:
            if x not in dic.keys():
                dic[x] = lab
                label.append(dic[x])
                lab += 1
            else:
                label.append(dic[x])
        return [label, dic]
    else:
        lab, label = start, []
        for x in array:
            label.append(legend[x])
        return [label, legend]
### remove empty strings by NaN    
def removeEmptyString(array):
    return [x if x != "" else np.nan for x in array]
### transform the string of time zones into a unique number
def timesplit(time):
    if type(time) == list or type(time) == type(np.array([])):
        return [timesplit(x) for x in time]
    elif type(time) == str or unicode:
        tmp = re.split(r'[:-]+|T|Z|\.', time)
        timer = ast.literal_eval(tmp[1].split('0')[1] + tmp[2] + tmp[3])
        return timer
```

We now have all we need to convert the DataFrames pdA and pdM into quantitative ones, making them readily usable by Machine Learning algorithms.


```python
import copy

### Dummify the Data Frames
pdAq = pd.DataFrame()
col_q = ['fol_id', 'fol_location', 'fol_type', 'followersCount', 'friendsCount', 'generator', 
         'geo', 'id', 'klout_score', 'languages', 'listedCount', 'location', 'postedTime', 
         'retweetCount', 'sharetype', 'statusesCount', 'twitterTimeZone', 'user_mentions', 
         'verb']
legendA, legendM = [], []
for name in col_q:
    if type(pdA[name][0]) == int or type(pdA[name][0]) == np.int64:
        pdAq = pd.concat([pdAq, pd.DataFrame({name: removeEmptyString(pdA[name])}, 
                                             index=pdA.index)], axis=1)
    elif name == 'postedTime':
        temp = dummify(timesplit(np.array(pdA[name])))
        legendA.append([name, temp[1]])
        pdAq = pd.concat([pdAq, pd.DataFrame({name: temp[0]}, index=pdA.index)], axis=1)
    else:
        temp = dummify(np.array(pdA[name]))
        legendA.append([name, temp[1]])
        pdAq = pd.concat([pdAq, pd.DataFrame({name: temp[0]}, index=pdA.index)], axis=1)
        
pdMq = pd.DataFrame()
for name in col_q:
    if type(pdM[name][0]) == int or type(pdM[name][0]) == np.int64:
        pdMq = pd.concat([pdMq, pd.DataFrame({name: removeEmptyString(pdM[name])}, 
                                             index=pdM.index)], axis=1)
    elif name == 'postedTime':
        ind = [i for i in range(len(legendA)) if legendA[i][0] == name][0]
        leg = copy.deepcopy(legendA[ind][1])
        if not all(np.unique(pdM[name])) in leg.keys():
            keysM = [timesplit(x) for x in np.array(pdM[name]) if x not in leg.keys()]
            start = max(leg.values())+1
            temp = dummify(keysM, start=start)
            leg.update(temp[1])
        temp = dummify(timesplit(np.array(pdM[name])), legend=leg)
        legendM.append([name, leg])
        pdMq = pd.concat([pdMq, pd.DataFrame({name: temp[0]}, index=pdM.index)], axis=1)
    else:
        ind = [i for i in range(len(legendA)) if legendA[i][0] == name][0]
        leg = copy.deepcopy(legendA[ind][1])
        if not all(np.unique(pdM[name])) in leg.keys():
            keysM = [x for x in np.array(pdM[name]) if x not in leg.keys()]
            start = max(leg.values())+1
            temp = dummify(keysM, start=start)
            leg.update(temp[1])
        temp = dummify(np.array(pdM[name]), legend=leg)
        legendM.append([name, temp[1]])
        pdMq = pd.concat([pdMq, pd.DataFrame({name: temp[0]}, index=pdM.index)], axis=1)
        
print [[legendA[i][0], len(legendA[i][1])] for i in range(len(legendA))]
print [[legendM[i][0], len(legendM[i][1])] for i in range(len(legendM))]
pdAq.head()
```

    /Users/boulenge/anaconda/lib/python2.7/site-packages/numpy/lib/arraysetops.py:200: FutureWarning: numpy not_equal will not check object identity in the future. The comparison did not return the same result as suggested by the identity (`is`)) and will change.
      flag = np.concatenate(([True], aux[1:] != aux[:-1]))


    [['fol_id', 2948], ['fol_location', 2], ['fol_type', 2], ['generator', 35], ['geo', 21], ['id', 126], ['languages', 4], ['location', 60], ['postedTime', 1577], ['sharetype', 4], ['twitterTimeZone', 25], ['user_mentions', 1432], ['verb', 2]]
    [['fol_id', 4734], ['fol_location', 2], ['fol_type', 2], ['generator', 127], ['geo', 22], ['id', 1298], ['languages', 16], ['location', 716], ['postedTime', 2057], ['sharetype', 4], ['twitterTimeZone', 72], ['user_mentions', 1948], ['verb', 2]]





<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>fol_id</th>
      <th>fol_location</th>
      <th>fol_type</th>
      <th>followersCount</th>
      <th>friendsCount</th>
      <th>generator</th>
      <th>geo</th>
      <th>id</th>
      <th>klout_score</th>
      <th>languages</th>
      <th>listedCount</th>
      <th>location</th>
      <th>postedTime</th>
      <th>retweetCount</th>
      <th>sharetype</th>
      <th>statusesCount</th>
      <th>twitterTimeZone</th>
      <th>user_mentions</th>
      <th>verb</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1299</td>
      <td>999</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>45</td>
      <td>0</td>
      <td>42</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>6301</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>1165</td>
      <td>618</td>
      <td>1</td>
      <td>0</td>
      <td>1</td>
      <td>45</td>
      <td>0</td>
      <td>55</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>3532</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2</td>
      <td>0</td>
      <td>0</td>
      <td>198</td>
      <td>522</td>
      <td>2</td>
      <td>0</td>
      <td>2</td>
      <td>29</td>
      <td>0</td>
      <td>9</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>1157</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>3</td>
      <td>0</td>
      <td>0</td>
      <td>7451</td>
      <td>1182</td>
      <td>2</td>
      <td>0</td>
      <td>3</td>
      <td>57</td>
      <td>0</td>
      <td>706</td>
      <td>2</td>
      <td>2</td>
      <td>0</td>
      <td>0</td>
      <td>29617</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>4</td>
      <td>0</td>
      <td>0</td>
      <td>482</td>
      <td>242</td>
      <td>1</td>
      <td>0</td>
      <td>4</td>
      <td>39</td>
      <td>0</td>
      <td>34</td>
      <td>1</td>
      <td>3</td>
      <td>0</td>
      <td>0</td>
      <td>5580</td>
      <td>2</td>
      <td>1</td>
      <td>0</td>
    </tr>
  </tbody>
</table>
</div>



Check for instance that the binary variable 'verb' has been translated the same way for both pdA and pdM:


```python
print np.mean(pdAq['verb']), np.mean(pdMq['verb'])
print legendA[-1][1], legendM[-1][1]
```

    0.212701612903 0.401940109658
    {u'post': 0, u'share': 1} {u'post': 0, u'share': 1}


## Post type prediction 

Assume we had all the information contained in the Dataframes pdA and pdM above, but for the feature 'verb' that decides whether a post is an original comment or a share. Could we design an algorithm to recover that information from the remaining features? And if yes, then which features are the most relevant to do so? Let us try a näive model with Logistic Regression, using all the available features in the dataframe pdA, just so we get a sense of whether or not that prediction is feasible.


```python
import random
import sys
import time

from sklearn import linear_model
clf = linear_model.LogisticRegression()

pdAq = pdAq.dropna()
n = len(pdAq.index)
N = int(0.75*n)

Run = range(50)
Acc = []

for run in Run:
    row_train = random.sample(range(n), N)
    row_test = [i for i in range(n) if i not in row_train]

    col_train = [x for x in pdAq.columns if x != 'verb']

    clf.fit(pdAq.iloc[row_train, :][col_train], pdAq.iloc[row_train, :]['verb'])
    pred = clf.predict(pdAq.iloc[row_test, :][col_train])
    target = pdAq.iloc[row_test, :]['verb']
    Acc.append(np.mean(pred == target))
    
print "Minimum accuracy over {} runs: {}".format(len(Run), min(Acc))
print "Maximum accuracy reached over {} runs: {}".format(len(Run), max(Acc))
```

    Minimum accuracy over 50 runs: 0.92349726776
    Maximum accuracy reached over 50 runs: 1.0


As we can see even after trying 50 times, the minimum accuracy that was reached is about 92%, a very good score that suggests that this prediction is doable. Now is Logistic Regression the best model there is for that problem?

But first, let us define another dummified dataset with all the data, including the source of the observation (0 for Employee and 1 for Mention):


```python
pdAq['source'] =  [0 for i in range(len(pdAq.index))]
pdMq['source'] =  [1 for i in range(len(pdMq.index))]

pdAq = pdAq.dropna()
pdMq = pdMq.dropna()
pdTq = pd.concat([pdAq, pdMq], axis=0).reset_index()
pdTq = pdTq.iloc[:, 1:]
pdTq.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>fol_id</th>
      <th>fol_location</th>
      <th>fol_type</th>
      <th>followersCount</th>
      <th>friendsCount</th>
      <th>generator</th>
      <th>geo</th>
      <th>id</th>
      <th>klout_score</th>
      <th>languages</th>
      <th>listedCount</th>
      <th>location</th>
      <th>postedTime</th>
      <th>retweetCount</th>
      <th>sharetype</th>
      <th>statusesCount</th>
      <th>twitterTimeZone</th>
      <th>user_mentions</th>
      <th>verb</th>
      <th>source</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1299</td>
      <td>999</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>45</td>
      <td>0</td>
      <td>42</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>6301</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>1165</td>
      <td>618</td>
      <td>1</td>
      <td>0</td>
      <td>1</td>
      <td>45</td>
      <td>0</td>
      <td>55</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>3532</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2</td>
      <td>0</td>
      <td>0</td>
      <td>198</td>
      <td>522</td>
      <td>2</td>
      <td>0</td>
      <td>2</td>
      <td>29</td>
      <td>0</td>
      <td>9</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>1157</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>3</td>
      <td>0</td>
      <td>0</td>
      <td>7451</td>
      <td>1182</td>
      <td>2</td>
      <td>0</td>
      <td>3</td>
      <td>57</td>
      <td>0</td>
      <td>706</td>
      <td>2</td>
      <td>2</td>
      <td>0</td>
      <td>0</td>
      <td>29617</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>4</td>
      <td>0</td>
      <td>0</td>
      <td>482</td>
      <td>242</td>
      <td>1</td>
      <td>0</td>
      <td>4</td>
      <td>39</td>
      <td>0</td>
      <td>34</td>
      <td>1</td>
      <td>3</td>
      <td>0</td>
      <td>0</td>
      <td>5580</td>
      <td>2</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
    </tr>
  </tbody>
</table>
</div>



Let us still use all the features for now, just trying to see which model is best over 10 runs and keep heir lowest and highest accuracies:


```python
import random
import sys
import time

from sklearn import linear_model
logit = linear_model.LogisticRegression()

from sklearn.naive_bayes import BernoulliNB
bnb = BernoulliNB()

from sklearn.ensemble import GradientBoostingClassifier
xgb = GradientBoostingClassifier()

from sklearn.neighbors import KNeighborsClassifier
knn = KNeighborsClassifier(n_neighbors=len(pdTq.columns)-1)

from sklearn.ensemble import RandomForestClassifier
rfc = RandomForestClassifier(n_estimators=10, criterion='gini')

Mod = [logit, bnb, knn, xgb, rfc]
Mod_name = {logit: "Logit", bnb: "Bayes", knn: "K-NN", xgb: "XGBoost", rfc: "RandomForest"}


n = len(pdTq.index)
N = int(0.75*n)

Run = range(10)

col_train, col_train2 = [x for x in pdAq.columns if x != 'verb'], []
Nft = len(col_train)

Acc = {Mod_name[mod]: [1, 0] for mod in Mod}

for mod in Mod:
    for run in Run:
        row_train = random.sample(range(n), N)
        row_test = [i for i in range(n) if i not in row_train]

        mod.fit(pdTq.iloc[row_train, :][col_train], pdTq.iloc[row_train, :]['verb'])
        pred = mod.predict(pdTq.iloc[row_test, :][col_train])
        target = pdTq.iloc[row_test, :]['verb']
        acc = np.mean(pred == target)
        if acc < Acc[Mod_name[mod]][0]:
            Acc[Mod_name[mod]][0] = round(acc, 2)
        if acc > Acc[Mod_name[mod]][1]:
            Acc[Mod_name[mod]][1] = round(acc, 2)
        if Mod_name[mod] == "RandomForest":
            tp = [col_train[i] for i in range(Nft) if mod.feature_importances_[i] > 0.05]
            for name in tp:
                if name not in col_train2:
                    col_train2.append(name)
print Acc
print col_train2
```

    {'Logit': [0.86, 1.0], 'RandomForest': [1.0, 1.0], 'XGBoost': [1, 1.0], 'K-NN': [0.74, 0.78], 'Bayes': [1, 1.0]}
    ['retweetCount', 'user_mentions', 'sharetype']


It turns out the best models are Random Forest Classifier, XGBoost (which is slower) and Naïve Bayes. We see that K-NN does a rather poor job (even when varying the number of nighbors). This suggests that the data is not very well linearly separated and disqualify standard support vector machines methods which explain why we didn't try them here.

We further use a method from the sklearn Random Forest package that computes the features importance. After running it 10 times, we see that the only variables that scored over 0.05 were 'user_mentions' and more importantly 'retweetCount' which seems highly determinant in predicting the type of post.

One may suspect the 100% accuracy observed many times with different models is an indication that the type of post and the retweetCount variables are very similar, perhaps too much and we may think that if one is missing, so should the other. When we exclude the retweetCount variable, things look different and it is easier to spot how well do each model performs. But the good news is that the prediction is still possible, though not as precise as before.


```python
Run = range(10)

col_train, col_train2 = [x for x in pdAq.columns if x != 'verb' and x != 'retweetCount'], []
Nft = len(col_train)

Acc = {Mod_name[mod]: [1, 0] for mod in Mod}

for mod in Mod:
    for run in Run:
        row_train = random.sample(range(n), N)
        row_test = [i for i in range(n) if i not in row_train]

        mod.fit(pdTq.iloc[row_train, :][col_train], pdTq.iloc[row_train, :]['verb'])
        pred = mod.predict(pdTq.iloc[row_test, :][col_train])
        target = pdTq.iloc[row_test, :]['verb']
        acc = np.mean(pred == target)
        if acc < Acc[Mod_name[mod]][0]:
            Acc[Mod_name[mod]][0] = round(acc, 2)
        if acc > Acc[Mod_name[mod]][1]:
            Acc[Mod_name[mod]][1] = round(acc, 2)
        if Mod_name[mod] == "RandomForest":
            tp = [col_train[i] for i in range(Nft) if mod.feature_importances_[i] > 0.1]
            for name in tp:
                if name not in col_train2:
                    col_train2.append(name)
print Acc
print "Feature importances:"
print [tuple([x, round(y, 2)]) for x, y in zip(col_train, rfc.feature_importances_)]
```

    {'Logit': [0.77, 0.82], 'RandomForest': [0.85, 0.88], 'XGBoost': [0.87, 0.89], 'K-NN': [0.75, 0.77], 'Bayes': [0.8, 0.84]}
    Feature importances:
    [('fol_id', 0.05), ('fol_location', 0.01), ('fol_type', 0.04), ('followersCount', 0.06), ('friendsCount', 0.1), ('generator', 0.05), ('geo', 0.0), ('id', 0.06), ('klout_score', 0.07), ('languages', 0.01), ('listedCount', 0.04), ('location', 0.07), ('postedTime', 0.06), ('sharetype', 0.07), ('statusesCount', 0.07), ('twitterTimeZone', 0.03), ('user_mentions', 0.21), ('source', 0.0)]


One feature dominates all the other in terms of predictive power: 'user_mentions' that lists the users being mentioned within the tweet's structure. Mentioning users already seems an indication that a tweet is a either a reply or a share of another post. Perhaps that feature is too big of a give away for a tweet's type. To see whether that information is determinant in deciding whether a tweet is a share or not, let us first exclude it from our model.

It seems as though more variables are needed to make our prediction work without 'user_mentions', as 11 of them have scores around 5%. Using much less than 11 features might thus damage the predictive power. Let us try nonetheless to select only a slew of variables that seem mostly mostly uncorrolated to the type of post and that should be easy enough to collect. For instance looking at a user's profile, we may be tempted to first classify it by time zone, number of followers and location (which may add a little precision to the Time Zone). Remember the geographical inaccuracy we observed for the 'location' feature. Here it might be that it helps us as a user's tendency to joke about his/her location may have something to say about his/her character and the kind of posts we are more likely to find on the profile.

We therefore select only these 4 features with moderate predictive power and use a 20-fold Cross-Validation to better tune our Random Forest model.


```python
from sklearn import ensemble
import sklearn.cross_validation as cv
from sklearn.cross_validation import KFold, train_test_split
from sklearn.grid_search import GridSearchCV
import random

col_train = ['twitterTimeZone', 'followersCount', 'location', 'postedTime']

X_train, X_test, y_train, y_test = \
    cv.train_test_split(pdTq[col_train], pdTq['verb'], test_size=0.25, random_state=0)

param = {'n_estimators': [2, 4, 10, 20], 
         'criterion': ['gini', 'entropy'],
         'max_depth': [None, 5, 10, 20, 50], 
         'min_samples_split': [1, 2, 3], 
         'min_samples_leaf': [1, 2], 
         'max_features': ['auto', 'sqrt'],
         'oob_score': [False]}

print str(np.product([len(param[x]) for x in param])) + " models to test"

rf = ensemble.RandomForestClassifier()
rfsearch = GridSearchCV(rf, param, verbose=1, cv=20)
rfsearch.fit(X_train, y_train)
print(rfsearch.best_score_)
print(rfsearch.best_params_)
```

    480 models to test
    Fitting 20 folds for each of 480 candidates, totalling 9600 fits


    [Parallel(n_jobs=1)]: Done  49 tasks       | elapsed:    1.7s
    [Parallel(n_jobs=1)]: Done 199 tasks       | elapsed:   11.7s
    [Parallel(n_jobs=1)]: Done 449 tasks       | elapsed:   27.8s
    [Parallel(n_jobs=1)]: Done 799 tasks       | elapsed:   52.7s
    [Parallel(n_jobs=1)]: Done 1249 tasks       | elapsed:  1.2min
    [Parallel(n_jobs=1)]: Done 1799 tasks       | elapsed:  1.7min
    [Parallel(n_jobs=1)]: Done 2449 tasks       | elapsed:  2.3min
    [Parallel(n_jobs=1)]: Done 3199 tasks       | elapsed:  3.0min
    [Parallel(n_jobs=1)]: Done 4049 tasks       | elapsed:  3.9min
    [Parallel(n_jobs=1)]: Done 4999 tasks       | elapsed:  5.0min
    [Parallel(n_jobs=1)]: Done 6049 tasks       | elapsed:  6.3min
    [Parallel(n_jobs=1)]: Done 7199 tasks       | elapsed:  7.4min
    [Parallel(n_jobs=1)]: Done 8449 tasks       | elapsed:  8.9min


    0.771587743733
    {'oob_score': False, 'min_samples_leaf': 2, 'n_estimators': 20, 'min_samples_split': 2, 'criterion': 'entropy', 'max_features': 'sqrt', 'max_depth': 10}


    [Parallel(n_jobs=1)]: Done 9600 out of 9600 | elapsed: 10.5min finished


The best model we obtained used a Forest of 20 trees, each of which being at most 10 steps deep and based on the Entropy index as purity criterion. The best score obtained was almost 76%, not too bad for such a reduced model.

Let us now see if we can do much better or not by using again the feature 'user_mentions' this time. Since it previously gave better scores than Random Forest, we also use the XGBoost algorithm from the sklearn library. Like Random Forest, it builds a collection of trees, but each new tree is built using an objective function that uses the previous trees prediction to update the new one, thus learning from past errors. Each tree's contribution is also shrunk by a learning rate factor so that the prediction is not fully optimized at a given step, leaving room for later steps and preventing overfit.


```python
from sklearn import ensemble
import sklearn.cross_validation as cv
from sklearn.cross_validation import KFold, train_test_split
from sklearn.grid_search import GridSearchCV
import random

col_train = ['user_mentions', 'followersCount', 'location', 'postedTime']

X_train, X_test, y_train, y_test = \
    cv.train_test_split(pdTq[col_train], pdTq['verb'], test_size=0.25, random_state=0)

param = {'loss': ['deviance', 'exponential'],
         'learning_rate': [0.01, 0.1, 0.5, 1],
         'n_estimators': [2, 4, 10, 20],
         'max_depth': [None, 5, 10, 20], 
         'min_samples_split': [1, 2, 3, 5], 
         'min_samples_leaf': [1, 2, 3], 
         'max_features': [None, 0.5]}

print str(np.product([len(param[x]) for x in param])) + " models to test"

xgb = ensemble.GradientBoostingClassifier()
xgbsearch = GridSearchCV(xgb, param, verbose=1, cv=5)
xgbsearch.fit(X_train, y_train)
print(rfsearch.best_score_)
print(rfsearch.best_params_)
```

    3072 models to test
    Fitting 5 folds for each of 3072 candidates, totalling 15360 fits


    [Parallel(n_jobs=1)]: Done  49 tasks       | elapsed:    9.2s
    [Parallel(n_jobs=1)]: Done 199 tasks       | elapsed:   39.6s
    [Parallel(n_jobs=1)]: Done 449 tasks       | elapsed:  1.5min
    [Parallel(n_jobs=1)]: Done 799 tasks       | elapsed:  1.8min
    [Parallel(n_jobs=1)]: Done 1249 tasks       | elapsed:  2.3min
    [Parallel(n_jobs=1)]: Done 1799 tasks       | elapsed:  3.8min
    [Parallel(n_jobs=1)]: Done 2449 tasks       | elapsed:  5.9min
    [Parallel(n_jobs=1)]: Done 3199 tasks       | elapsed:  6.7min
    [Parallel(n_jobs=1)]: Done 4049 tasks       | elapsed:  9.4min
    [Parallel(n_jobs=1)]: Done 4999 tasks       | elapsed: 10.9min
    [Parallel(n_jobs=1)]: Done 6049 tasks       | elapsed: 14.1min
    [Parallel(n_jobs=1)]: Done 7199 tasks       | elapsed: 15.9min
    [Parallel(n_jobs=1)]: Done 8449 tasks       | elapsed: 20.2min
    [Parallel(n_jobs=1)]: Done 9799 tasks       | elapsed: 24.3min
    [Parallel(n_jobs=1)]: Done 11249 tasks       | elapsed: 27.9min
    [Parallel(n_jobs=1)]: Done 12799 tasks       | elapsed: 31.6min
    [Parallel(n_jobs=1)]: Done 14449 tasks       | elapsed: 35.6min
    [Parallel(n_jobs=1)]: Done 15360 out of 15360 | elapsed: 37.8min finished


    0.787541149658
    {'oob_score': False, 'min_samples_leaf': 2, 'n_estimators': 4, 'min_samples_split': 1, 'criterion': 'entropy', 'max_features': 'auto', 'max_depth': 8}


More elaborate, XGBoost is also slower to fit observations. Using only 5-folds Cross-Validation, the best model that includes 'iser_mentions' only reached close to 79%, not a terrific improvement from our previous simple model.

Let us actually go back to that first tuned and reduced model. Recall the best one had used 20 Trees to make its prediction, the larger number of Trees we had allowed it to use. Perhaps we should therefore keep increasing the number of Trees in the Forest. But how many Trees should we use? To help us decide, we run different models by looping over a list of number of Trees to use, all others parameters being fixed. To make the test more robust, each model is ran several time and we only keep the average accuracy of each model. After this is done we may plot the accuracy depending on the number of Trees used in the model and expect to see a curve with shaped like an elbow. Once increasing the number of Trees doesn't spectacularly improve the accuracy we will know that is a limit to the number of Trees we should use. Doing so we also fight against overfit.


```python
Run, n = range(10), len(pdTq.index)
NbTrees = range(5, 100, 5)
N = int(0.75*n)
mean_acc = []
col_train = ['twitterTimeZone', 'followersCount', 'location', 'postedTime']
cpt = 1
for nb in NbTrees:
    rf = ensemble.RandomForestClassifier(oob_score=False, min_samples_leaf=2, n_estimators=nb, 
                                    min_samples_split=2, criterion='entropy', 
                                    max_features='auto', max_depth=10)
    mod_acc = []
    for run in Run:
        row_train = random.sample(range(n), N)
        row_test = [i for i in range(n) if i not in row_train]

        rf.fit(pdTq.iloc[row_train, :][col_train], pdTq.iloc[row_train, :]['verb'])
        pred = rf.predict(pdTq.iloc[row_test, :][col_train])
        target = pdTq.iloc[row_test, :]['verb']
        mod_acc.append(np.mean(pred == target))
    mean_acc.append(np.mean(mod_acc))
    cpt += 1
ind = [i for i in range(len(mean_acc)) if mean_acc[i] == max(mean_acc)][0]
print "Best mean-accuracy: {} for {} Trees".format(mean_acc[ind], NbTrees[ind])
```

    Best mean-accuracy: 0.762490508732 for 55 Trees



```python
Run, n = range(10), len(pdTq.index)
NbTrees_10 = range(5, 100, 10)
N = int(0.75*n)
mean_acc_10 = []
col_train = ['twitterTimeZone', 'followersCount', 'location', 'postedTime']
cpt = 1
for nb in NbTrees:
    rf = ensemble.RandomForestClassifier(oob_score=False, min_samples_leaf=2, n_estimators=nb, 
                                    min_samples_split=2, criterion='entropy', 
                                    max_features='auto', max_depth=10)
    mod_acc_10 = []
    for run in Run:
        row_train = random.sample(range(n), N)
        row_test = [i for i in range(n) if i not in row_train]

        rf.fit(pdTq.iloc[row_train, :][col_train], pdTq.iloc[row_train, :]['verb'])
        pred = rf.predict(pdTq.iloc[row_test, :][col_train])
        target = pdTq.iloc[row_test, :]['verb']
        mod_acc_10.append(np.mean(pred == target))
    mean_acc_10.append(np.mean(mod_acc_10))
    cpt += 1
ind = [i for i in range(len(mean_acc_10)) if mean_acc_10[i] == max(mean_acc_10)][0]
print "Best mean-accuracy: {} for {} Trees".format(mean_acc_10[ind], NbTrees[ind])
```

    Best mean-accuracy: 0.765375854214 for 60 Trees


Since there seems to be some variance in the models, we used steps of respectively 5 and 10 Trees to better capture the trend:


```python
import matplotlib.pyplot as plt
%matplotlib inline
plt.rcParams['figure.figsize'] = 10, 5
plt.plot(NbTrees, mean_acc, linestyle="--", marker="o", c="b", label="mean_acc_5")
plt.plot(NbTrees, mean_acc_10, linestyle="--", marker="o", c="r", label="mean_acc_10")
plt.title("Mean accuracy of the models by number of Trees in the Forest", fontsize=15)
plt.legend(loc='upper left')
```




    <matplotlib.legend.Legend at 0x130b21950>




![png](output_70_1.png)


It appears the mean accuracy improves until between 40 and 60 Trees, with the best result obtained between 55 and 60 Trees. It seems this is where we need to fix our number of Trees, say at 55. To further evaluate this tuned result model's accuracy variance, we run it 100 more times:


```python
rf = ensemble.RandomForestClassifier(oob_score=False, min_samples_leaf=2, n_estimators=55, 
                                    min_samples_split=2, criterion='entropy', 
                                    max_features='auto', max_depth=10)
Run, n = range(100), len(pdTq.index)
N = int(0.75*n)
best_acc = []
for run in Run:
    row_train = random.sample(range(n), N)
    row_test = [i for i in range(n) if i not in row_train]

    rf.fit(pdTq.iloc[row_train, :][col_train], pdTq.iloc[row_train, :]['verb'])
    pred = rf.predict(pdTq.iloc[row_test, :][col_train])
    target = pdTq.iloc[row_test, :]['verb']
    best_acc.append(np.mean(pred == target))
print "Lowest/Average/Highest accuracy over {} runs: {}/{}/{}".\
format(len(Run), round(min(best_acc), 2), round(np.mean(best_acc), 2), round(max(best_acc), 2))
print "Standard Deviation of the accuracy: {}".format(np.std(best_acc))
```

    Lowest/Average/Highest accuracy over 100 runs: 0.74/0.76/0.79
    Standard Deviation of the accuracy: 0.00961244138289


We end up with a quite simple reduced model based on 4 features with about 76% accuracy, even though we do not use the 'retweetCount' variable with the best predictive power. The standard deviation of the accuracy is also quite low, barely at 0.01 making the two digits accuracy relatively stable.

## Number of retweets prediction 

The number of retweets is most likely a non-linear feature. Instead of using a Linear Regression to fisrt test whether the prediction is doable or not, we thus use a regression tree method, and since it is available in the sklearn library we use the more advanced Random Forest regression. Instead of using one regression tree, the algorithms relies on several trees which are made uncorrolated by picking features at random. The regression results are then averaged among all trees. Evidently such a method is more flexible and can handle certain nonlinear behaviours.


```python
from sklearn import ensemble

col_train = [x for x in pdTq.columns if x != 'retweetCount']

rf1 = ensemble.RandomForestRegressor(n_estimators=len(col_train),max_depth =50,oob_score=True)
rf2 = ensemble.RandomForestRegressor(n_estimators=len(col_train),max_depth =50,oob_score=True)

N = int(0.75*len(pdTq.index))
row_train = random.sample(range(len(pdTq.index)), N)
row_test = [i for i in range(len(pdTq.index)) if i not in row_train]
target = pdTq.iloc[row_test, :]['retweetCount']

row_train_large = [i for i in row_train if pdTq.iloc[i, :]['retweetCount'] > 1]
row_train_small = [i for i in row_train if i not in row_train_large]

rf1.fit(pdTq.iloc[row_train_large, :][col_train],\
        pdTq.iloc[row_train_large, :]['retweetCount'])
rf2.fit(pdTq.iloc[row_train_small, :][col_train],\
        pdTq.iloc[row_train_small, :]['retweetCount'])

pred_large = rf1.predict(pdTq.iloc[row_test, :][col_train])
pred_small = rf2.predict(pdTq.iloc[row_test, :][col_train])

pred = []
for x, y in zip(pred_large, pred_small):
    if y == 0:
        pred.append(y)
    elif x > np.percentile(pred_large, 95):
        pred.append(x)
    else:
        pred.append(np.mean([x, y]))
print "   Average absolute errors:"
print "Large prediction: {}".format(np.mean(np.abs(pred_large-target)))
print "Small prediction: {}".format(np.mean(np.abs(pred_small-target)))
print "Mixed prediction: {}".format(np.mean(np.abs(pred-target)))
```

       Average absolute errors:
    Large prediction: 748.538384686
    Small prediction: 73.6772968869
    Mixed prediction: 160.074551413


The average absolute errors could make us doubt our models. For the mixed prediction, about 160 count errors on average looks like quite a large error. However one should realize the number of retweets in our dataset ranges from 0 to 155609 and contains 70% of 0s.

If we take a look at what the predictions look like on a plot, we see that the mixed one actually follows the target pretty well.


```python
import matplotlib.pyplot as plt
%matplotlib inline
Xx, Ytarget = range(len(pred)), np.sort(target)
Ylarge, Ysmall, Ymixed = np.sort(pred_large), np.sort(pred_small), np.sort(pred)
plt.rcParams['figure.figsize'] = 10, 5
plt.subplot(2, 1, 1)
plt.plot(Xx, Ytarget, c="k", label="target")
plt.plot(Xx, Ylarge, c="r", label="large")
plt.plot(Xx, Ysmall, c="b", label="small")
plt.plot(Xx, Ymixed, c="m", label="mixed")
plt.xlim(0, 1400)
plt.ylim(0, 400)
plt.title("Visual fit of the models", fontsize=25)
plt.legend(loc='upper left')

plt.subplot(2, 1, 2)
plt.plot(Xx, Ytarget, c="k")
plt.plot(Xx, Ylarge, c="r")
plt.plot(Xx, Ysmall, c="b")
plt.plot(Xx, Ymixed, c="m")
plt.xlim(1250, len(pred))
plt.ylim(0, max(target))
plt.show()
```


![png](output_78_0.png)


Once again, we can simplify the model based on the features importance. Interestingly we see the source of the information (Employee or simply mentioning Brandwatch) does not play any role in the prediction.


```python
print [tuple([x, round(y, 2)]) for x, y in zip(col_train, rf1.feature_importances_)]
print [tuple([x, round(y, 2)]) for x, y in zip(col_train, rf2.feature_importances_)]
```

    [('fol_id', 0.07), ('fol_location', 0.0), ('fol_type', 0.04), ('followersCount', 0.13), ('friendsCount', 0.08), ('generator', 0.02), ('geo', 0.0), ('id', 0.03), ('klout_score', 0.06), ('languages', 0.01), ('listedCount', 0.18), ('location', 0.03), ('postedTime', 0.02), ('sharetype', 0.06), ('statusesCount', 0.03), ('twitterTimeZone', 0.02), ('user_mentions', 0.25), ('verb', 0.0), ('source', 0.0)]
    [('fol_id', 0.0), ('fol_location', 0.0), ('fol_type', 0.0), ('followersCount', 0.0), ('friendsCount', 0.0), ('generator', 0.0), ('geo', 0.0), ('id', 0.0), ('klout_score', 0.0), ('languages', 0.0), ('listedCount', 0.0), ('location', 0.0), ('postedTime', 0.0), ('sharetype', 0.0), ('statusesCount', 0.0), ('twitterTimeZone', 0.0), ('user_mentions', 0.0), ('verb', 1.0), ('source', 0.0)]


For further ease, let us also use a naïve Bayes classifier for the small values instead of a regressor.


```python
col_train_large = [x for x, y in zip(col_train, rf1.feature_importances_) if y > 0.05]
col_train_small = [x for x, y in zip(col_train, rf2.feature_importances_) if y > 0.05]

from sklearn.naive_bayes import BernoulliNB
bnb = BernoulliNB()

rf1 = ensemble.RandomForestRegressor(n_estimators=len(col_train),max_depth =50,oob_score=True)
N = int(0.75*len(pdTq.index))
row_train = random.sample(range(len(pdTq.index)), N)
row_test = [i for i in range(len(pdTq.index)) if i not in row_train]
target = pdTq.iloc[row_test, :]['retweetCount']

row_train_large = [i for i in row_train if pdTq.iloc[i, :]['retweetCount'] > 1]
row_train_small = [i for i in row_train if i not in row_train_large]

rf1.fit(pdTq.iloc[row_train_large, :][col_train_large],\
        pdTq.iloc[row_train_large, :]['retweetCount'])
bnb.fit(pdTq.iloc[row_train_small, :][col_train_small],\
        pdTq.iloc[row_train_small, :]['retweetCount'])

pred_large = rf1.predict(pdTq.iloc[row_test, :][col_train_large])
pred_small = bnb.predict(pdTq.iloc[row_test, :][col_train_small])

pred = []
for x, y in zip(pred_large, pred_small):
    if y == 0:
        pred.append(y)
    elif x > np.percentile(pred_large, 95):
        pred.append(x)
    else:
        pred.append(np.mean([x, y]))
print "   Average absolute errors:"
print "Large prediction: {}".format(np.mean(np.abs(pred_large-target)))
print "Small prediction: {}".format(np.mean(np.abs(pred_small-target)))
print "Mixed prediction: {}".format(np.mean(np.abs(pred-target)))
```

       Average absolute errors:
    Large prediction: 489.676337769
    Small prediction: 81.8390280942
    Mixed prediction: 226.83353315


Visually, the simplified model also looks quite good:


```python
import matplotlib.pyplot as plt
%matplotlib inline
Xx, Ytarget = range(len(pred)), np.sort(target)
Ylarge, Ysmall, Ymixed = np.sort(pred_large), np.sort(pred_small), np.sort(pred)
plt.rcParams['figure.figsize'] = 10, 5
plt.subplot(2, 1, 1)
plt.plot(Xx, Ytarget, c="k", label="target")
plt.plot(Xx, Ylarge, c="r", label="large")
plt.plot(Xx, Ysmall, c="b", label="small")
plt.plot(Xx, Ymixed, c="m", label="mixed")
plt.xlim(0, 1400)
plt.ylim(0, 400)
plt.title("Visual fit of the models", fontsize=25)
plt.legend(loc='upper left')

plt.subplot(2, 1, 2)
plt.plot(Xx, Ytarget, c="k")
plt.plot(Xx, Ylarge, c="r")
plt.plot(Xx, Ysmall, c="b")
plt.plot(Xx, Ymixed, c="m")
plt.xlim(1250, len(pred))
plt.ylim(0, max(target))
plt.show()
```


![png](output_84_0.png)


## Source prediction and $\chi^2$ test

As a last prediction, we use the source variable we included to the DataFrame pdTq to recall whether an observation was orinitated from either pdA or pdM, that is authored by a Brandwatch employee or by someone only mentioning the company's name. We already mentioned this feature might not be very well defined as some people sampled based on the fact they mentioned the company might also happen to also be an employee.

That being said, I am curious to see whether some variables seem to depend on the source or not. We start by using the $\chi^2$ test of independence for source and each of the other avaiable features.


```python
import scipy.stats

col_chi2 = [x for x in pdTqq.columns if x != 'source']
source_dep = []
for col in col_chi2:
    ind = [i for i in range(len(pdTq.index))\
                   if pdTq.iloc[i, :]['source'] != 0 and pdTq.iloc[i, :][col] != 0]
    a = [pdTq.iloc[i, :]['source'] for i in ind]
    b = [pdTq.iloc[i, :][col] for i in ind]
    p = scipy.stats.chi2_contingency([a, b])[1]
    print "p-value for Chi2 test of source/{}: {}".format(col, p)
    if p > 0.05:
        source_dep.append(col)
```

    p-value for Chi2 test of source/fol_id: 1.67647479473e-75
    p-value for Chi2 test of source/fol_location: 1.0
    p-value for Chi2 test of source/fol_type: 1.0
    p-value for Chi2 test of source/followersCount: 0.0
    p-value for Chi2 test of source/friendsCount: 0.0
    p-value for Chi2 test of source/generator: 1.55926980185e-36
    p-value for Chi2 test of source/geo: 1.0
    p-value for Chi2 test of source/id: 0.0
    p-value for Chi2 test of source/klout_score: 1.0
    p-value for Chi2 test of source/languages: 1.0
    p-value for Chi2 test of source/listedCount: 0.0
    p-value for Chi2 test of source/location: 0.0
    p-value for Chi2 test of source/postedTime: 1.0
    p-value for Chi2 test of source/retweetCount: 1.22164353903e-240
    p-value for Chi2 test of source/sharetype: 1.0
    p-value for Chi2 test of source/statusesCount: 0.0
    p-value for Chi2 test of source/twitterTimeZone: 2.65711355651e-14
    p-value for Chi2 test of source/user_mentions: 0.0
    p-value for Chi2 test of source/verb: 1.0


Using only the non-zero values take by every variable, it turns out the Null hypothesis of the independence test is to be reject for 8 of the variables. Once again there seems to be a dichotomy here: either the p-values are extremely small or equal to 1, letting suspect there is more than a mere correlation. Running again a few classification models we find again 100% accuracies scores letting suspect there are serious overlaps in the variables.


```python
import random
import sys
import time

from sklearn import linear_model
logit = linear_model.LogisticRegression()

from sklearn.naive_bayes import BernoulliNB
bnb = BernoulliNB()

from sklearn.ensemble import GradientBoostingClassifier
xgb = GradientBoostingClassifier()

from sklearn.neighbors import KNeighborsClassifier
knn = KNeighborsClassifier(n_neighbors=len(pdTq.columns)-1)

from sklearn.ensemble import RandomForestClassifier
rfc = RandomForestClassifier(n_estimators=10, criterion='gini')

Mod = [logit, bnb, knn, xgb, rfc]
Mod_name = {logit: "Logit", bnb: "Bayes", knn: "K-NN", xgb: "XGBoost", rfc: "RandomForest"}


n = len(pdTq.index)
N = int(0.75*n)

Run = range(10)
col_train = source_dep
col_train2 = []

Nft = len(col_train)

Acc = {Mod_name[mod]: [1, 0] for mod in Mod}

for mod in Mod:
    for run in Run:
        row_train = random.sample(range(n), N)
        row_test = [i for i in range(n) if i not in row_train]

        mod.fit(pdTq.iloc[row_train, :][col_train], pdTq.iloc[row_train, :]['source'])
        pred = mod.predict(pdTq.iloc[row_test, :][col_train])
        target = pdTq.iloc[row_test, :]['source']
        acc = np.mean(pred == target)
        if acc < Acc[Mod_name[mod]][0]:
            Acc[Mod_name[mod]][0] = round(acc, 2)
        if acc > Acc[Mod_name[mod]][1]:
            Acc[Mod_name[mod]][1] = round(acc, 2)
        if Mod_name[mod] == "RandomForest":
            tp = [col_train[i] for i in range(Nft) if mod.feature_importances_[i] > 0.05]
            for name in tp:
                if name not in col_train2:
                    col_train2.append(name)
print Acc
print col_train2
```

    {'Logit': [0.97, 0.98], 'RandomForest': [1.0, 1.0], 'XGBoost': [1.0, 1.0], 'K-NN': [1.0, 1.0], 'Bayes': [0.6, 0.64]}
    ['postedTime', 'klout_score']


# Text Analysis

When dealing with online data such as tweets, text mining is of course very useful and perhaps the most valuable part of the analysis. For Natural Language Processing (NLP), we may use the Python library nltk that includes basic tools such as tokenization to break up the text in words and symbols, stemming to reduce te text by converting words into their root form.

We start by looking at the summaries contained in in both DataFrames pdA and pdT. We first redefine a dataframe with all observations as well as a feature 'source' indicating which dataframe the observation is coming from.


```python
pdA['source'] =  [0 for i in range(len(pdA.index))]
pdM['source'] =  [1 for i in range(len(pdM.index))]

pdTt = pd.concat([pdA, pdM], axis=0).reset_index()
pdTt = pdTt.iloc[:, 1:]
pd.set_option('display.max_columns', 50)
pdTt.iloc[:2, :]
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>body</th>
      <th>favoritesCount</th>
      <th>fol_body</th>
      <th>fol_folCount</th>
      <th>fol_hashtags</th>
      <th>fol_id</th>
      <th>fol_location</th>
      <th>fol_name</th>
      <th>fol_summary</th>
      <th>fol_type</th>
      <th>followersCount</th>
      <th>friendsCount</th>
      <th>generator</th>
      <th>geo</th>
      <th>id</th>
      <th>image</th>
      <th>klout_score</th>
      <th>languages</th>
      <th>listedCount</th>
      <th>location</th>
      <th>name</th>
      <th>postedTime</th>
      <th>preferredUsername</th>
      <th>provider</th>
      <th>retweetCount</th>
      <th>sharetype</th>
      <th>statusesCount</th>
      <th>summary</th>
      <th>twitterTimeZone</th>
      <th>url</th>
      <th>user_mentions</th>
      <th>verb</th>
      <th>source</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>https://t.co/EABi9mVIG0</td>
      <td>0</td>
      <td></td>
      <td></td>
      <td></td>
      <td>726293037647716353</td>
      <td></td>
      <td></td>
      <td>https://t.co/EABi9mVIG0</td>
      <td></td>
      <td>1299</td>
      <td>999</td>
      <td>Facebook</td>
      <td></td>
      <td>117696030</td>
      <td>https://pbs.twimg.com/profile_images/570657567...</td>
      <td>45</td>
      <td>en</td>
      <td>42</td>
      <td>Coimbatore, Bangalore, Chennai</td>
      <td>Sathish Kumar S</td>
      <td>2016-04-30T06:12:08.000Z</td>
      <td>satisonline</td>
      <td>Twitter</td>
      <td>0</td>
      <td></td>
      <td>6301</td>
      <td>Social Analytics &amp; Listening Professional | #I...</td>
      <td>New Delhi</td>
      <td>https://www.facebook.com/story.php?story_fbid=...</td>
      <td></td>
      <td>post</td>
      <td>0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>I am very happy ✔ I am very sad ✔ I am very em...</td>
      <td>0</td>
      <td></td>
      <td></td>
      <td></td>
      <td>726295324382363649</td>
      <td></td>
      <td></td>
      <td>I am very happy ✔ I am very sad ✔ I am very em...</td>
      <td></td>
      <td>1165</td>
      <td>618</td>
      <td>Twitter for Android</td>
      <td></td>
      <td>326939646</td>
      <td>https://pbs.twimg.com/profile_images/517215609...</td>
      <td>45</td>
      <td>en</td>
      <td>55</td>
      <td>Brighton, UK</td>
      <td>Barry Prendergast</td>
      <td>2016-04-30T06:21:13.000Z</td>
      <td>render_ghost</td>
      <td>Twitter</td>
      <td>0</td>
      <td></td>
      <td>3532</td>
      <td>I'm the @Brandwatch #Vizia design guy.\n\nOpin...</td>
      <td>London</td>
      <td>https://twitter.com/daveixd/status/72612063108...</td>
      <td></td>
      <td>post</td>
      <td>0</td>
    </tr>
  </tbody>
</table>
</div>



A first observation is that there are 4 different languages recorded in the tweets coming from Brandwatch Employees (plain and Brtish english, spanish and german), while there are 16 in total when looking at all observations. Two of them are actually the same since the variable 'languages' differentiates between plain english 'en' and british english 'en-gb'. I'm sure some would argue this is of the utmost importance, but after a few trials, there doesn't seem to be much of a difference between the two.


```python
import nltk
import string
from collections import Counter
import re
from nltk.corpus import stopwords
from nltk.stem.porter import *
stemmer = nltk.stem.PorterStemmer()

var = 'summary'

### exclude punctuation symbols from the text
exclude = set(string.punctuation)
### Observations decalred either in plain english or British english
df_en = pdTt[pdTt.languages == 'en']
df_gb = pdTt[pdTt.languages == 'en-gb']
### all the tweets from Brandwatch employees
nEN, nGB = len(df_en.index), len(df_gb.index)
entxt = "--".join([df_en.iloc[i, :][var].lower() for i in range(nEN)\
                           if df_en.iloc[i, :][var] != None\
                           and df_en.iloc[i, :][var].lower() not in exclude])
gbtxt = "--".join([df_gb.iloc[i, :][var].lower() for i in range(nGB)\
                           if df_gb.iloc[i, :][var] != None\
                           and df_gb.iloc[i, :][var].lower() not in exclude])
### kepp only alphanumeric characters
entxt = re.findall(r"[\w']+", entxt)
gbtxt = re.findall(r"[\w']+", gbtxt)
### tokenize the result to obtain only roots/general meaning of the previous words
# tokens_en, tokens_gb = entxt, gbtxt
tokens_en, tokens_gb = [], []
for word1, word2 in zip(entxt, gbtxt):
    for w1, w2 in zip(nltk.word_tokenize(word1), nltk.word_tokenize(word2)):
        tokens_en.append(w1)
        tokens_gb.append(w2)
### remove the obvious and very common words such as "the", "a", "co" (from web links) etc..
### stopwords.words('english') is a list of 137 such words
stop = stopwords.words('english') + ["http", "https", "'s", "'", "co"]
# stemmed_en = [stemmer.stem(w) for w in tokens_en]
stemmed_en = tokens_en
filtered_en = [w for w in stemmed_en if not w in stop]
# stemmed_gb = [stemmer.stem(w) for w in tokens_gb]
stemmed_gb = tokens_gb
filtered_gb = [w for w in stemmed_gb if not w in stop]

count_en = Counter(filtered_en)
count_gb = Counter(filtered_gb)

filtered_en = tb(" ".join(filtered_en))
filtered_gb = tb(" ".join(filtered_gb))

print count_en.most_common(10)
print count_gb.most_common(10)
```

    [(u'brandwatch', 179), (u'social', 64), (u'opinions', 51), (u'geek', 41), (u'tech', 40), (u'pr', 35), (u'scribbler', 33), (u'bod', 33), (u'telegraph', 33), (u'calmzine', 33)]
    [(u'brandwatch', 159), (u'fan', 76), (u'manager', 75), (u'someone', 40), (u'closet', 40), (u'mummy', 40), (u'pr', 40), (u'thoughts', 40), (u'else', 40), (u'stolen', 40)]


It planly appears that counting words only is not very informative. Another approach would consist in forming several documents and compare their inner frequency to one another.

## The TF-IDF measure

To compare the documents, we can for instance use the Inverse Document Frequency, or IDF, and have it mulitplied with the Term Frequency, or TF, to obtain the TF-IDF.

The idea behind TF is straightforward: plain count of words. But the one behind IDF a little less so. If a word is frequent in a document at hand, it might well represent what the text within the document is all about. But if that same word is frequent in the sense that it is also contained in many other documents, then it is perhaps not a very good unique identifier of a classifier. IDF is an inverse frequency, the more rare among every documents, the higher the score. Whereas when a word is common to many documents IDF helps us penalizes a word's frequency. For instance, the words "brandwatch" and "rt" (from @RT) clearly are the ones with the highest frequencies. But they also appear in most if not every document. Therefore their IDF score might be extremely low and we should expect not to see them appear in every document's high TF-IDF scores.

For ease of implementation, we introduce the TextBlob library along with the required, TF, IDF and TF-IDF associated functions:


```python
import math
from textblob import TextBlob as tb

def tf(word, blob):
    return float(blob.words.count(word)) / len(blob.words)

def n_containing(word, bloblist):
    return sum(1 for blob in bloblist if word in blob.words)

def idf(word, bloblist):
    return math.log(float(len(bloblist)) / (1 + n_containing(word, bloblist)))

def tfidf(word, blob, bloblist):
    return tf(word, blob) * idf(word, bloblist)
```

We now may use the previous functions, apply them to every word in both our EN, EN-GB documents and order them y increasing TF-IDF values.


```python
bloblist = [filtered_en, filtered_gb]
separate_name = ["Plain English", "British English"]
for i, blob in enumerate(bloblist):
    print("Top words in {}:".format(separate_name[i]))
    scores = {word: tfidf(word, blob, bloblist) for word in blob.words}
    sorted_words = sorted(scores.items(), key=lambda x: x[1], reverse=True)
    for word, score in sorted_words[:15]:
        print("\tWord: {}, TF-IDF: {}".format(word, score))
```

    Top words in Plain English:
    	Word: code, TF-IDF: 0.0
    	Word: tweeter, TF-IDF: 0.0
    	Word: skate, TF-IDF: 0.0
    	Word: go, TF-IDF: 0.0
    	Word: tv, TF-IDF: 0.0
    	Word: tweet, TF-IDF: 0.0
    	Word: opportunities, TF-IDF: 0.0
    	Word: veg, TF-IDF: 0.0
    	Word: program, TF-IDF: 0.0
    	Word: permanently, TF-IDF: 0.0
    	Word: updated, TF-IDF: 0.0
    	Word: woman, TF-IDF: 0.0
    	Word: dell, TF-IDF: 0.0
    	Word: synthesizers, TF-IDF: 0.0
    	Word: cmo, TF-IDF: 0.0
    Top words in British English:
    	Word: removals, TF-IDF: 0.0
    	Word: help, TF-IDF: 0.0
    	Word: messi, TF-IDF: 0.0
    	Word: years, TF-IDF: 0.0
    	Word: london, TF-IDF: 0.0
    	Word: tuesdays, TF-IDF: 0.0
    	Word: find, TF-IDF: 0.0
    	Word: weekly, TF-IDF: 0.0
    	Word: enthusiast, TF-IDF: 0.0
    	Word: web, TF-IDF: 0.0
    	Word: avid, TF-IDF: 0.0
    	Word: based, TF-IDF: 0.0
    	Word: hills, TF-IDF: 0.0
    	Word: la, TF-IDF: 0.0
    	Word: socialmedia, TF-IDF: 0.0


Something quite interesting happens. The highest TF-IDF scores are all 0.0. This means that all worse score were actually negative... And based on the above definitions, this can only happen if the IDF score is negative, that is if a word appears in every document of the list. In our case we only have 2 documents, so an IDF score is either negative 0 (assuming that nothing went wrong in the pre-processing and that some terms were listed in 0 document). Here things seem to have worked out well since the best scores are all 0s, and must thus well represent their document as unique identifier.

Perhaps a little disappointingly, the only differences we spot here have more to do with people' interest than linguistic. Remember from the maps that the US areas covered where mostly in New-York, Chicago and the Bay area, where people are for instance concerned with their health and look, which may explain why we find the keyword "veg" there. In the UK, football (or soccer depending on which side of the pond you stands on) is very popular while still not as much as Basketball, NFL football or Baseball in the US. This may explain why we find the keyword "messi" in the UK, as a frequent one in the UK that never appeared the US area. There are a few more oddities, such as the word "woman" that apparently doesn't appear in the UK area. The UK can cheer up though, as the keyword "enthusiast" doesn't appear in the US area.

A word of caution though: the variable 'languages' had missing entries, and some might still be wrongly set. For instance even though I am French, I usually set my profiles to English and will sometimes use US or UK english, rather randomly, depending on my mood. Something we may not exclude from the average user's behaviour.

## On the source again

We turn back to trying to spot differences between the two initial datasets that come either fro Employees or people mentioning the company. So as before, we first create a list of TextBlob documents which are separated by the nature of the 'source' variable we previously introduced.


```python
import nltk
import string
from collections import Counter
import re
from nltk.corpus import stopwords
from nltk.stem.porter import *
stemmer = nltk.stem.PorterStemmer()

def is_ascii(s):
    return all(ord(c) < 128 for c in s)

var, foc, doc = 'body', 'source', []
separation = np.unique(pdTt[foc])

### exclude punctuation symbols from the text
exclude = set(string.punctuation)
### Extract the observations according to the foc variable, making a document
for val in separation:
    ind = [i for i in range(len(pdTt.index)) if pdTt.iloc[i, :][foc] == val]
    df = pdTt.iloc[ind, :]
    n = len(df.index)
    ### joining all the text in the var feature
    txt = "--".join([df.iloc[i, :][var].lower() for i in range(n)\
                           if df.iloc[i, :][var] != None\
                           and df.iloc[i, :][var].lower() not in exclude])
    ### keep only alphanumeric characters
    txt = re.findall(r"[\w']+", txt)
    ### tokenize the result to obtain only roots/general meaning of the previous words
    # tokens = txt
    tokens = []
    for word in txt:
        for w in nltk.word_tokenize(word):
            tokens.append(w)
    ### remove the obvious and very common words such as "the", "a", "co" (from web links)
    ### stopwords.words('english') is a list of 137 such words
    stop = stopwords.words('english') + ["http", "https", "'s", "'", "co"]
    stemmed = tokens
    # stemmed = [stemmer.stem(w) for w in tokens]
    filtered = [w for w in stemmed if not w in stop and is_ascii(w)]
    filtered = tb(" ".join(filtered))
    doc.append(filtered)
```

We then sift through the resulting documents and order them by increasing TF-IDF values.


```python
bloblist = doc
separate_name = ["Employee", "Mention"]
for i, blob in enumerate(doc):
    print("Top words within the {} dataset:".format(separate_name[i]))
    scores = {word: tfidf(word, blob, bloblist) for word in blob.words}
    sorted_words = sorted(scores.items(), key=lambda x: x[1], reverse=True)
    for word, score in sorted_words[:3]:
        print("\tWord: {}, TF-IDF: {}".format(word, score))
```

    Top words within the Employee dataset:
    	Word: consolidatednws, TF-IDF: 0.0
    	Word: ionf6k7bq2, TF-IDF: 0.0
    	Word: ydyvq0fs1r, TF-IDF: 0.0
    Top words within the Mention dataset:
    	Word: titans, TF-IDF: 0.0
    	Word: devenir, TF-IDF: 0.0
    	Word: d2xxx3t5xl, TF-IDF: 0.0


This time again, all the highest score are 0, an indication that the pre-processing went well.

Yet, the results may seem rather disappointing at first glance. What could the unique identifier "ionf6k7bq2" mean? The good news is that Google is our friend and knows better than to be easily fooled. The string "ionf6k7bq2" is in fact coming from a Twitter link and the first result on the most popular search engine is a mention on Brandwatch's Twitter account to the SaveOurNHS hashtag.


```python
from PIL import Image
img = Image.open('BW-doc.png')
img
```




![png](output_108_0.png)



As it turns out, the string 'ionf6k7bq2' is a Twiiter link that refers users to an article on Brandwatch's blog:


```python
from IPython.display import HTML
HTML('<iframe src = https://www.brandwatch.com/2016/02/react-twitters-most-influential\
            -junior-doctors-speak-out/?utm_content=buffer62c8c&utm_medium=social&utm_source\
            =react-twitter&utm_campaign=owned+social width = 800 height = 600></iframe>')
```




<iframe src = https://www.brandwatch.com/2016/02/react-twitters-most-influential            -junior-doctors-speak-out/?utm_content=buffer62c8c&utm_medium=social&utm_source            =react-twitter&utm_campaign=owned+social width = 800 height = 600></iframe>



## Organizing documents by Time Zones

Another interesing way to split the data is to do it by Time Zones. TF-IDF is particularly well adapted to finding a region's specificities as unique words identifiers. Let us first split the data accordingly and sift through the database one last time.


```python
import nltk
import string
from collections import Counter
import re
from nltk.corpus import stopwords
from nltk.stem.porter import *
stemmer = nltk.stem.PorterStemmer()

var, foc, doc, TZ, n, lim = 'body', 'twitterTimeZone', [], [], len(pdTt.index), 50
for tmp in np.unique(pdTt[foc]):
    nb = len([x for x in pdTt[foc] if x == tmp])
    if nb > lim and tmp != None:
        TZ.append(tmp)

### exclude punctuation symbols from the text
exclude = set(string.punctuation)
### Extract the observations according to the foc variable, making a document
for tz in TZ:
    ### joining all the text in the var feature
    ind = [i for i in range(n) if pdTt.iloc[i, :][foc] == tz]
    df = pdTt.iloc[ind, :]
    ndf = len(df.index)
    ### joining all the text in the var feature
    txt = "--".join([df.iloc[i, :][var].lower() for i in range(ndf)\
                           if df.iloc[i, :][var] != None\
                           and df.iloc[i, :][var].lower() not in exclude])
    ### keep only alphanumeric characters
    txt = re.findall(r"[\w']+", txt)
    ### tokenize the result to obtain only roots/general meaning of the previous words
    # tokens = txt
    tokens = []
    for word in txt:
        for w in nltk.word_tokenize(word):
            tokens.append(w)
    ### remove the obvious and very common words such as "the", "a", "co" (from web links)
    ### as well as a few other obvious foreign ones ("zum", "die", "avec", "sur", "de")
    ### stopwords.words('english') is a list of 137 such words
    stop = stopwords.words('english') + ["http", "https", "'s", "'", "co",\
                                         "zum", "die", "avec", "sur", "de"]
    stemmed = tokens
    # stemmed = [stemmer.stem(w) for w in tokens]
    filtered = [w for w in stemmed if not w in stop and is_ascii(w)]
    filtered = tb(" ".join(filtered))
    doc.append(filtered)
```


```python
bloblist = doc
for i, blob in enumerate(bloblist):
    print("Top words in Time Zone {}".format(TZ[i]))
    scores = {word: tfidf(word, blob, bloblist) for word in blob.words}
    sorted_words = sorted(scores.items(), key=lambda x: x[1], reverse=True)
    for word, score in sorted_words[:3]:
        print("\tWord: {}, TF-IDF: {}".format(word, score))
```

    Top words in Time Zone Amsterdam
    	Word: brighton, TF-IDF: 0.00799795754434
    	Word: grunandy, TF-IDF: 0.00377742685009
    	Word: job, TF-IDF: 0.00370417878103
    Top words in Time Zone Athens
    	Word: uyv27kamqz, TF-IDF: 0.00775653449882
    	Word: bw_chat, TF-IDF: 0.00731358420109
    	Word: expertise, TF-IDF: 0.00505321060522
    Top words in Time Zone Berlin
    	Word: brandwatchde, TF-IDF: 0.0113074306357
    	Word: von, TF-IDF: 0.00920282493562
    	Word: smwhh, TF-IDF: 0.00600184234932
    Top words in Time Zone Central Time (US & Canada)
    	Word: e1owksg7sl, TF-IDF: 0.045942550493
    	Word: cigarchat, TF-IDF: 0.0139825153674
    	Word: daily, TF-IDF: 0.00683197453899
    Top words in Time Zone Dublin
    	Word: producthunt, TF-IDF: 0.00408455806055
    	Word: investors, TF-IDF: 0.00408455806055
    	Word: startupturkey, TF-IDF: 0.00408455806055
    Top words in Time Zone Eastern Time (US & Canada)
    	Word: kxd4fnwohk, TF-IDF: 0.00499730075089
    	Word: bw_chat, TF-IDF: 0.00370099997259
    	Word: sxsw2016, TF-IDF: 0.00318010047784
    Top words in Time Zone London
    	Word: brighton, TF-IDF: 0.00745703569237
    	Word: job, TF-IDF: 0.00454583770011
    	Word: engineer, TF-IDF: 0.00207949326626
    Top words in Time Zone New Delhi
    	Word: stories, TF-IDF: 0.00585335566407
    	Word: unconnected, TF-IDF: 0.00501826013075
    	Word: djr4ktg0gg, TF-IDF: 0.00501826013075
    Top words in Time Zone Pacific Time (US & Canada)
    	Word: learn, TF-IDF: 0.00262792510445
    	Word: bw_chat, TF-IDF: 0.00224955423127
    	Word: brands, TF-IDF: 0.00178954887033
    Top words in Time Zone Paris
    	Word: brandwatchfr, TF-IDF: 0.0173345608276
    	Word: le, TF-IDF: 0.00879868077743
    	Word: pour, TF-IDF: 0.00766293365443
    Top words in Time Zone Quito
    	Word: e5oiw5oyce, TF-IDF: 0.0768995480355
    	Word: e5oiw5ga0o, TF-IDF: 0.0205065461428
    	Word: 1xv9lo0dlo, TF-IDF: 0.0051266365357
    Top words in Time Zone Singapore
    	Word: msmw2016, TF-IDF: 0.0169996154576
    	Word: anntran, TF-IDF: 0.0135996923661
    	Word: ladyxtel, TF-IDF: 0.0132898480054


Now the results are more interesting as the TF-IDF metric penalizes the term's frequencies by the log-inverse number of other documents they appear in. This smooths out the terms that would be very particular to a given Time Zone by giving a chance to terms perhaps less common but which also appear elsewhere. This is not perfect however, as we see for instance that the german preposition "von" appears in the Berlin Time Zone.

Of course we knew where the data comes from already, but now we know more of its nature. Many of the above keywords are concerned with social media events such as "bw_chat" (found as the first result in PT, 3rd in ET zones and second in Athens) to gather people with a common interest in Data Science and Analysis. Other keywords suggest that a lot of posts are dedicated to job openings and searches. In the London Time Zone, Brandwatch's headquarters, the keywords "job" and "engineer" are even respectively 2nd and 3rd results, so the data was presumably retrieved at a time when Brandwatch was actively recruiting engineers in Brighton.

Some other keywords directly come from an influencer's profile, such as @GrunAndy that is the 2nd best result in Amsterdam:


```python
from PIL import Image
img = Image.open('grunandy.png')
img
```




![png](output_116_0.png)



Others still are from Twitter links posted on an influential user, or a popular hastag such as #cigarchat, or even web links to social events or festivals:


```python
from IPython.display import HTML
HTML('<iframe src = http://www.sxsw.com width = 800 height = 600></iframe>')
```




<iframe src = http://www.sxsw.com width = 800 height = 600></iframe>



# Conclusion

First of all, as is often the case at the end of a new project, I feel I have learned a lot from a technical standpoint. In that case, I learned how to untar an archive within Python, how to read a JSON, even though I quickly decided to switch to pandas DataFrame which I feel offer a more readable and pleasant organization of the data, making it easier to then design models.

I also learned more about Brandwatch itself after discovering popular links and events in the data. I first though I would try to augment the data by sifting through Twitter's API, and maybe use it to obtain labelled sentiment data. But after a while it seemed to me the data at hand was enough of a challenge already. The only addition to the data I used was to look for geographical coordinates with the GoogleMaps API so I could plot some interesting map as I feel that a first step in understanding the data is often to visualize it.

I realize that the modelling analysis part is somewhat basic and standard, as is probably the text mining part, but I also feel that this project represents enough work already to vouch for my ability to tackle lengthy tasks and become an asset for Brandwatch.


```python

```
