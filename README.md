# Python web scraping for consumer industry insights

## Project Highlights  
  
Curious about how to track what people are saying online? Then check out this Python web scraping toolkit. This project provides a comprehensive implementation of the Selenium library to automatically gather data from the web and monitor consumer sentiment trends in the activewear industry.
<br>
<br>

In this Python project, you will explore the following key areas:  

1. **Real-world application:** Analyze consumer feedback from top activewear brands.  
2. **Comprehensive toolkit:** Leverage Selenium to efficiently scrape reviews and ratings.
3. **Sentiment Analysis:** Apply natural language processing to understand trends. 
4. **Data visualization:** Visualize and interpret consumer sentiments. 

<br>

### Getting Started  
  
Ensure you have Python installed on your machine. You will also need to install the following libraries:  
```bash
pip install pandas numpy nltk re tqdm time matplotlib selenium   
```

Library imports
```bash
import pandas as pd
import numpy as np
import nltk
import re
from tqdm import tqdm
import time
import matplotlib.pyplot as plt
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By   
```

Initialize translator (if required)
```bash
from deep_translator import GoogleTranslator

# Use any translator you like, in this example Google Translator
# from deep_translator import (GoogleTranslator,
#                              ChatGptTranslator,
#                              MicrosoftTranslator,
#                              PonsTranslator,
#                              LingueeTranslator,
#                              DeeplTranslator)
```

Initialize Chrome WebDriver
<br>
Consult the following page for chromedriver versions: https://developer.chrome.com/docs/chromedriver/downloads
```bash
# Specify the path to your Chrome driver
driver_path = r"..\chromedriver.exe"

# Set up Chrome options (optional)
chrome_options = Options()

# Initialize the Chrome WebDriver using the specified path
service = Service(driver_path)
browser = webdriver.Chrome(service=service, options=chrome_options)

# Open the specified URL
browser.get('https://www.trustpilot.com/review/www.adidas.com')
```

Subset reviews
```bash
subset = browser.find_element(By.CLASS_NAME,value="styles_reviewsContainer__3_GQw")
```

Initialize an empty dataframe to store results
```bash
results = pd.DataFrame()  
```

Fetch reviews
```bash
for i in tqdm(range(1, 50)):
    try:
        # Construct URL for each page
        if i == 1:
            website = "https://www.trustpilot.com/review/www.adidas.com?languages=all"
        else:
            website = f"https://www.trustpilot.com/review/www.adidas.com?languages=all&page={i}"
        
        browser.get(website)

        # Wait for page to load
        time.sleep(6)
        
        # Extract elements
        all_backup = [x.text for x in browser.find_elements(By.XPATH, '//*[contains(@data-service-review-card-paper, "true")]')]
        all_names = [x.text for x in browser.find_elements(By.XPATH, '//*[contains(@data-consumer-name-typography, "true")]')]
        all_nreviews = [x.text for x in browser.find_elements(By.XPATH, '//*[contains(@data-consumer-reviews-count-typography, "true")]')]
        all_titles = [x.text for x in browser.find_elements(By.XPATH, '//*[contains(@data-review-title-typography, "true")]')]
        all_reviews = [y for y in [x.text for x in browser.find_elements(By.XPATH, '//*[contains(@data-service-review-text-typography, "true")]')] if "Lorem ipsum dolor sit amet" not in y]
        all_countries = [x.text for x in browser.find_elements(By.XPATH, '//*[contains(@data-consumer-country-typography, "true")]')]
        all_dates = [x.text for x in browser.find_elements(By.XPATH, '//*[contains(@data-service-review-date-time-ago, "true")]')]
        all_exp_dates = [x.text for x in browser.find_elements(By.XPATH, '//*[contains(@data-service-review-date-of-experience-typography, "true")]')]
        all_stars = [y for y in [x.find_element(By.XPATH, "*").get_attribute("alt") for x in browser.find_elements(By.XPATH, '//*[contains(@class, "star-rating")]')] if "Rated" in y]
        
        # Check if all key lists have items (assuming 20 reviews per page as expected)
        if len(all_reviews) > 0:
            # Create DataFrame if there are reviews
            result = pd.DataFrame([all_names, all_nreviews, all_titles, all_reviews, all_countries, all_dates, all_exp_dates, all_stars, all_backup]).T
            result.columns = ["name", "nreviews", "title", "review", "country", "date", "experience_date", "rating", "backup"]
            results = pd.concat([results, result], ignore_index=True)
        else:
            print(f"No reviews on page {i}")

        # Break loop if fewer than 20 reviews are found, indicating the last page
        if len(all_backup) != 20:
            print(f"End of reviews at page {i}")
            break

    except Exception as e:
        print(f"Error on page {i}\n{e}")
```

Set-up final dataframe
```bash
merge = pd.concat([results, results2], axis=0, ignore_index=True)
```

