# Guidewiring

Key Considerations for Guidewire Billing API Integration:

Authentication: Guidewire APIs typically use authentication mechanisms (e.g., Basic Auth, OAuth). You'll need to handle authentication properly in your connector. This will involve storing credentials securely (using Fivetran's configuration) and including the necessary headers in your requests.
API Endpoints: Identify the specific API endpoints for retrieving billing plan data. The documentation is your guide here. You'll likely have a base URL for the Guidewire instance and then specific paths for billing plans.
Data Structure: Understand the JSON structure of the billing plan data returned by the Guidewire API. This will inform how you define your schema and how you extract data in the update function.
Pagination: Guidewire APIs might return data in paginated form. Your connector needs to handle pagination correctly to retrieve all billing plans, not just the first page. Look for details in the API documentation on how to handle next page or continuation tokens.
Rate Limiting: Be mindful of rate limits imposed by the Guidewire API. Implement error handling and potentially add delays to your requests if necessary.
Error Handling: Implement robust error handling to manage API errors, network issues, and unexpected data formats. Use the log functionality from the fivetran_connector_sdk to log errors and warnings.
State Management: Use Fivetran's state management to track progress and efficiently retrieve updates. For billing plans, you might track the last updated timestamp or an ID to fetch new or modified records.
Modified connector.py for Guidewire Billing Plans:

Python

from datetime import datetime
import requests as rq
from fivetran_connector_sdk import Connector
from fivetran_connector_sdk import Logging as log
from fivetran_connector_sdk import Operations as op

# Configuration Keys (Important: Use Fivetran UI for secure storage)
GUIDEWIRE_BASE_URL = "guidewire_base_url"  # e.g., "http://your-guidewire-instance/billing/rest/v1"
GUIDEWIRE_USERNAME = "guidewire_username"
GUIDEWIRE_PASSWORD = "guidewire_password"

def schema(configuration: dict):
    """
    Defines the schema for billing plan data. Adjust this based on the 
    actual structure of the Guidewire Billing API response.
    """
    return [
        {
            "table": "billing_plans",
            "primary_key": ["id"],  # Assuming 'id' is the primary key
            "columns": {
                "id": "STRING",
                "plan_name": "STRING",
                "description": "STRING",
                "start_date": "UTC_DATETIME",
                "end_date": "UTC_DATETIME",
                "status": "STRING",
                # Add more columns as needed based on the API response
            },
        }
    ]

def str2dt(incoming: str) -> datetime:
    """Helper function to convert string to datetime (adjust format as needed)."""
    try:
        return datetime.strptime(incoming, "%Y-%m-%dT%H:%M:%S.%fZ")  # Adjust format
    except ValueError:
        return None  # Or handle the exception as appropriate

def update(configuration: dict, state: dict):
    """
    Fetches and processes billing plan data from the Guidewire Billing API.
    """
    log.info("Fetching billing plan data from Guidewire")

    base_url = configuration.get(GUIDEWIRE_BASE_URL)
    username = configuration.get(GUIDEWIRE_USERNAME)
    password = configuration.get(GUIDEWIRE_PASSWORD)

    if not base_url or not username or not password:
        log.error("Missing required configuration: base URL, username, or password")
        return  # Or raise an exception

    # --- Authentication (Basic Auth Example - Adapt as needed) ---
    auth = (username, password)

    # --- API Endpoint (Adapt based on Guidewire Docs) ---
    billing_plans_url = f"{base_url}/billingplans"  # Example endpoint

    # --- State Management (Example: Last updated timestamp) ---
    last_updated = state.get("last_updated")  # Assuming you store the last updated time

    try:
        while billing_plans_url:  # Handle pagination
            log.info(f"Fetching data from: {billing_plans_url}")
            
            # --- Make the API Request ---
            response = rq.get(billing_plans_url, auth=auth)  # Include auth

            response.raise_for_status()  # Raise an exception for bad status codes

            data = response.json()
            
            # --- Process the Data (Adapt based on API response structure) ---
            plans = data.get("data", [])  # Adjust based on the actual response structure
            #Guidewire responses may vary - this is an example
            
            if not plans:
               log.info("No billing plans found in this response")
               break #Exit loop if no plans are found

            for plan in plans:
                # --- Example Data Extraction (Adapt based on API response) ---
                plan_id = plan.get("id")
                plan_name = plan.get("attributes", {}).get("planName")
                description = plan.get("attributes", {}).get("description")
                start_date_str = plan.get("attributes", {}).get("startDate")
                end_date_str = plan.get("attributes", {}).get("endDate")
                status = plan.get("attributes", {}).get("status")

                start_date = str2dt(start_date_str) if start_date_str else None
                end_date = str2dt(end_date_str) if end_date_str else None

                # --- State Check (if using last_updated) ---
                #   if last_updated and plan.get("updated_at") < last_updated:
                #       continue

                yield op.upsert(
                    table="billing_plans",
                    data={
                        "id": plan_id,
                        "plan_name": plan_name,
                        "description": description,
                        "start_date": start_date,
                        "end_date": end_date,
                        "status": status,
                        # Add more fields as needed
                    },
                )
                
                # --- Update State (Example: Update last_updated) ---
                #   last_updated = plan.get("updated_at") #example
                
            # --- Handle Pagination (Example: Check for a 'next' link) ---
            #   billing_plans_url = data.get("links", {}).get("next")
            #   if not billing_plans_url:
            #       break
            billing_plans_url = None #simple no pagination example
                
        # --- Checkpoint the State ---
        #   yield op.checkpoint(state={"last_updated": last_updated})
        yield op.checkpoint(state={}) #no state example
    
    except rq.exceptions.RequestException as e:
        log.error(f"API request error: {e}")
    except ValueError as e:
        log.error(f"Error parsing JSON: {e}")
    except Exception as e:
        log.error(f"An unexpected error occurred: {e}")

