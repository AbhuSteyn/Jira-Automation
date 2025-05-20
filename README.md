
# GitHub-to-JIRA Ticket Automator

This project automates the creation of JIRA tickets when a GitHub issue comment starts with the `/ticket` command. The workflow is as follows:

1. A developer comments `/ticket` followed by ticket details in a GitHub issue.
2. GitHub sends a webhook payload with the comment and associated issue details to the Flask server.
3. The Flask server parses the webhook payload:
   - It extracts the comment text and checks for the `/ticket` command.
   - It then removes the command text and uses the remainder as the ticket description.
4. Using the JIRA REST API, the Flask application creates a new JIRA ticket:
   - The issue title is used to generate a default ticket summary.
   - The comment text (without the `/ticket` command) is used as the ticket description.
5. The JIRA API returns a response—if successful, the Flask server responds back to GitHub with a success message; otherwise, an error is returned.
6. This feedback can be logged and even inserted as a comment on the GitHub issue if desired.

## Overview

By integrating GitHub issue comments with JIRA through a custom webhook, teams can streamline ticket management. Instead of manually recreating an issue in JIRA, the `/ticket` command automates the process, reduces duplication, and ensures that both systems stay in sync.

## Details on Data Flow

### How Comment Details Are Passed from GitHub to JIRA

- **GitHub Webhook Payload:**
  - When an issue comment event occurs (either on creation or editing of a comment), GitHub sends a JSON payload to the configured webhook URL.
  - This payload contains the entire issue context (the issue title, number, etc.) and details of the comment (such as `comment.body`, which contains the text of the comment).
  
- **Extracting the Comment in the Flask Server:**
  - The Flask application listens on the `/github-webhook` endpoint.
  - It reads the JSON payload, and specifically looks at `data["comment"]["body"]` for the comment text.
  - If the comment text starts with `/ticket`, the command is stripped and the following text is used as the ticket description.
  - Relevant context is also extracted—for example, the issue title (`data["issue"]["title"]`) to build a ticket summary.

- **Sending Data to JIRA:**
  - The Flask server then constructs a JSON payload expected by the JIRA REST API.
  - This payload includes fields such as:
    - **Project Key:** indicating the project in JIRA.
    - **Summary:** often constructed as "Ticket from GitHub: [Issue Title]".
    - **Description:** the comment content provided after `/ticket`.
  - The payload is sent via an HTTP POST request to JIRA’s create issue endpoint.

### How Feedback Is Received Back from JIRA and Webhook

- **JIRA Response:**
  - JIRA responds with a status code and a JSON message. A `200` or `201` status code typically confirms that the ticket was created successfully.
  - If there is an error (due to bad credentials or misconfigured fields, etc.), JIRA sends back an error payload with details.

- **Flask Webhook Feedback:**
  - The Flask application checks the status code returned from JIRA.
  - On success, it logs the success and returns a JSON response, e.g. `{"message": "JIRA ticket created successfully"}`, which is sent back to the POST request caller (GitHub).
  - On error, it sends a JSON response with the error details so that the originator of the webhook (or logs) can be used to troubleshoot the problem.
  - Optionally, the feedback could be posted as a comment in the GitHub issue to inform the user.

## Prerequisites

- **Python 3.9+** installed.
- **Flask** installed for the webhook server.
- Configured GitHub repository with issues enabled and webhook notifications set.
- A JIRA account with API access (using API tokens is recommended).
- Environment variables for connecting to JIRA (configured in a `.env` file).

## Setup and Configuration

### 1. Clone the Repository

```bash
git clone https://github.com/yourusername/github-to-jira-ticket-automator.git
cd github-to-jira-ticket-automator
```

### 2. Set Up Your Python Environment

Create a virtual environment and install dependencies:

```bash
python -m venv venv
source venv/bin/activate      # On Windows: venv\Scripts\activate
pip install -r requirements.txt
```

*Example `requirements.txt`:*
```
Flask==2.x.x
python-dotenv==1.x.x
requests==2.x.x
```

### 3. Create a `.env` File

In the project root, create a `.env` file with the following environment variables:

```dotenv
# Flask settings
FLASK_PORT=5000
FLASK_DEBUG=True

# JIRA API settings
JIRA_BASE_URL=https://your-domain.atlassian.net
JIRA_USERNAME=your-email@example.com
JIRA_API_TOKEN=your_jira_api_token_here
JIRA_PROJECT_KEY=PROJ
```

### 4. Configure the GitHub Webhook

