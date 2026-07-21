import os
import json
from fastapi import FastAPI, Request, Response, WebSocket, WebSocketDisconnect
from twilio.rest import Client

app = FastAPI()

# Retrieve credentials from Environment Variables
TWILIO_ACCOUNT_SID = os.getenv("TWILIO_ACCOUNT_SID")
TWILIO_AUTH_TOKEN = os.getenv("TWILIO_AUTH_TOKEN")
TWILIO_PHONE_NUMBER = os.getenv("TWILIO_PHONE_NUMBER")
MY_PHONE_NUMBER = os.getenv("MY_PHONE_NUMBER")


@app.get("/")
async def root():
    return {"message": "The AI Voice Agent code is stored on GitHub!"}


@app.get("/trigger-call")
async def trigger_call(request: Request):
    client = Client(TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN)
    host = request.url.hostname
    
    # Place an outbound call pointing to our /incoming-call webhook
    call = client.calls.create(
        to=MY_PHONE_NUMBER,
        from_=TWILIO_PHONE_NUMBER,
        url=f"https://{host}/incoming-call"
    )
    return {"message": f"Calling you now! Call ID: {call.sid}"}


@app.post("/incoming-call")
async def handle_incoming_call(request: Request):
    # Dynamically grab your Render domain to establish the secure WebSocket connection
    host = request.url.hostname
    twiml_response = f"""<?xml version="1.0" encoding="UTF-8"?>
    <Response>
        <Connect>
            <Stream url="wss://{host}/media-stream" />
        </Connect>
    </Response>
    """
    return Response(content=twiml_response, media_type="application/xml")


@app.websocket("/media-stream")
async def handle_media_stream(websocket: WebSocket):
    await websocket.accept()
    print("WebSocket connection opened!")
    
    try:
        while True:
            # Continuously receive real-time audio event payloads from Twilio
            data = await websocket.receive_text()
            msg = json.loads(data)
            
            if msg.get("event") == "start":
                print("Twilio media stream started!")
            elif msg.get("event") == "media":
                # Incoming base64-encoded audio payload from Twilio
                pass 
            elif msg.get("event") == "stop":
                print("Twilio media stream stopped.")
                break
                
    except WebSocketDisconnect:
        print("WebSocket disconnected.")
