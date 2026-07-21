import os
import json
import base64
import asyncio
import websockets
from fastapi import FastAPI, Request, Response, WebSocket, WebSocketDisconnect
from twilio.rest import Client
from groq import Groq

app = FastAPI()

# Load Environment Variables
TWILIO_ACCOUNT_SID = os.getenv("TWILIO_ACCOUNT_SID")
TWILIO_AUTH_TOKEN = os.getenv("TWILIO_AUTH_TOKEN")
TWILIO_PHONE_NUMBER = os.getenv("TWILIO_PHONE_NUMBER")
MY_PHONE_NUMBER = os.getenv("MY_PHONE_NUMBER")
DEEPGRAM_API_KEY = os.getenv("DEEPGRAM_API_KEY")
GROQ_API_KEY = os.getenv("GROQ_API_KEY")

# Initialize Groq Client
groq_client = Groq(api_key=GROQ_API_KEY)

# Simple conversation memory
system_prompt = {
    "role": "system", 
    "content": "You are a witty, concise, and helpful AI voice assistant on a phone call. Keep your answers brief, natural, and under 2 sentences so the call flows fast."
}


async def generate_groq_response(user_text: str):
    """Sends the transcribed user text to Groq and prints the reply."""
    try:
        print(f"🧠 Thinking response for: '{user_text}'...", flush=True)
        
        # Call Groq's high-speed Llama 3.3 model
        completion = groq_client.chat.completions.create(
            model="llama-3.3-70b-versatile",
            messages=[
                system_prompt,
                {"role": "user", "content": user_text}
            ],
            temperature=0.7,
            max_tokens=100
        )
        
        reply = completion.choices[0].message.content
        print(f"🤖 AI Response: {reply}", flush=True)
        return reply

    except Exception as e:
        print(f"Groq API Error: {e}", flush=True)


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
    print("Twilio WebSocket connection opened!", flush=True)
    
    deepgram_url = "wss://api.deepgram.com/v1/listen?encoding=mulaw&sample_rate=8000&channels=1&model=nova-3&interim_results=true"
    
    try:
        async with websockets.connect(
            deepgram_url,
            additional_headers={"Authorization": f"Token {DEEPGRAM_API_KEY}"}
        ) as deepgram_ws:
            
            async def listen_to_deepgram():
                try:
                    async for message in deepgram_ws:
                        data = json.loads(message)
                        
                        if data.get("type") == "Results":
                            transcript = data.get("channel", {}).get("alternatives", [{}])[0].get("transcript", "")
                            
                            if transcript.strip():
                                if data.get("is_final"):
                                    print(f"✅ Final: {transcript}", flush=True)
                                    # Trigger Groq brain when a final sentence is captured
                                    asyncio.create_task(generate_groq_response(transcript))
                                else:
                                    print(f"⏳ Partial: {transcript}", flush=True)
                                    
                except Exception as e:
                    print(f"Deepgram listen error: {e}", flush=True)
                    
            asyncio.create_task(listen_to_deepgram())
            
            while True:
                data = await websocket.receive_text()
                msg = json.loads(data)
                
                if msg.get("event") == "start":
                    print("Twilio media stream started!", flush=True)
                
                elif msg.get("event") == "media":
                    payload = msg["media"]["payload"]
                    chunk = base64.b64decode(payload)
                    await deepgram_ws.send(chunk)
                
                elif msg.get("event") == "stop":
                    print("Twilio media stream stopped.", flush=True)
                    break
                    
    except WebSocketDisconnect:
        print("Twilio WebSocket disconnected.", flush=True)
    except Exception as e:
        print(f"Connection error: {e}", flush=True)
