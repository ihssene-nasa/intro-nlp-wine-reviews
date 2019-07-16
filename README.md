# Intro to NLP - Predict the Quality and Price of a Wine from a Wine Experts Description. Then deploy the model via Azure Functions.
A project to introduce you to a simple Bag of Words NLP using SciKit Learns and Python. You can use this same logic for document classification or any text classification problem you may be trying to solve.

## Prerequisites
There are a few different ways to follow along on this tutorial:
1. Create an [Azure account](https://azure.microsoft.com/en-us/free/) and [Create Workspace](https://docs.microsoft.com/en-us/azure/machine-learning/service/quickstart-run-cloud-notebook) to use the Notebook VMs. This gives you a LOT of functionality and I would highly recommend this for models you plan to put in production.
2. [Azure Notebooks](https://notebooks.azure.com/) - an online Jupyter notebook that makes it easy to share and access your notebook from anywhere.
3. [Download Jupyter](https://jupyter.org/) notebooks and run it locally. The notebook is included in the source for this tutorial.

Once you are set with one of the above notebook environment configurations its time to start building!

## Import packages and data
### 1. Import the Packages
```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
import math
from sklearn.model_selection import train_test_split
from sklearn.svm import LinearSVC, SVC
from sklearn.calibration import CalibratedClassifierCV
from sklearn.metrics import precision_recall_curve
from sklearn.linear_model import LogisticRegression
from sklearn.feature_extraction.text import CountVectorizer
```

### 2. We need Data!
![data](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcSwvLv12Qt9SOXvdGwlqQP0ORHhvO1OI7hAxqAvXbf3tpRl4t2Isw)
1. I used a dataset I found on Kaggle. Kaggle is an online community of data scientists. 
2. Download the dataset from this repo or kaggle.
* [Wine Dataset from Repo](dataset/winemag-review.csv)
* [Kaggle Dataset](https://www.kaggle.com/zynicide/wine-reviews)

3. Import the data as a [Pandas](https://pandas.pydata.org/pandas-docs/stable/) DataFrame
```python
#File path to the csv file
csv_file = r"C:\path-to-your-file\winemag-review.csv"

# Read csv file into dataframe
df = pd.read_csv(csv_file)

# Print first 5 rows in the dataframe
df.head()
```
## Visualize the data
Once we have the data then its time to analyize it and do some [Feature Selection and Engineering](https://docs.microsoft.com/en-us/azure/machine-learning/team-data-science-process/create-features). We will visualize our data using [Seaborn](https://seaborn.pydata.org/). This will allow us to see if there is a strong correlation between different data points and help us answer questions about our data. Since our initial question was around predicting `price` or `points` from the `description` we already know that our `feature` will be the `description` and our `label` will be `price` or `points`. 

For fun, lets ask some questions about the data and answer them by graphing it with Seaborn.

### 1. Is there a correlation between price and points?
```python
sns.barplot(x = 'points', y = 'price', data = df)
```
![graph](\imgs\priceandpoints.PNG)

```python
sns.boxplot(x = 'points', y = 'price', data = df)
```
![graph](\imgs\priceandpoints2.PNG)

### 2. Does one wine critic give higher ratings than the others?

```python
sns.catplot(x = 'points', y = 'taster_name', data = df)
```
![graph](\imgs\tasterpoints.PNG)

### 3. Lets look at a WordCloud of the `description` Text

```python
from wordcloud import WordCloud, STOPWORDS
import matplotlib.pyplot as plt
text = df.description.values
wordcloud = WordCloud(
    width = 3000,
    height = 2000,
    background_color = 'black',
    stopwords = STOPWORDS).generate(str(text))
fig = plt.figure(
    figsize = (40, 30),
    facecolor = 'k',
    edgecolor = 'k')
plt.imshow(wordcloud, interpolation = 'bilinear')
plt.axis('off')
plt.tight_layout(pad=0)
plt.show()
```
![graph](\imgs\wordcloud.PNG)
<small>I like to think of this WordCloud as a cheatsheet of discriptive words to use when tasting wine to make yourself sound like a wine expert :D</small>


### What other questions could you ask and answer by graphing this data?
![graph](\imgs\dfhead.PNG)

## Create Calculated Columns for Labels

We are going to do a multi-classification for the price and points of the wines reviewed by the wine critics. Right now our points and price are a number feature*. We are going to create a couple functions to generate calculated columns based on the values in the points and price columns to use are our labels.

![graph](\imgs\dfinfo.PNG)

<small>*NOTE: if we wanted to predict a specific price or point value we would want to build a regression model not a multi-classification. It really just depends on what your goal is</small>
### Create Quality column from points of bad, ok, good, great.

### 1. Function to return string quality based on points value.

```python
def getQuality(points):
    if(points <= 85):
        return 'bad'
    elif(points<=90 ):
        return 'ok'
    elif(points<=95):
        return 'good'
    elif(points<=100):
        return 'great'
    else:
        return 'If this gets hit, we did something wrong!'
```

### 2. Next lets apply the function to the points column of the dataframe and add a new column named `quality`.

```python
df['quality'] = df['points'].apply(getQuality)
```
### 3. Lets visualize our new column against the price column like we did above.

```python
sns.catplot(x = 'quality', y = 'price', data = df)
```
![graph](\imgs\pricequality.PNG)


```python
sns.barplot(x = 'quality', y = 'price', data = df)
```
![graph](\imgs\pricequality2.PNG)

we now have quality buckets based on the points to use as a label class for our multi-classification model.

### Create priceRange column from price column of `1-30`, `31-50`, `51-100`, `Above 100` and `0` for columns with NaN.

### 1. Function to return string priceRange based on price value.

```python
def getPriceRange(price):
    if(price <= 30):
        return '1-30'
    elif(price<=50):
        return '31-50'
    elif(price<=100): 
        return '51-100'
    elif(math.isnan(price)):
        return '0'
    else:
        return 'Above 100'
```
### 2. Next lets apply the function to the points column of the dataframe and add a new column named `quality`.

```python
df['priceRange'] = df['price'].apply(getPriceRange)
```

### 3. Print totals for each priceRange assigned to see how the labels are distributed

```python
df.groupby(df['priceRange']).size()
```
---
```python
priceRange
0             8996
1-30         73455
31-50        27746
51-100       16408
Above 100     3366
dtype: int64
```
## We now have our labels for both models to predict quality and priceRange. Next we need to take our description text and process NLP with the library SciKitLearn to create a Bag-of-Words using the [CountVectorizer](https://scikit-learn.org/stable/modules/generated/sklearn.feature_extraction.text.CountVectorizer.html) functionality.

The docs do a great job of explaining the CountVectorizer. I recommend reading through them to get a full understanding of whats going on, however I will go over some of the basics here.

At a high level the CountVectorizer is taking the text of the description, removing stop words (such as “the”, “a”, “an”, “in”), creating a tokenization of the words and then creating a vector of numbers that represents the description. The text description is now represented as numbers with only the words we care about and can be processed by the computer to train a model. Remember the computer understand numbers and words can be represented as numbers so the computer can "understand".

Before we jump into the CountVectorizer code and functionality. I want to list out a some terms and point out that CountVectorizer _does not_ do the Lemmetiization or Stemming for you.

TODO: get descriptions for words
* StopWords:
* N-Gram:
* Lemmetization:
* Stemming:

*NOTE: CountVectorizer doesn't do all of these things for you but does enough for simple models like this.

Lets take a look at how we do this now.

These are all the properties that you can set within the CountVectorizer. Many of them are defaulted or if set override other parts of the CountVectorizer. We are going to leave most of the defaults and then play with changing some of them to get better results for our model.

```python
CountVectorizer(input=’content’, encoding=’utf-8’, decode_error=’strict’, strip_accents=None, lowercase=True, preprocessor=None, tokenizer=None, stop_words=None, token_pattern='(?u)\b\w\w+\b', ngram_range=(1, 1), analyzer='word', max_df=1.0, min_df=1, max_features=None, vocabulary=None, binary=False, dtype=<class 'numpy.int64'>)
```
## Create the function to get the vector and vectorizer from the `description` feature.

### 1. There are different CountVectorizer configurations commented out so that we can play with different configs and see how it changes our result. Additionally this will help us look at one description and pick apart what is actually happening in the CountVectorizer.

```python
def get_vector_feature_matrix(description):
    vectorizer = CountVectorizer(lowercase=True, stop_words="english", max_features=5)
    #vectorizer = CountVectorizer(lowercase=True, stop_words="english")
    #vectorizer = CountVectorizer(lowercase=True, stop_words="english",ngram_range=(1, 2), max_features=20)

    #vectorizer = CountVectorizer(lowercase=True, stop_words="english", tokenizer=stemming_tokenizer) 
    vector = vectorizer.fit_transform(np.array(description))
    return vector, vectorizer
```

### 2. For the first run we are going to have the below config. What this is saying is that we want to convert the text to lowercase, remove the english stopwords and we only want 5 words as feature tokens.

```python
vectorizer = CountVectorizer(lowercase=True, stop_words="english", max_features=5)
```

### 3. Next lets call our function and pass in the description column from the dataframe. 

This returns the `vector` and the `vectorizer`. The `vectorizer` is what we apply to our text to create the number `vector` representation of our text so that the machine learning model can learn. Later we will save our `vectorizer` to a file so that it can be used again and again to create on next text data to classify data once we have our candidate model.

```python
vector, vectorizer = get_vector_feature_matrix(df['description'])
```
If we print the vectorizer we can see the current default parametners for it.

```python
print(vectorizer)
```
---
```python
CountVectorizer(analyzer='word', binary=False, decode_error='strict',
        dtype=<class 'numpy.int64'>, encoding='utf-8', input='content',
        lowercase=True, max_df=1.0, max_features=5, min_df=1,
        ngram_range=(1, 1), preprocessor=None, stop_words='english',
        strip_accents=None, token_pattern='(?u)\\b\\w\\w+\\b',
        tokenizer=None, vocabulary=None)
```
### 4. Lets examine our variables and data to understand whats happening here.

```python
print(vectorizer.get_feature_names())
```
---
```python
['aromas', 'flavors', 'fruit', 'palate', 'wine']
```
Here we are getting the features of the vectorizer. Because we told the CountVectorizer to have a `max_feature = 5` it will build a vocabulary that only consider the top feature words ordered by term frequency across the corpus. This means that our `description` vectors would _only_ include these words when they are tokenized, all the other words would be ignored.

Lets print out our first `description` and first `vector` to see this represented.

```python
print(vector.toarray()[0])
```
---
```python
[1 0 1 1 0]
```

```python
df['description'].iloc[0]
```
---
```python
"_Aromas_ include tropical _fruit_, broom, brimstone and dried herb. The _palate_ isn't overly expressive, offering unripened apple, citrus and dried sage alongside brisk acidity."
```

The vector array (`[1 0 1 1 0]`) that represents the vectorization features (`['aromas', 'flavors', 'fruit', 'palate', 'wine']`) in first description in the corpus. 1 indicates its present and 0 indicates not present in the order of the vectorization features.

Play around with different indexes of the vector and description. You will notice that there isn't lemmitzation so words like `fruity` and `fruits` are being ignored since only `fruit` is included in the vector and we didn't lemmitize the description to transform them into their root word.

Each feature word is assigned a number so when w






```python
print(vectorizer.vocabulary_)
```
---
```python
{'aromas': 0, 'fruit': 2, 'palate': 3, 'wine': 4, 'flavors': 1}
```





TODO: I wonder if we had an additional feautre of the type of blend? Could that improve accuracy?

TODO: build an app where you can type a description of a wine and it will predict the wuality and price of the wine. Host http request on azure functions in a vue mobile web app.