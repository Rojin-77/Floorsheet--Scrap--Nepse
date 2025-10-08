# Floorsheet--Scrap--Nepse
The floorsheet data of around 10 thousands rows contained in 200 pages of 500 rows in each pages can be scrapped from here by creating a loop on merolagani pages.
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


def get_page_table(driver, table_class):
    """Extract table data from the current page."""
    soup = BeautifulSoup(driver.page_source, 'html.parser')
    table = soup.find("table", {"class": table_class})
    
    if table is None:
        print("No table found on the page.")
        return pd.DataFrame()
    
    tab_data = [[cell.text.strip() for cell in row.find_all(["th", "td"])]
                for row in table.find_all("tr")]
    
    return pd.DataFrame(tab_data)


def scrape_data(driver, date):
    """Scrape data from all pages."""
    start_time = datetime.now()
    search(driver, date)
    
    df = pd.DataFrame()
    
    while True:
        page_df = get_page_table(driver, "table table-bordered table-striped table-hover sortable")
        df = pd.concat([df, page_df], ignore_index=True)
        
        try:
            next_btn = driver.find_element(By.LINK_TEXT, 'Next')
            driver.execute_script("arguments[0].click();", next_btn)
            time.sleep(1.5)  # Allow time for page to load
            dismiss_alert(driver)
        except NoSuchElementException:
            break

    print(f"Time taken to scrape: {datetime.now() - start_time}")
    return df


def clean_df(df):
    """Clean the DataFrame by removing duplicates and formatting columns."""
    new_df = df.drop_duplicates(keep='first')
    new_df.columns = new_df.iloc[0]
    new_df = new_df[1:]

    # Remove unwanted column
    if "#" in new_df.columns:
        new_df.drop("#", axis=1, inplace=True)

    # Clean numeric columns
    for col in ["Rate", "Amount"]:
        if col in new_df.columns:
            new_df[col] = new_df[col].str.replace(",", "", regex=False).astype(float)

    return new_df


def main():
    date = "09/07/2025" # Date format MM/DD/YYYY

    # Chrome headless mode
    options = Options()
    options.headless = True
    driver = webdriver.Chrome(options=options)

    try:
        df = scrape_data(driver, date)
        final_df = clean_df(df)

        output_directory = r"C:\Users\croje\OneDrive\0000.Daily Trade Data"
        os.makedirs(output_directory, exist_ok=True)

        file_name = date.replace("/", "_") + ".csv"
        output_path = os.path.join(output_directory, file_name)

        if not final_df.empty:
            final_df.to_csv(output_path, index=False)
            print(f"Data saved to {output_path}")
        else:
            print("No data available to save.")
    finally:
        driver.quit()


if __name__ == "__main__":
    main()