Parsing and cleaning data
```bash
# Function to split the text
def split_text(text):
    if 'ago' in text:
        return text.split('ago', 1)  # Split at the first occurrence of 'ago'
    elif ', 202' in text:
        return text.split(', 202', 1)  # Split at the first occurrence of ',202'
    elif ', 201' in text:
        return text.split(', 201', 1)  # Split at the first occurrence of ',201'
    return [text, '']  # Return original text and empty string if neither is found

# Apply the function to split the text_column
merge[['part1', 'part2']] = merge['backup'].apply(split_text).apply(pd.Series)

# Split the 'backup' column after "Date of experience"
third_part = merge['part2'].str.split('Date of experience', n=1, expand=True)

# Assign column names to the resulting DataFrame  
third_part.columns = ['body', 'tail']  

# Combine with original DataFrame if needed
merge = pd.concat([merge, third_part['body']], axis=1)

# Parse rating column
merge['rating'] = merge['rating'].apply(lambda x: int(re.search(r'Rated (\d+)', x).group(1)))

# Parse date column: Use str.extract with a regex pattern to extract the date  
merge['experience_date_parsed'] = merge['experience_date'].str.extract(r'Date of experience: (.+)')  

#Drop columns
merge = merge.drop(columns=["part1", "part2", "experience_date"])
```

Use translator
```bash
translator = GoogleTranslator(source="auto", target="en")

# Define a function to translate text with checks for nan and length
def safe_translate(text):
    if isinstance(text, str) and len(text) <= 5000:
        return translator.translate(text)
    elif isinstance(text, float) and np.isnan(text):
        return text
    return text

# Translate columns
merge["body_v2"] = merge["body"].apply(safe_translate)
```

Sentiment analysis (HuggingFace Repo)
```bash
from transformers import pipeline

# Start by creating an instance of pipeline() specifying a task you want to use it for
# The pipeline() downloads and caches a default pretrained model and tokenizer for sentiment analysis
classifier = pipeline(task="sentiment-analysis", model="cardiffnlp/twitter-roberta-base-sentiment-latest", tokenizer="cardiffnlp/twitter-roberta-base-sentiment-latest")

# A short note on tokenization, stop-words, and any sort of pre-processing steps for text mining applications:
# Most pretrained models (like BERT, DistilBERT, and RoBERTa) are designed to handle special characters and punctuation effectively, so there is typically no need to remove special characters or stopwords, unless they interfere with the task (e.g., unnecessary noise).
# Stopwords (e.g., "the", "is", "and") are generally left in the text when using modern pretrained models like BERT or RoBERTa because the models are trained on text that includes them. Removing stopwords may disrupt the context, which is important for models that leverage context in sentence embeddings.

# For testing purposes create a subset (if required)
# search = merge['body'].head(100)
search = merge['body']

articles = []

for index in search.index:
    try:
        content = search.loc[index]
        
        if pd.isnull(content):
            # If 'text' is NaN, set the sentiment score as blank
            sentiment_label = ''
            sentiment_score = ''

        else:         
            sentiment = classifier(content)
            sentiment_label = sentiment[0]['label']
            sentiment_score = sentiment[0]['score']
        
        # Keep additional columns from the original DataFrame
        articles_info = {'body': content, 
                        'sentiment': sentiment_label,
                        'score': sentiment_score,
                        
                        'title': merge.loc[index, 'title'],
                        'review': merge.loc[index, 'review'],
                        'country': merge.loc[index, 'country'],
                        'experience_date_parsed': merge.loc[index, 'experience_date_parsed'],
                        
                        'rating': merge.loc[index, 'rating'],
                        'backup': merge.loc[index, 'backup'],
                        'brand': merge.loc[index, 'brand']
                        }
            
        articles.append(articles_info)

    except:
        pass

df = pd.DataFrame(articles)
```

Normalize sentiment/polarity scores
```bash
# Function to normalize sentiment scores
def normalize_sentiment(df):
    label = df['sentiment']
    score = df['score']
    
    if label == 'positive':
        return score  # Positive is mapped to score (0 to 1)
    elif label == 'negative':
        return -score  # Negative is mapped to negative score (-1 to 0)
    elif label == 'neutral':
        return (score - 0.5) * 2  # Neutral is scaled between -0.5 to 0.5

# Apply normalization function to each row
df['normalized_score'] = df.apply(normalize_sentiment, axis=1)

# Count the number of reviews by sentiment
sentiment_counts = df.groupby(['sentiment']).size()
print(sentiment_counts)

# Visualize sentiment
fig = plt.figure(figsize=(6,6), dpi=100)
ax = plt.subplot(111)
sentiment_counts.plot.pie(ax=ax, autopct='%1.1f%%', startangle=270, fontsize=12, label="")
```
