import os
import requests
from flask import Flask, request, jsonify

app = Flask(__name__)

# Configuration from environment variables
META_VERIFY_TOKEN = os.environ.get("META_VERIFY_TOKEN", "my_secret_verify_token_123")
META_PAGE_ACCESS_TOKEN = os.environ.get("META_PAGE_ACCESS_TOKEN")
MANUS_API_KEY = os.environ.get("MANUS_API_KEY")
MANUS_API_BASE_URL = "https://api.manus.ai/v2"

# In-memory map for user sessions (In production, use a database)
user_task_map = {}

@app.route("/", methods=["GET"])
def home():
    return "Manus AI - Meta Integration Bridge is running!", 200

@app.route("/webhook", methods=["GET", "POST"])
def webhook():
    if request.method == "GET":
        # Meta Webhook Verification
        mode = request.args.get("hub.mode")
        token = request.args.get("hub.verify_token")
        challenge = request.args.get("hub.challenge")

        if mode == "subscribe" and token == META_VERIFY_TOKEN:
            print("WEBHOOK_VERIFIED")
            return challenge, 200
        return "Verification failed", 403

    elif request.method == "POST":
        data = request.json
        if data.get("object") == "page":
            for entry in data.get("entry", []):
                for messaging_event in entry.get("messaging", []):
                    sender_id = messaging_event.get("sender", {}).get("id")
                    if messaging_event.get("message") and not messaging_event["message"].get("is_echo"):
                        message_text = messaging_event["message"].get("text")
                        if message_text:
                            handle_incoming_message(sender_id, message_text)
        return "EVENT_RECEIVED", 200

def handle_incoming_message(sender_id, text):
    task_id = user_task_map.get(sender_id)
    
    headers = {
        "Content-Type": "application/json",
        "x-manus-api-key": MANUS_API_KEY
    }

    if not task_id:
        # Create a new Manus AI task
        payload = {
            "message": {"content": [{"type": "text", "text": text}]},
            "interactive_mode": True,
            "hide_in_task_list": True
        }
        response = requests.post(f"{MANUS_API_BASE_URL}/task.create", headers=headers, json=payload).json()
        if response.get("ok"):
            user_task_map[sender_id] = response["task_id"]
            # We'll wait for the Manus webhook to send the response back to Meta
    else:
        # Send follow-up message to existing task
        payload = {
            "task_id": task_id,
            "message": {"content": [{"type": "text", "text": text}]}
        }
        requests.post(f"{MANUS_API_BASE_URL}/task.sendMessage", headers=headers, json=payload)

@app.route("/manus-callback", methods=["POST"])
def manus_callback():
    data = request.json
    event_type = data.get("event_type")
    task_detail = data.get("task_detail", {})

    if event_type == "task_stopped":
        task_id = task_detail.get("task_id")
        message = task_detail.get("message", "")
        
        # Find the sender_id associated with this task_id
        sender_id = next((k for k, v in user_task_map.items() if v == task_id), None)
        
        if sender_id and message:
            send_meta_message(sender_id, message)
            
            # If task is finished, we could optionally clear the mapping
            if task_detail.get("stop_reason") == "finish":
                # del user_task_map[sender_id]
                pass

    return "OK", 200

def send_meta_message(recipient_id, text):
    url = f"https://graph.facebook.com/v19.0/me/messages?access_token={META_PAGE_ACCESS_TOKEN}"
    payload = {
        "recipient": {"id": recipient_id},
        "message": {"text": text}
    }
    requests.post(url, json=payload)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 5000)))
