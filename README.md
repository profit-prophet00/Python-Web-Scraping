# Python Web Scraping Application

## Project Highlights  
  
Have you ever wondered how developers and businesses alike automatically scrape data from the web? Dive into the world of web scraping with this ready-to-deploy web scraping toolkit, where we use one of the most common Python libraries for web-scraping - Selenium. This project provides a comprehensive implementation of this library to estimate consumer sentiment trends in the sports apparal using Python.

<br>

...

<br>

In this Python project, you will explore the following key areas:  
  
1. **Understanding x**: y.  
2. **Data Retrieval**: y.  
3. **Model Implementation**: y.  
4. **Visualization**: y.  
5. **Sensitivity Analysis**: y. 

<br>
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
```bash
# Established activewear companies
  # https://www.trustpilot.com/review/www.adidas.com
  # https://www.trustpilot.com/review/puma.com
  # https://www.trustpilot.com/review/www.nike.com

# New comers: 
  # https://www.trustpilot.com/review/arcteryx.com
  # https://www.trustpilot.com/review/www.salomon.com
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

...
