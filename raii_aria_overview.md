# Ratiio Backend: RAII Tech & ARIA Overview

This document provides a brief overview of the Ratiio backend architecture, specifically focusing on the integration of **ARIA** (the core AI model) and **RAII Tech** (the client-side encryption protocol).

## 🧠 ARIA (The AI Engine & Agentic Workflow)

The backend is built around **ARIA**, a proprietary, voice-first AI model designed to function as a personal, autonomous agent (striving towards AGI-like capabilities). Instead of a simple request-response model, ARIA operates as an intelligent agent with persistent memory and conversational contextualization.

### How ARIA functions as an Agent / AGI
- **Persistent Memory (Agentic State):** ARIA acts as a continuous agent by extracting and embedding user interactions into long-term memory. It stores these in a Qdrant Vector DB (384-dimensional vectors). When a user speaks, the agent queries this DB to pull the top 5 relevant memories, giving it long-term context and a sense of "state" across sessions.
- **Tiered Intelligence:** Routes users based on required agentic reasoning capabilities:
  - **Free Tier:** ARIA 1.7B (Faster, lightweight assistant)
  - **Pro Tier:** ARIA 7B (More complex reasoning, deeper context window)
- **Voice-First Agent Pipeline:** The system chains multiple tools together, allowing the agent to "hear" and "speak":
  1. **Detection:** Silero VAD (Voice Activity Detection)
  2. **Transcription:** Whisper STT (Speech-to-Text)
  3. **Security Check:** 5-layer Threat Detector (protects the agent against injections/abuse)
  4. **Memory Retrieval:** Qdrant Vector DB fetches contextual memories.
  5. **Generation:** ARIA via vLLM generates the response based on prompt + retrieved memory.
  6. **Synthesis:** ElevenLabs TTS (Text-to-Speech) voices the response.

---

## 🔐 RAII Tech (Zero-Knowledge Architecture)

**RAII Tech** (Resource Acquisition Is Initialization) defines the strict client-side encryption protocol (v1.0). Its core principle is that **the server never sees plaintext data**. The server only stores opaque ciphertext and wrapped keys.

### Protocol Breakdown

1. **Key Derivation (Argon2id):**
   - A Key Encryption Key (KEK) is derived from the user's password using the Argon2id hashing algorithm. 
   - The KEK never leaves the client device; only a non-secret salt is shared with the server.

2. **File Encryption (AES-256-GCM):**
   - Each file (e.g., audio, transcripts) generates a random 32-byte Data Encryption Key (DEK).
   - The file is encrypted on the client device using this DEK.

3. **Key Wrapping:**
   - The DEK is encrypted (wrapped) using the KEK so that only the user can recover it.

4. **Upload & Download:**
   - The server receives the encrypted payload (ciphertext + wrapped DEK + salts/nonces).
   - Upon download, the client re-derives the KEK from the user's password, unwraps the DEK, and decrypts the file.

5. **Cryptographic Erasure (Deletion):**
   - When a user deletes a file, the server destroys the wrapped DEK. 
   - Even though encrypted fragments might physically remain on disk, they become mathematically indistinguishable from random bytes and permanently unrecoverable.

---

## 🛡️ Aegis / Threat Detection (Security Shield)

The backend features a robust 5-layer threat detection system that acts as the platform's "Aegis" (security shield), actively defending against malicious inputs and prompt injections before they reach the ARIA engine. 

### The 5 Layers of Detection:
1. **Keyword Scan:** Scans for prompt injection patterns and malicious commands.
2. **Length Abuse:** Blocks inputs exceeding 2000 characters to prevent buffer/context flooding.
3. **Encoding Detection:** Identifies obfuscation attempts via Base64, hex, or URL encoding.
4. **Repetition Attack:** Detects repetitive phrases designed to break the model's generation limits.
5. **Unicode Abuse:** Prevents zero-width character attacks and RTL overrides.

Threats are scored from Level 1 to 5. Levels 1-3 are logged and allowed (with warnings), while Levels 4-5 are immediately blocked and trigger a security alert.

---

## 🏛️ System Architecture Highlights
- **100% Asynchronous:** Built entirely on FastAPI with non-blocking I/O (asyncpg for DB operations, async Redis connections).
- **Scalability:** Stateless API pods, utilizing PostgreSQL (relational data), Redis (caching/rate limits/sessions), and Qdrant (vectors). 
- **Security First:** Features a robust 5-layer threat detection system, rate limiting, and JWT blacklisting.
