# webscrapping


```
import requests
from bs4 import BeautifulSoup
import time
import csv


BASE_URL = "https://www.myjobmag.co.ke"


CATEGORY_URLS = [
    f"{BASE_URL}/jobs-by-title/data-engineer",
    f"{BASE_URL}/jobs-by-title/data-analyst",


]

headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"
}

def get_job_details(job_url):
    """Visits a specific job page and extracts detailed info."""
    try:
        response = requests.get(job_url, headers=headers, timeout=10)
        if response.status_code != 200: return None

        soup = BeautifulSoup(response.text, 'html.parser')


        company = "N/A"
        company_tag = soup.find('div', class_='job-details-company')
        if not company_tag: company_tag = soup.find('a', href=lambda href: href and "jobs-at" in href)
        if company_tag: company = company_tag.get_text(strip=True)


        experience = "N/A"
        date_posted = "N/A"
        summary_items = soup.find_all('li')
        for item in summary_items:
            text = item.get_text(strip=True)
            if "Experience" in text: experience = text.replace("Experience", "").strip()
            elif "Posted" in text: date_posted = text.replace("Posted", "").strip()

        description_div = soup.find('div', class_='job-details-desc')
        if not description_div: description_div = soup.find('div', id='job-description')
        full_description = description_div.get_text(strip=True) if description_div else "N/A"

        return {
            "Company": company,
            "Date Posted": date_posted,
            "Experience": experience,
            "Full Description": full_description
        }
    except Exception as e:

        return None

def main():
    scraped_data = []
    seen_urls = set()

    for category_url in CATEGORY_URLS:
        print(f"\n--- Starting Category: {category_url} ---")
        page_number = 1

        while True:
            if page_number == 1:
                current_page_url = category_url
            else:
                current_page_url = f"{category_url}/{page_number}"

            print(f"Checking Page {page_number} ({current_page_url})...")

            try:
                response = requests.get(current_page_url, headers=headers, timeout=10)


                if response.status_code == 404:
                    print("Page not found (404). End of category.")
                    break
                elif response.status_code != 200:
                    print(f"Unexpected status {response.status_code}. Stopping.")
                    break

                soup = BeautifulSoup(response.text, 'html.parser')


                jobs = soup.find_all('li', class_='job-list-li')

                if not jobs:
                    main_content = soup.find('div', class_='job-list')
                    if main_content:
                        jobs = main_content.find_all('li')

                if not jobs:
                    print("No jobs list found on this page. Moving on.")
                    break

                jobs_found_on_this_page = 0

                for job in jobs:
                    h2 = job.find('h2')
                    if not h2: continue
                    a_tag = h2.find('a')
                    if not a_tag: continue

                    job_title = a_tag.get_text(strip=True)

                    href = a_tag['href']
                    if href.startswith("http"):
                        job_link = href
                    else:
                        job_link = BASE_URL + href if href.startswith("/") else BASE_URL + "/" + href

                    if job_link in seen_urls:
                        continue

                    title_lower = job_title.lower()
                    if "data" in title_lower or "analytics" in title_lower or "engineer" in title_lower:
                        print(f"   -> Found: {job_title}")
                        details = get_job_details(job_link)

                        if details:
                            scraped_data.append({
                                "Title": job_title,
                                "Company": details['Company'],
                                "Date Posted": details['Date Posted'],
                                "Experience": details['Experience'],
                                "Skills/Desc": details['Full Description'][:9000000000000000000000000000000000000000000000000000000009999999999999000000000000000] ,
                                "Link": job_link
                            })
                            seen_urls.add(job_link)
                            jobs_found_on_this_page += 1

                        time.sleep(0.5)

                if jobs_found_on_this_page == 0 and page_number > 1:
                    print("No new unique jobs found. Stopping category.")
                    break

                page_number += 1
                time.sleep(1)

            except Exception as e:
                print(f"Error: {e}")
                break

    print(f"\nCompleted! Total unique jobs scraped: {len(scraped_data)}")
    if scraped_data:
        keys = scraped_data[0].keys()
        with open('myjobmag_fixe.csv', 'w', newline='', encoding='utf-8') as f:
            writer = csv.DictWriter(f, fieldnames=keys)
            writer.writeheader()
            writer.writerows(scraped_data)
        print("Saved to myjobmag_fixe.csv")
    else:
        print("No jobs found. Check your Internet connection or the URLs.")

if __name__ == "__main__":
    main()
```
