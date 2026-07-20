from fastapi import FastAPI, Request, Response

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "The AI Voice Agent code is stored on GitHub!"}

@app.post("/incoming-call")
async def handle_incoming_call(request: Request):
    # This XML tells Twilio to read a message aloud to the caller
    twiml_response = """<?xml version="1.0" encoding="UTF-8"?>
    <Response>
        <Say>Hello! Your AI voice agent is successfully connected to your server.</Say>
    </Response>
    """
    return Response(content=twiml_response, media_type="application/xml")
