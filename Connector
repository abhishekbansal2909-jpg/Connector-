import os
import json
import base64
import asyncio
import websockets
from fastapi import FastAPI, Request, Response, WebSocket, WebSocketDisconnect
from twilio.rest import Client

# Ensure you retrieve DEEPGRAM_API_KEY alongside your Twilio keys
DEEPGRAM_API_KEY = os.getenv("DEEPGRAM_API_KEY")
TWILIO_ACCOUNT_SID = os.getenv("TWILIO_ACCOUNT_SID")
# ... [Keep your other variable loads here] ...

# ... [Keep your root, /trigger-call, and /incoming-call routes exactly the same] ...

@app.websocket("/media-stream")
async def handle_media_stream(websocket: WebSocket):
    await websocket.accept()
    print("Twilio WebSocket connection opened!")
    
    # Configure Deepgram to match Twilio's exact audio format
    deepgram_url = "wss://api.deepgram.com/v1/listen?encoding=mulaw&sample_rate=8000&channels=1&model=nova-3"
    
    try:
        # Open a secondary WebSocket directly to Deepgram
        async with websockets.connect(
            deepgram_url,
            extra_headers={"Authorization": f"Token {DEEPGRAM_API_KEY}"}
        ) as deepgram_ws:
            
            # 1. Background task to listen for Deepgram's text transcriptions
            async def listen_to_deepgram():
                try:
                    async for message in deepgram_ws:
                        data = json.loads(message)
                        # Deepgram sends 'is_final' when a sentence is complete
                        if data.get("is_final"):
                            transcript = data.get("channel", {}).get("alternatives", [{}])[0].get("transcript", "")
                            if transcript.strip():
                                print(f"🗣️ You said: {transcript}")
                except Exception as e:
                    print(f"Deepgram listen error: {e}")
                    
            # Fire up the listening task
            asyncio.create_task(listen_to_deepgram())
            
            # 2. Main loop: Receive Twilio audio and forward it to Deepgram
            while True:
                data = await websocket.receive_text()
                msg = json.loads(data)
                
                if msg.get("event") == "start":
                    print("Twilio media stream started!")
                
                elif msg.get("event") == "media":
                    # Extract the base64 audio payload and decode it into raw bytes
                    payload = msg["media"]["payload"]
                    chunk = base64.b64decode(payload)
                    
                    # Instantly stream the raw audio chunk to Deepgram
                    await deepgram_ws.send(chunk)
                
                elif msg.get("event") == "stop":
                    print("Twilio media stream stopped.")
                    break
                    
    except WebSocketDisconnect:
        print("Twilio WebSocket disconnected.")
    except Exception as e:
        print(f"Connection error: {e}")
