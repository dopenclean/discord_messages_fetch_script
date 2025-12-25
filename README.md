# Discord Channel Scraper

## Instructions

1. Install required Python package: `pip install requests`
2. Add your Discord authorization token to the `USER_AUTH_TOKEN` variable
3. Update the `CHANNEL_ID` if needed
4. Run the script: `python script_name.py`

The script will fetch all messages from the specified Discord channel and save them to `all_channel_messages.json`. 
It supports incremental updates - if you run it again, it will only fetch new messages since the last run.

## Script

```python
import requests
import json
import time

# --- 1. CONFIGURATION ---
USER_AUTH_TOKEN = "" # YOUR AUTH token on discord.com/channels/@me

CHANNEL_ID = "" # Add channel ID you desire
OUTPUT_FILENAME = "all_channel_messages.json"

# --- 2. API SETUP ---
API_URL = f"https://discord.com/api/v9/channels/{CHANNEL_ID}/messages"

HEADERS = {
    "Authorization": USER_AUTH_TOKEN,
    "Content-Type": "application/json"
}

# --- 3. LOAD EXISTING MESSAGES ---
def load_existing():
    try:
        with open(OUTPUT_FILENAME, "r", encoding="utf-8") as f:
            existing = json.load(f)
        if not isinstance(existing, list) or not existing:
            raise ValueError("Invalid or empty existing messages.")
        last_known_id = existing[0]["id"]
        print(f"Loaded {len(existing)} existing messages. Latest ID: {last_known_id}")
        return existing, last_known_id
    except (FileNotFoundError, json.JSONDecodeError, ValueError, KeyError):
        print("No valid existing file found. Starting full fetch.")
        return [], None

# --- 4. MESSAGE FETCHING FUNCTION ---
def fetch_messages(last_known_id):
    new_messages = []
    params = {"limit": 100}
    done = False

    print(f"Starting message fetch from channel {CHANNEL_ID}...")

    while not done:
        try:
            response = requests.get(API_URL, headers=HEADERS, params=params)

            if response.status_code == 429:
                retry_after = response.json().get('retry_after', 5)
                print(f"Rate limit hit! Waiting for {retry_after} seconds.")
                time.sleep(retry_after)
                continue

            response.raise_for_status()
            batch = response.json()

            if not batch:
                break

            filtered = []
            for msg in batch:
                if last_known_id is None or int(msg["id"]) > int(last_known_id):
                    filtered.append(msg)
                else:
                    done = True
                    break

            new_messages.extend(filtered)

            if done:
                break

            if len(filtered) == len(batch):
                params["before"] = batch[-1]["id"]
                print(f"Fetched {len(batch)} messages, added {len(filtered)}. Total new: {len(new_messages)}")
            else:
                print(f"Fetched {len(batch)} messages, added {len(filtered)} (partial). Total new: {len(new_messages)}")
                break

            time.sleep(1)

        except requests.exceptions.RequestException as e:
            print(f"An error occurred during API request: {e}")
            break
        except ValueError:
            print("Invalid message ID encountered. Skipping.")
            break

    print(f"\nFinished fetching. New messages collected: {len(new_messages)}")
    return new_messages

# --- 5. SAVE ALL MESSAGES ---
def save_all_messages(messages):
    with open(OUTPUT_FILENAME, "w", encoding="utf-8") as f:
        json.dump(messages, f, ensure_ascii=False, indent=4)

    print(f"\nSuccess! Saved {len(messages)} messages to {OUTPUT_FILENAME}.")

# --- 6. EXECUTION ---
if __name__ == "__main__":
    if USER_AUTH_TOKEN == "YOUR_DISCORD_AUTHORIZATION_HEADER":
        print("ERROR: Please update USER_AUTH_TOKEN in the script configuration.")
    elif CHANNEL_ID == "YOUR_CHANNEL_ID":
        print("ERROR: Please update CHANNEL_ID in the script configuration.")
    else:
        existing, last_known_id = load_existing()
        new_messages = fetch_messages(last_known_id)
        if new_messages or not existing:
            total_messages = new_messages + existing
            save_all_messages(total_messages)
        else:
            print("No new messages found. Existing file unchanged.")
```