connector = Connector(update=update, schema=schema)

if __name__ == "__main__":
    connector.debug()
Explanation of Changes and Important Notes:

Configuration:
I've added placeholder configuration keys (GUIDEWIRE_BASE_URL, GUIDEWIRE_USERNAME, GUIDEWIRE_PASSWORD). Crucially, you should configure these securely within the Fivetran UI itself, not hardcoding them in the code.
Authentication:
The code includes a basic example of HTTP Basic Authentication. Guidewire might use other methods (OAuth, etc.). You MUST adapt the authentication part to match your Guidewire API's requirements. Refer to your Guidewire documentation.
API Endpoint:
The billing_plans_url is a placeholder. You need to replace it with the correct endpoint from the Guidewire Billing API documentation.
Data Extraction:
The data extraction within the update function (plan_id, plan_name, etc.) is example code. You'll need to carefully inspect the JSON response from the Guidewire API and adjust the code to correctly extract the fields you need. Use response.json() and print it out or inspect it in a debugger to understand the structure.
Pagination:
I've included a placeholder for pagination handling. Guidewire's API might use different pagination strategies (e.g., page numbers, page sizes, continuation tokens). You MUST implement the pagination logic according to Guidewire's documentation. If there is no pagination, you can remove the while loop and related logic.
State Management:
State management is crucial for efficient updates. I've included an example of how you might use a last_updated timestamp. Determine the best strategy for your use case and the Guidewire API (e.g., using IDs, timestamps, or other criteria). If you don't need state, you can checkpoint with an empty state.
Error Handling:
The code includes basic error handling using try-except blocks and the log module. Enhance this error handling as needed to make your connector robust.
Date Handling:
The str2dt function now includes a try-except block to handle potential ValueError exceptions that can occur if the date string format is unexpected. Adjust the date format string ("%Y-%m-%dT%H:%M:%S.%fZ") to match the format used by the Guidewire API.
Logging:
The log module is used for logging information, warnings, and errors. Use this effectively to help with debugging and monitoring your connector.
To use this code:

Install the Fivetran Connector SDK:
Bash

pip install fivetran_connector_sdk
Replace Placeholder Values:
Fill in the correct configuration keys in the Fivetran UI.
Replace the placeholder API endpoint URL.
Adapt the authentication code.
Adjust the data extraction logic.
Implement pagination (if needed).
Implement state management (if needed).
Test the Connector:
Use the connector.debug() method to test your connector locally.
Deploy to Fivetran:
Follow the Fivetran documentation to deploy your connector.
Important Reminder:

Consult the Guidewire Billing API Documentation: The most critical step is to thoroughly review the official Guidewire Billing API documentation. It will provide the accurate details you need for authentication, endpoints, data structures, pagination, and any other specific requirements.
Security Best Practices: Never hardcode sensitive information (like passwords) directly into your connector code. Always use Fivetran's secure configuration mechanisms.
This revised code provides a much stronger foundation for building your Guidewire Billing API connector. Remember to tailor it precisely to the specifics of the Guidewire API you are using.


Sources and related content
