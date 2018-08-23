---
title: Better NPS Analysis Using NLTK
layout: post
---

Like many companies, we regularly ask our customers for feedback in the form of NPS surveys after their travels. However, we have an additional challenge in that we receive thousands of comments a day in 11 different languages which need to be interpreted by Category Managers (CMs) and Sales Managers (SMs) who are very unlikely to be able to read that language (our CMs/SMs come from all over the world, not many speak Finnish). So we set up a small translation and feature pipeline in order to help our CMs get to the root cause of customer problems faster and more easily identify common trends and patterns.

The starting point of this data pipe is to get all comments into English, which is already a huge time savings for our team:
```python
from google.cloud import translate

from bibrary.db import run_query

os.environ['GOOGLE_APPLICATION_CREDENTIALS'] = "/path/to/client_secrets.json"
target_language = 'en'
translate_client = translate.Client()

records = run_query("select order_id, comment from {} where translated_comment is not null")
translated = []
for row in records:
    translated.append({'order_id': row[0],
                       'comment': row[1],
                       'translated_comment': translate_client.translate(row[1])['translatedText']})
```

While the translations are useful, we also want to enrich this with some information about customer sentiment in order to quickly spot unexpected deviations (for example, receiving a high satisfaction score but using negative language might indicate specific frustrations). For the sake of simplicity, we use the NLTK vader sentiment library:
```python
from nltk.sentiment.vader import SentimentIntensityAnalyzer

def add_sentiment(text):
    sid = SentimentIntensityAnalyzer()
    scores = sid.polarity_scores(text)
    res = dict()
    res['overall_sentiment'] = scores['compound']
    res['positive_sentiment'] = scores['pos']
    res['neutral_sentiment'] = scores['neu']
    res['negative_sentiment'] = scores['neg']
    return res

for row in records:
    row.update(add_sentiment(row['translated_comment']))
```

At this point, our CMs already have sufficient data to begin doing analysis much quicker than they historically would have been able. But with a little bit more work, we can also estimate the likely cause for issue (or joy) that the customer told us about, allowing us to easily spot patterns without needing to read thousands of remarks. To do so, we use a combination of TF-IDF and some basic matrix factorization to determine topics, then score each comment for the relative weight of each comment:

```python
import numpy as np

from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.decomposition import NMF

def train_topics(training_docs):
    tfidf_vectorizer = TfidfVectorizer(stop_words='english')
    tf_idf = tfidf_vectorizer.fit_transform(training_docs)
    nmf = NMF(n_components=30, random_state=1337,
              alpha=0.1, l1_ratio=0.5).fit(tf_idf)
    tfidf_feature_names = tfidf_vectorizer.get_feature_names()
    return nmf, tfidf_vectorizer, tfidf_feature_names


def topic_scores(nmf_model, tfidf_vectorizer, text):
    text_vals = nmf_model.transform(tfidf_vectorizer.transform([text]))[0]
    total_wt = np.sum(text_vals)
    res = dict()
    for idx, val in enumerate(text_vals):
        res['topic_{}'.format(idx)] = np.sum(val) / total_wt
    return res

nmf, tf_vector, tf_features = train_topics([x['translated_comment'] for x in records])
for row in records:
    row.update(topic_scores(nmf, tf_vector, row['translated_comment']))
```

At this point, we have now added a score ranging from 0-1 for all 30 topics to each NPS record, giving a reasonable indication of what the person's comment was about. However, because these topics are simply integer labels (ex. "topic_14"), we need to provide some additional label data about each topic to allow the CM to interpret what each one is about. To do so, we pull out the 20 most significant terms for each topic and attach a significance weight to the term, allowing a human translation of the machine score: 

```python
import json

def topic_keywords(nmf, feature_names, n_top_words=20):
    res = []
    for topic_idx, topic in enumerate(nmf.components_):
        features = [(feature_names[i], np.sum(topic[i]))
                    for i in topic.argsort()[:-n_top_words - 1:-1]]
        for f in features:
            out = {'topic_id': topic_idx,
                   'word': f[0],
                   'word_weight': f[1]}
            res.append(out)
    return res


for row in topic_keywords(nmf, tf_features):
    print(json.dumps(row))
```

All of this data can then easily be loaded into the data warehouse for reporting and analysis in the CM's favorite analysis tool, Tableau. As an example of how much easier this makes their job, here is an actual record before and after processing:

**Before:** 8/10 _Ihan ok matka, lennot hyvät kuten myös hotelli. Hotellissa hyvä aamupala. Hotellin respassa epäystävällinen aasialainen nais virkailija joka ei oikein sovellu asiakaspalveluun._

---

**After:** 8/10: _Okay ok, flights well as the hotel. The hotel has a good breakfast. The hotel is hosted by an unfriendly Asian female clerk who is not well suited for customer service._
* **Sentiment:** 0.77 - 29.7% **Positive**, 7% **Negative**
* **Key Topics:** Hotel Quality (37%), Hotel Service (27%)