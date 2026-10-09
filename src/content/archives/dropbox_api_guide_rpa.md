---
title: Dropbox API Guide for RPA Projects
description: Using Dropbox API on RPA Projects
date: 2023-11-26 11:28
draft: false
---

>[!WARNING] Oct. 2026 note: This is an old guide I wrote back in 2023. This might be out-of-date

A non-nonsense guide

## 1. Pre-requisites

To access Dropbox API, you'll need to install the following packages:

- requests_oauthlib
- dropbox
- RPAFramework

You can install them with `pip`:

```bash
pip install requests-oauthlib dropbox rpaframework
```

## 2. Setting Up A Dropbox App

1. Go to [dropbox.com](dropbox.com) and login to the account you with to have access to in the API
2. Go to [dropbox.com/developers](dropbox.com/developers) and click in `App Console`
3. Click in `Create app`
4. Choose the `Scoped access` as API, `Full Dropbox` as the access type, and name your app. 

![~~](https://bear-images.sfo2.cdn.digitaloceanspaces.com/avancini-1706279276-0.png)

5. Click in `Create app`

You have now created successfully your Dropbox App!

## 3. Configuring the App

1. Inside the app settings, write down and save in a safe place the App key and the App secret
2. In OAuth2 Redirect URI, add a localhost URI with a unused port: (for example `https://localhost:474`)
   ![img2](https://i.imgur.com/iiAdQ3r.png)
3. Go to permissions, and set the needed scope for your app ![img3](https://i.imgur.com/5f4jHhY.png)

## 4. Getting the Access Token

_Wait, isn't there a `generate token` in the app settings?_

Yes! But this token is temporary, and is only valid for a couple of hours. Dropbox represents that the token expires by using the `sl.` pre-fix in the token.

To to get the access token, we'll need to go through the OAuth2 pipeline.

Dropbox requires you to manually login, and you can’t use the API until you do so.

To do that programmatically, we can do the following workaround using Selenium:

```python
import dropbox
from time import sleep

from requests_oauthlib import OAuth2Session
from retry import retry
from RPA.Browser.Selenium import Selenium

import dropbox
from dropbox.common import PathRoot
from dropbox.exceptions import ApiError, HttpError

from outlook import Outlook # Not a real library. Explanation after the code.

# retry if something fails
@retry(Exception, 3, 3)
def get_token() -> str:
  """
  Retrieves a token from Dropbox. As dropbox requires Oauth2, with user input of the email and password and
  accepting the connection, it opens a new browser window to do the Auth process.

  Returns:
      str: The access token.

  Raises:
      Exception: If the token cannot be retrieved.
  """
  browser = Selenium()

  try:
      #Setting up
      client_id = "{your_client_id}"
      client_secret = "{your_client_id}"
      authorization_base_url = "https://www.dropbox.com/oauth2/authorize"
      token_url = "https://api.dropboxapi.com/oauth2/token"

      #creates the OAuth2 session
      dropbox = OAuth2Session(client_id, redirect_uri="https://localhost:474")  # The same url you set in the App Settings

      # Get the dropbox Authentication page URL
      authorization_url, _ = dropbox.authorization_url(authorization_base_url)

      print("Logging into Dropbox")

      # Login to dropbox
      browser.open_available_browser(authorization_url, download=True)
      browser.wait_until_element_is_visible(locator="//input[@type='email']")
      browser.input_text(locator="//input[@type='email']", text="{account_email}")
      browser.click_button(locator="//button[@type='submit']")
      browser.wait_until_element_is_visible(locator="//input[@type='password']")
      browser.input_text(locator="//input[@type='password']", text="{account_password}")
      browser.click_button(locator="//button[@type='submit']")
      browser.wait_until_element_is_not_visible(locator="//input[@type='password']", timeout=120)

      # Sometimes it can ask for a 2FA email verification code. 
      if browser.does_page_contain_element(locator="//input[@name='code']"):
          # Example code, here the token is retrieved from outlook programmatically 
          token = Outlook().get_dropbox_token()
          browser.input_text(locator="//input[@type='text']", text=token)
          browser.click_button(locator="//button[@type='submit']")

      # it takes a few seconds to load the redirect page
      sleep(5)
      
      # The token will be in the url of the redirect page
      auth_response = browser.driver.current_url
      print("Trying to fetch token from url: {token_url}")

      # Get the token from the url
      token = dropbox.fetch_token(token_url, authorization_response=auth_response, client_secret=client_secret)

      browser.close_all_browsers()
      return token.get("access_token")
  except Exception as e:
      # if something goes wrong, close the browser
      browser.close_all_browsers()
      print(f"Error connecting to dropbox: {e}")
      raise e
```

- Sometimes dropbox can ask for a 2FA code sent via email. After getting the email message from your own module to read emails via code, you can retrieve the code from the email body this way:

```python
def get_token(message_body: str) -> str:

    # the code comes with a between a <strong> tag, and has 6 digits
    pattern = r"<strong>(\d{6})</strong>"

    # try to find the code in the message
    if match := re.search(pattern, message_body):
        security_token = match.group()
    else:
        # raise an error if couldn't find the token
        raise Exception("Could not find security token")

    # Remove the <strong> tags, and return the correct code
    return security_token.replace("<strong>", "").replace("</strong>", "")
```

## 5. Downloading Files

```python
def download_file(token:str, dropbox_file_path: str = "", local_file_path: str = "") -> None:
    """
    Download file from Dropbox

    args:
        token (str): Dropbox Access Token
        dropbox_file_path (str): The path to the file to download from Dropbox.
        local_file_path (str): The local path where to save the downloaded file.

    returns:
        None
    """
    # Creates the dropbox client
    dbx = dropbox.Dropbox(self.token)
    try:
        # Tries to Download to local file
        logger.info("Trying to download file '%s' to '%s'", dropbox_file_path, local_file_path)
        with open(local_file_path, "wb") as local_file:
            # get the binary data, and writes it to a local file
            _, res = dbx.sharing_get_shared_link_file(url=self.__credentials["team-folder"], path=dropbox_file_path)
            local_file.write(res.content)

        logger.info("File '%s' downloaded from Dropbox and saved as '%s'", dropbox_file_path, local_file_path)

    #if the file is not found, the API returns an API error
    except ApiError as err:
        if err.error.is_path() and "not_found" in err.user_message_text:
            logger.error("File not found: %s", err)
        else:
            logger.error("Error downloading Dropbox file: %s", err)
    except HttpError as err:
        logger.error("Error downloading Dropbox file: %s", err)
```

## 6. Uploading Files to Dropbox

### 6.1 Uploading to user's folder

```python
def send_file(file_path: str, dropbox_path: str):
    """
    Uploads a file to Dropbox.

    Args:
        file_path (str): The path to the file to be uploaded.
        dropbox_path (str): The path in Dropbox where the file will be uploaded.
    """
    # Creates Dropbox client
    dbx = dropbox.Dropbox(self.token)
    try:
        # reads the file and upload it
        with open(file_path, "rb") as f:
            dbx.files_upload(f.read(), dropbox_path)
    # If the file already exists, it returns an ApiError
    except ApiError:
        logger.info("File already uploaded")
    except Exception as e:
        logger.error("Could not send file: %s", e)
        return ""
    logger.info("File sent")

```

### 6.2 Uploading to shared/team folder

It is not possible to use the previous snippet to upload to shared folders. To do so, it will be needed to do some tweaks to the code:

```python
def send_file_to_shared_folder(file_path: str, dropbox_path: str, shared_folder: str):
    """
    Uploads a file to a Dropbox shared folder.

    Args:
        shared_folder (str): shared folder's name
        file_path (str): The path to the file to be uploaded.
        dropbox_path (str): The path in Dropbox where the file will be uploaded.
    """
    dbx = dropbox.Dropbox(self.token)

    # find namespace id for shared folder
    folders = dbx.sharing_list_folders()
    namespace_id = None
    for entry in folders.entries:
        if entry.name == shared_folder:
            namespace_id = entry.shared_folder_id

    # Creates a clone of the instance with the Dropbox-API-Path-Root header as the instance of PathRoot
    if not namespace_id:
        #if the process failed, uploads to local folder, as the dropbox client will not be touched here.
        logger.error("Could not find namespace id, uploading file to Dropbox user account")
    else:
        logger.info("Uploading file to Dropbox shared folder")
        # create a new client, with the shared folder as the root
        dbx = dbx.with_path_root(PathRoot.namespace_id(namespace_id))

    # Try to upload the file
    try:
        with open(file_path, "rb") as f:
            dbx.files_upload(f.read(), dropbox_path)
    except ApiError:
        logger.info("File already uploaded")
    except Exception as e:
        logger.error("Could not send file: %s", e)
        return ""
    logger.info("File sent")
```
