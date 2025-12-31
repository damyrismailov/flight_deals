# Flight Deals Finder

Python project that checks flight prices for a list of destinations and notifies you when a cheaper deal appears.  
It pulls destinations and target prices from a Google Sheet (via Sheety), looks up real flight offers via the Amadeus API, and sends alerts with Twilio when it finds a flight under your target price.

## Main features

- Reads a table of destinations and target prices from a Google Sheet using the Sheety API.
- Fills in missing IATA airport codes for each city by querying the Amadeus locations API.
- Searches the Amadeus flight offers API for real flights:
  - origin airport is defined by `ORIGIN_CITY_IATA` (in my case `"AYT"` – Antalya),
  - destination airports come from the sheet,
  - date window is from tomorrow up to six months from today,
  - filters for non-stop flights and a maximum number of results.
- Extracts the cheapest flight for each destination (price, origin airport, destination airport, departure date, return date).
- Compares the cheapest price with the user’s target price stored in the Google Sheet (`lowestPrice` column).
- When a cheaper flight is found, sends an alert using Twilio:
  - SMS via a Messaging Service SID,
  - optional WhatsApp message via Twilio sandbox / verified number.
- Uses environment variables and a `.env` file to keep all API keys, usernames and passwords out of the source code.

## What I learned

- Structuring a project with multiple classes and separating responsibilities:
  - `DataManager` for reading/writing Google Sheet data,
  - `FlightSearch` for talking to the Amadeus API,
  - `FlightData` as a simple object to represent a single flight deal,
  - `NotificationManager` for sending SMS / WhatsApp alerts with Twilio.
- Using the `requests` library to authenticate and call several REST APIs (Sheety, Amadeus, Twilio).
- Authenticating against the Amadeus API by first requesting an OAuth token, then using that token in the `Authorization: Bearer <token>` header.
- Parsing nested JSON responses to extract only the fields I need (price, airports, dates).
- Reading secrets from environment variables with `os.environ` and `python-dotenv` instead of hard-coding keys in the code.
- Coordinating several components from a single `main.py` script and keeping the control flow readable.

## Project structure

- `main.py` – orchestrates the workflow:
  - loads destination data from the sheet,
  - fills in missing IATA codes and updates the sheet,
  - searches for flights for each destination,
  - finds the cheapest flight,
  - triggers notifications when a deal is below the target price.
- `data_manager.py` – handles all communication with the Google Sheet via Sheety:
  - authenticates with Basic Auth using username and password from environment variables,
  - downloads the current destination list,
  - updates rows with new IATA codes.
- `flight_search.py` – talks to the Amadeus API:
  - requests a new access token using `client_credentials`,
  - looks up airport / city IATA codes,
  - searches for flight offers with filters (date window, non-stop, currency).
- `flight_data.py` – `FlightData` class that stores:
  - `price`,
  - `origin_airport`,
  - `destination_airport`,
  - `out_date`,
  - `return_date`,
  - plus a helper function to pick the cheapest flight from the API response.
- `notification_manager.py` – wraps Twilio:
  - sends SMS alerts using a Messaging Service SID and a destination phone number,
  - optionally sends WhatsApp alerts using Twilio sandbox numbers.
- `.env` – local file (not committed) that stores all credentials and configuration values.
- `requirements.txt` (optional) – can list dependencies like `requests`, `python-dotenv`, `twilio`.

## Setup

1. Install Python 3.
2. Install dependencies with pip:

       pip install requests python-dotenv twilio

3. **Google Sheet + Sheety**
   - Create a Google Sheet with columns such as:
     - `city`
     - `iataCode`
     - `lowestPrice`
   - Connect the sheet to Sheety and get the API endpoint URL for the `prices` tab.
   - Enable Basic Auth in Sheety and set a username and password.

4. **Amadeus API**
   - Create an Amadeus for Developers account.
   - Create an application and get:
     - `API Key` (client ID),
     - `API Secret` (client secret).
   - Use the test environment URLs (as in `flight_search.py`).

5. **Twilio**
   - Create a Twilio account.
   - Get:
     - `Account SID`,
     - `Auth Token`,
     - a **Messaging Service SID** for SMS,
     - a verified phone number to receive SMS.
   - (Optional) Set up the WhatsApp sandbox if you want WhatsApp alerts.

6. **Environment variables (`.env`)**

   Create a `.env` file in the project root with values like:

       SHEETY_USERNAME=your_sheety_username
       SHEETY_PASSWORD=your_sheety_password
       SHEETY_URL=https://api.sheety.co/.../flightDeals/prices

       AMADEUS_API=your_amadeus_api_key
       AMADEUS_SECRET=your_amadeus_api_secret

       TWILIO_SID=your_twilio_account_sid
       TWILIO_AUTH_TOKEN=your_twilio_auth_token
       MESSAGING_SERVICE_SID=your_twilio_messaging_service_sid
       NUMBER=+90XXXXXXXXXX

       # Optional for WhatsApp
       TWILIO_WHATSAPP_NUMBER=+1415XXXXXXX
       TWILIO_VERIFIED_NUMBER=+90XXXXXXXXXX

   Make sure `.env` is in `.gitignore` so you don’t commit secrets.

7. **Origin city and destinations**

   - In `main.py`, set your home airport IATA code:

       ORIGIN_CITY_IATA = "AYT"

   - Fill the Google Sheet with the cities you care about and your desired `lowestPrice` for each destination.

## How to run

1. Activate your virtual environment (if you use one).
2. From the project folder, run:

       python main.py

3. The script will:
   - download the destination data from the sheet,
   - fill in missing IATA airport codes and update the sheet,
   - search for flights from your origin to each destination between tomorrow and six months from now,
   - find the cheapest non-stop flights,
   - send you an SMS (and optional WhatsApp) alert when the price is below your target.

You can schedule `main.py` to run regularly (for example with a cron job, Windows Task Scheduler or a cloud service) to keep checking for new flight deals automatically.
