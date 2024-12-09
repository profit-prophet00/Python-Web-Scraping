# Scrape & Analyze: Python Web Scraping for Activewear Insights

## Project Highlights  
  
Discover consumer sentiment trends with Python Web Scraping: Ever wondered how to glean insights from the web automatically? Leverage the power of web scraping with this Python toolkit and dive into real-time consumer sentiment analysis, in the activewear industry. This project provides a comprehensive implementation of the Selenium library to collect data from the web and later estimate consumer sentiment trends.

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
pip install pandas numpy nltk re tqdm time selenium   
```

Library imports
```bash
import pandas as pd
import numpy as np
import nltk
import re
from tqdm import tqdm
import time
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
Consult the following website for recent documentation on chromedriver versions: https://developer.chrome.com/docs/chromedriver/downloads
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

Potential targets for analysis
<br>
Established activewear companies
1. https://www.trustpilot.com/review/www.adidas.com
2. https://www.trustpilot.com/review/puma.com
3. https://www.trustpilot.com/review/www.nike.com

New comers: 
1. https://www.trustpilot.com/review/arcteryx.com
2. https://www.trustpilot.com/review/www.salomon.com


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

# Decode each entry to ensure consistent encoding, when writing to .xlsx
merge['backup'] = merge['backup'].apply(lambda x: x.encode('utf-8').decode('unicode_escape') if isinstance(x, str) else x)
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
merge["body_DE"] = merge["backup"].apply(safe_translate)
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

# Parse date column
# Use str.extract with a regex pattern to extract the date  
merge['experience_date_parsed'] = merge['experience_date'].str.extract(r'Date of experience: (.+)')  

#Drop columns
merge = merge.drop(columns=["part1", "part2", "experience_date"])
```
