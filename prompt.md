# The Prompt

You can feed this prompt to [ChatGPT](https://chatgpt.com/), [Claude](https://claude.ai/), or any other AI tool of your choice.

    I have the following guide about scraping a website. Please adapt it to be for how to scrape a different website. Walk me through the steps one by one to guide me in creating a new walkthrough.

    Proceed in several steps, ONE AT A TIME. Do not proceed to the next step until you get a satisfactory answer to the prior one.

    1. Ask me for the first URL I need to visit
    2. Ask me if I need to fill out a form, click a cookie banner, or perform other page interactions before we start scraping.
    2. Ask for the HTML that I should scrape for each row of data
    3. Confirm pagination details
    4. When you are ready to provide me the code, ask me if I want "flat" code that respects the "important notes" section. If yes, make sure to make code that works well in Jupyter notebook: only use async playwright, do not use asyncio, flatten the code to not use functions (make sure the data and df are available in other cells), and do not use a main function.

    If I fill out a form for searching, you may need to get the HTML for the entire form or parts of the form. If necessary use a similar walkthrough as below to understand the HTML. If we need to provide a list of inputs - zip codes, names, license IDs, etc - determine whether they are coming from a dataframe, and if so what the column name is. Be sure you know how to submit the form.

    To confirm row data details, walk me through how to copy the outer HTML of one "row" of data. I'm new to scraping and might need help. Decide on columns to be saved, then confirm them with me. If it isn't clear, explain what you need from me.

    Help me determine the pagination situation. Are there multiple pages of content? Is there a 'next page' button that can be pressed again and again? Do you need to click an incrementing number of pages? Have me copy any HTML for the pagination so you know how to parse/interact with it to be sure we scrape all of the data. Ask if infinite scroll is necessary.

    ## Important Notes: 

    - You MUST use async playwright
    - Do NOT use asyncio and do NOT wrap everything in an async function.
    - If displaying a dataframe, do NOT use print or ace_tools, just use df.head()
    - The sample tutorial reads a table with pd.read_html. If there is no table on the page, this is not possible and you must use the 'normal' selector process.
    - To protect against missing data, create the DataFrame from a list of dictionaries, not a dictionary of lists. Assume that not all rows have all columns, and guard against timeouts caused by missing data (e.g. some elements might not have a description).
    - If it is a multi-page scraper, provide code to test whether the scraper works on one page of data before jumping into the full process. Ask whether I am satisfied with the result before proceeding.
    - Be resilient against errors when scraping, printing details on failures to help debugging
    - You do NOT need to close the browser. We can figure that out ourselves.
    - Provide complete code
    - The site might be slow or I might have bad internet, set all timeouts to be 10 seconds at a minimum
    - To do the actual extraction, use Playwright's wait_for to assure the elements are on the page then await page.contents() with BeautifulSoup
    - Do NOT use a main function, just make code as "flat" as possible

    GUIDE:

    ## Installation

    We need to install a few tools first! Run the cell to install the Python packages and browsers that we'll need for our scraping adventure.

    ```
    %%python -m pip install lxml html5lib beautifulsoup4 pandas
    %%python -m pip install --quiet playwright
    !playwright install
    ```

    ## Opening up the browser and visiting our destination

    Note that we will NOT use asyncio or sync_playwright in the example below. We also do NOT wrap our code in a big async function.

    ```
    from playwright.async_api import async_playwright

    # "Hey, open up a browser"
    playwright = await async_playwright().start()
    browser = await playwright.chromium.launch(headless=False)

    # Create a new browser window
    page = await browser.new_page()
    await page.goto("https://newjersey.mylicense.com/verification/Search.aspx")
    ```

    ## Selecting an option from a dropdown

    You always start with await page.locator("select").select_option("whatever option you want"). You'll probably get an error because there are multiple dropdowns on the page, but Playwright doesn't know which one you want to use! Just read the error and figure out the right one.

    ```
    # await page.locator("select").select_option("Acupuncture")
    await page.locator("#t_web_lookup__profession_name").select_option("Perfusionist")
    ```

    Click the submit button.

    ```
    # await page.get_by_text("Search").click()
    await page.get_by_role("button", name="Search").click()
    ```

    ## Grab the tables from the page

    Pandas is the Python equivalent to Excel, and it's great at dealing with tabular data! Often the data on a web page that looks like a spreadsheet can be read with pd.read_html.

    You use await page.content() to save the contents of the page, then feed it to read_html to find the tables. len(tables) checks the number of tables you have, then you manually poke around to see which one is the one you're interested in. tables[0] is the first one, tables[1] is the second one, and so on...

    ```
    import pandas as pd
    from io import StringIO

    html = await page.content()
    tables = pd.read_html(StringIO(html))
    len(tables)
    tables[0]
    ```

    ## Clicking "next page" one

    Just like using a dropdown, select box or button, we'll use page.get_by_text to try to select the button.

    We add timeout=10000 to wait 5 seconds before confirming it isn't there.

    ```
    # page.get_by_text("Next Page").click()
    await page.locator("a:has-text('Next Page')").click(timeout=10000)
    ```

    ## Clicking "next page" until it disappears

    Eventually the "next page" link disappears, and Python starts screaming. We use try/except down below to say "don't worry little baby, it's okay, we'll just finish up if the button is gone."

    ```
    # all of our data will end up going here
    all_data = pd.DataFrame()

    while True:
        await page.wait_for_selector("table")

        # Get all of the tables on the page
        html = await page.content()
        tables = pd.read_html(StringIO(html))

        # Get the table (and edit if necessary)
        df = tables[0]

        # Add the tables on this page to the big list of stuff
        all_data = pd.concat([all_data, df], ignore_index = True)
        try:
            await page.locator("a:has-text('Next Page')").click(timeout=5000)
        except:
            break
    all_data
    ```

    ## Saving the results

    Now we'll save it to a CSV file! Easy peasy.

    ```
    all_data.to_csv("output.csv", index=False)
    ```
