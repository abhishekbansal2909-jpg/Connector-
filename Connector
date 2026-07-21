import os
import json
import base64
import asyncio
import websockets
from fastapi import FastAPI, Request, Response, WebSocket, WebSocketDisconnect
from twilio.rest import Client

# 1. Initialize the FastAPI app instance first
app = FastAPI()

# 2. Load Environment Variables
TWILIO_ACCOUNT_SID = os.getenv("TWILIO_ACCOUNT_SID")
TWILIO_AUTH_TOKEN = os.getenv("TWILIO_AUTH_TOKEN")
TWILIO_PHONE_NUMBER = os.getenv("TWILIO_PHONE_NUMBER")
MY_PHONE_NUMBER = os.getenv("MY_PHONE_NUMBER")
DEEPGRAM_API_KEY = os.getenv("DEEPGRAM_API_KEY")


@app.get("/")
async def root():
    return {"message": "The AI Voice Agent code is stored on GitHub!"}


@app.get("/trigger-call")
async def trigger_call(request: Request):
    client = Client(TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN)
    host = request.url.hostname
    
    call = client.calls.create(
        to=MY_PHONE_NUMBER,
        from_=TWILIO_PHONE_NUMBER,
        url=f"https://{host}/incoming-call"
    )
    return {"message": f"Calling you now! Call ID: {call.sid}"}


@app.post("/incoming-call")
async def handle_incoming_call(request: Request):
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
    print("Twilio WebSocket connection opened!")
    
    # Configure Deepgram to receive Twilio's raw 8000Hz mu-law audio
    deepgram_url = "wss://api.deepgram.com/v1/listen?encoding=mulaw&sample_rate=8000&channels=1&model=nova-3"
    
    try:
        async with websockets.connect(
            deepgram_url,
            extra_headers={"Authorization": f"Token {DEEPGRAM_API_KEY}"}
        ) as deepgram_ws:
            
            # Listen for incoming transcripts from Deepgram
            async def listen_to_deepgram():
                try:
                    async for message in deepgram_ws:
                        data = json.loads(message)
                        if data.get("is_final"):
                            transcript = data.get("channel", {}).get("alternatives", [{}])[0].get("transcript", "")
                            if transcript.strip():
                                print(f"🗣️ You said: {transcript}")
                except Exception as e:
                    print(f"Deepgram listen error: {e}")
                    
            # Launch transcription listener in the background
            asyncio.create_task(listen_to_deepgram())
            
            # Stream audio payloads from Twilio directly to Deepgram
            while True:
                data = await websocket.receive_text()
                msg = json.loads(data)
                
                if msg.get("event") == "start":
                    print("Twilio media stream started!")
                
                elif msg.get("event") == "media":
                    payload = msg["media"]["payload"]
                    chunk = base64.b64decode(payload)
                    await deepgram_ws.send(chunk)
                
                elif msg.get("event") == "stop":
                    print("Twilio media stream stopped.")
                    break
                    
    except WebSocketDisconnect:
        print("Twilio WebSocket disconnected.")
    except Exception as e:
        print(f"Connection error: {e}")
