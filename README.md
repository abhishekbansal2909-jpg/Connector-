# 🎙️ Real-Time Voice QA Agent

An ultra-low-latency, interruptible Voice AI agent built with **FastAPI**, **Twilio**, **Deepgram**, **Groq**, and **ElevenLabs**. 

This project demonstrates a production-ready bidirectional voice pipeline. It handles live telephony audio streams over WebSockets, executes near-instantaneous speech-to-text (STT) and text-to-speech (TTS), and features intelligent "barge-in" capabilities, allowing users to interrupt the AI mid-sentence for a natural conversational experience.

---

## 🏗️ System Architecture

The core of this system relies on a single **FastAPI WebSocket bridge** that orchestrates real-time audio and text chunking between four distinct APIs without saving intermediary files to disk.

```mermaid
graph TD
    User((📱 Caller)) <-->|Raw μ-law Audio| Twilio[Twilio Media Stream]
    Twilio <-->|WebSocket base64| FastAPI[FastAPI Server]
    
    subgraph AI Pipeline
    FastAPI -->|WebSocket| Deepgram[Deepgram STT]
    Deepgram -->|Real-time Transcripts| FastAPI
    
    FastAPI -->|REST API| Groq[Groq LLaMA-3.3 70B]
    Groq -->|Text Output| FastAPI
    
    FastAPI -->|WebSocket| ElevenLabs[ElevenLabs TTS]
    ElevenLabs -->|μ-law Audio Bytes| FastAPI
    end
