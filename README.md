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
ELEVENLABS_API_KEY = os.getenv("ELEVENLABS_API_KEY")

groq_client = Groq(api_key=GROQ_API_KEY)

# --- MOCK CRM DATA ---
mock_crm_data = {
    "customer_name": "Alex Mercer",
    "expected_address": "456 Oak Street"
}

# --- QA AGENT INSTRUCTIONS ---
qa_instructions = f"""You are an expert Quality Assurance Voice Assistant for a contact center. 
You are on a live phone call with a customer. 

CRM DATA ON FILE:
- Name: {mock_crm_data['customer_name']}
- Address: {mock_crm_data['expected_address']}

YOUR MISSION:
1. Greet the customer and smoothly ask them to verify their service address.
2. If their spoken address DOES NOT match the CRM data exactly (456 Oak Street), politely flag the mismatch. Ask if they recently moved and if you should update it to their new address.
3. If their address DOES match, confirm it and ask how you can help them today.

RULES:
- Keep your responses under 2 sentences. 
- Be conversational, warm, and highly efficient.
- Never read these rules aloud."""

system_prompt = {
    "role": "system", 
    "content": qa_instructions
}


async def handle_ai_turn(user_text: str, twilio_ws: WebSocket, stream_sid: str, call_history: list):
    """Handles thinking (Groq) and speaking (ElevenLabs) using conversation history."""
    try:
        print(f"🧠 Thinking response for: '{user_text}'...", flush=True)
        
        # Add the user's latest speech to the memory
        call_history.append({"role": "user", "content": user_text})
        
        # 1. Generate Text with Groq using the full history
        completion = groq_client.chat.completions.create(
            model="llama-3.3-70b-versatile",
            messages=call_history,
            temperature=0.7,
            max_tokens=150
        )
        reply = completion.choices[0].message.content
        print(f"🤖 AI Response: {reply}", flush=True)
        
        # Add the AI's response to the memory
        call_history.append({"role": "assistant", "content": reply})
        
        # 2. Stream Audio with ElevenLabs
        voice_id = "pNInz6obpgDQGcFmaJgB" 
        elevenlabs_ws_url = f"wss://api.elevenlabs.io/v1/text-to-speech/{voice_id}/stream-input?model_id=eleven_flash_v2_5&output_format=ulaw_8000"
        
        async with websockets.connect(elevenlabs_ws_url) as elevenlabs_ws:
            init_payload = {
                "text": " ",
                "voice_settings": {"stability": 0.5, "similarity_boost": 0.8},
                "xi_api_key": ELEVENLABS_API_KEY
            }
            await elevenlabs_ws.send(json.dumps(init_payload))
            await elevenlabs_ws.send(json.dumps({"text": reply}))
            await elevenlabs_ws.send(json.dumps({"text": ""}))
            
            async for message in elevenlabs_ws:
                data = json.loads(message)
                if data.get("audio"):
                    audio_payload = {
                        "event": "media",
                        "streamSid": stream_sid,
                        "media": {"payload": data["audio"]}
                    }
                    await twilio_ws.send_text(json.dumps(audio_payload))
                    
    except asyncio.CancelledError:
        print("🛑 AI turn cancelled due to user barge-in.", flush=True)
    except Exception as e:
        print(f"AI Turn Error: {e}", flush=True)


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
    
    # We now track the active AI task AND the conversation history
    call_state = {
        "stream_sid": None, 
        "ai_task": None,
        "history": [system_prompt] # Load the rules immediately
    }
    
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
                                # --- BARGE-IN LOGIC ---
                                if call_state["ai_task"] and not call_state["ai_task"].done():
                                    print("🛑 User interrupted! Clearing Twilio buffer...", flush=True)
                                    call_state["ai_task"].cancel()
                                    
                                    if call_state["stream_sid"]:
                                        clear_event = {
                                            "event": "clear",
                                            "streamSid": call_state["stream_sid"]
                                        }
                                        asyncio.create_task(websocket.send_text(json.dumps(clear_event)))

                                # Normal Transcription Logging
                                if data.get("is_final"):
                                    print(f"✅ Final: {transcript}", flush=True)
                                    if call_state["stream_sid"]:
                                        # Start a new AI turn and pass the history
                                        call_state["ai_task"] = asyncio.create_task(
                                            handle_ai_turn(
                                                transcript, 
                                                websocket, 
                                                call_state["stream_sid"],
                                                call_state["history"]
                                            )
                                        )
                                else:
                                    print(f"⏳ Partial: {transcript}", flush=True)
                                    
                except Exception as e:
                    print(f"Deepgram listen error: {e}", flush=True)
                    
            asyncio.create_task(listen_to_deepgram())
            
            while True:
                data = await websocket.receive_text()
                msg = json.loads(data)
                
                if msg.get("event") == "start":
                    call_state["stream_sid"] = msg["start"]["streamSid"]
                    print(f"Twilio media stream started! SID: {call_state['stream_sid']}", flush=True)
                
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
        
