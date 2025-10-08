# Floorsheet--Scrap--Nepse
The floorsheet data of around 10 thousands rows contained in 200 pages of 500 rows in each pages can be scrapped from here by creating a loop on merolagani pages.

My code is attached in the mentioned file. #used beautiful soup and selenium for extraction of the data


from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from selenium.common.exceptions import (
    NoSuchElementException,
    UnexpectedAlertPresentException,
    NoAlertPresentException
)
from bs4 import BeautifulSoup
from datetime import datetime
import pandas as pd
import os
import sys
import time


def dismiss_alert(driver):
    """Dismiss any alerts that may appear."""
    try:
        alert = driver.switch_to.alert
        alert.dismiss()
        print("Alert dismissed.")
    except NoAlertPresentException:
        print("No alert to dismiss.")
    except UnexpectedAlertPresentException:
        alert = driver.switch_to.alert
        alert.dismiss()
    except Exception as e:
        print(f"Error while trying to dismiss alert: {e}")


def remove_ads(driver):
    """Remove iframe ads that may block clicks."""
    try:
        driver.execute_script("""
            var ads = document.querySelectorAll("iframe[id^='aswift_']");
            ads.forEach(ad => ad.style.display = 'none');
        """)
        print("Ads hidden from page.")
    except Exception as e:
        print(f"Failed to hide ads: {e}")


def search(driver, date):
    """Search for the given date on the website."""
    driver.get("https://merolagani.com/Floorsheet.aspx")
    time.sleep(2)  # Give time for page to load

    dismiss_alert(driver)
    remove_ads(driver)

    try:
        date_input = driver.find_element(By.XPATH, '/html/body/form/div[4]/div[4]/div/div/div[1]/div[4]/input')
        search_btn = driver.find_element(By.XPATH, '/html/body/form/div[4]/div[4]/div/div/div[2]/a[1]')
    except NoSuchElementException as e:
        print(f"Error finding elements: {e}")
        driver.quit()
        sys.exit(1)

    date_input.clear()
    date_input.send_keys(date)
    
    # Scroll and click using JavaScript
    driver.execute_script("arguments[0].scrollIntoView(true);", search_btn)
    time.sleep(0.5)
    driver.execute_script("arguments[0].click();", search_btn)

    time.sleep(2)  # Wait for results to load
    dismiss_alert(driver)

    if driver.find_elements(By.XPATH, "//*[contains(text(), 'Could not find floorsheet matching the search criteria')]"):
        print("No data found for the given search.")
        print("Aborting script ......")
        sys.exit()