- Go to your GitHub repository **Settings** > **Webhooks**.
- Click **Add webhook**.
- **Payload URL**: Set it to your Flask server's public URL (e.g., `http://your-server-address:5000/github-webhook`).
- **Content type**: `application/json`
- **Events**: Select **Issue Comment** (or all events if desired).
- Save the webhook.

## Flask Application Code

Create a file named `app.py` in the project root:

```python
import os
import json
import requests
from flask import Flask, request, jsonify
from dotenv import load_dotenv

# Load environment variables from .env file
load_dotenv()

app = Flask(__name__)

# JIRA configuration
JIRA_BASE_URL = os.getenv("JIRA_BASE_URL")
JIRA_USERNAME = os.getenv("JIRA_USERNAME")
JIRA_API_TOKEN = os.getenv("JIRA_API_TOKEN")
JIRA_PROJECT_KEY = os.getenv("JIRA_PROJECT_KEY")

def create_jira_ticket(summary, description):
    """Creates a JIRA ticket using the REST API."""
    url = f"{JIRA_BASE_URL}/rest/api/3/issue"
    headers = {
        "Content-Type": "application/json"
    }
    auth = (JIRA_USERNAME, JIRA_API_TOKEN)
    payload = {
        "fields": {
            "project": {"key": JIRA_PROJECT_KEY},
            "summary": summary,
            "description": description,
            "issuetype": {"name": "Task"}  # Change as needed
        }
    }
    response = requests.post(url, headers=headers, auth=auth, data=json.dumps(payload))
    return response

@app.route('/github-webhook', methods=['POST'])
def github_webhook():
    data = request.json

    # Ensure we're processing a comment creation or edit event
    if data.get("action") in ["created", "edited"]:
        comment_body = data.get("comment", {}).get("body", "")
        
        if comment_body.startswith("/ticket"):
            # Remove the command to extract ticket description
            ticket_description = comment_body[len("/ticket"):].strip()
            issue_title = data.get("issue", {}).get("title", "GitHub Issue Ticket")
            ticket_summary = f"Ticket from GitHub: {issue_title}"
            
            # Create the ticket in JIRA
            jira_response = create_jira_ticket(ticket_summary, ticket_description)
            
            if jira_response.status_code in [200, 201]:
                # Provide successful feedback back to the webhook caller
                return jsonify({"message": "JIRA ticket created successfully"}), 200
            else:
                return jsonify({
                    "error": "Failed to create JIRA ticket",
                    "details": jira_response.text
                }), 500

    return jsonify({"message": "No action taken"}), 200

if __name__ == '__main__':
    port = int(os.getenv("FLASK_PORT", 5000))
    debug = os.getenv("FLASK_DEBUG", "False") == "True"
    app.run(port=port, debug=debug)
```

## Testing the Workflow

1. **Open a New Issue on GitHub**  
   Create a new issue in your repository.

2. **Post a Comment with `/ticket`**  
   In the comments of the issue, type:
   ```
   /ticket
   The deployment pipeline is failing intermittently. Please investigate.
   ```
   
3. **GitHub Webhook Trigger**  
   GitHub sends the comment data as a JSON payload to your Flask endpoint. The payload includes the comment body, issue title, and more.

4. **JIRA Ticket Creation**  
   The Flask server extracts the comment (removing `/ticket`), constructs a ticket summary and description, and sends a POST request to JIRA.

5. **Feedback Handling**  
   - **On Success:** JIRA returns a success status (200/201), and the Flask server responds with a JSON message:  
     `{"message": "JIRA ticket created successfully"}`
   - **On Failure:** The response includes error details from JIRA, aiding troubleshooting.
    
6. **Verify in JIRA**  
   Check your JIRA board to confirm the new ticket was created with the expected summary and description.

## Deployment Considerations

- **Production Hosting:** Deploy the Flask application on a secure host (with HTTPS), perhaps behind a reverse proxy.
- **Containerization:** Dockerize the application if needed, and deploy using orchestrators like Kubernetes.
- **Security:**  
  - Validate the webhook payloads using GitHub’s secret signature mechanism.
  - Secure the JIRA API endpoint and restrict access as needed.
- **Logging:**  
  Optionally, log results to a centralized system or update the GitHub issue with feedback.

## Conclusion

This project demonstrates a complete workflow where a GitHub comment command `/ticket` automatically generates a corresponding JIRA ticket. It details how comment data is extracted from GitHub, processed by a Flask webhook, sent to JIRA, and how the response from JIRA is communicated back. This integration streamlines issue tracking between development and operations teams, saving time and reducing manual effort.

Happy automating! 🚀
```
