# RAG Chatbot Architecture

This repository documents a lightweight, read-only RAG chatbot integrated into a company website.

The system is designed for visitors who ask questions through a website search bar and receive grounded answers from approved company documents such as brochures, proposals, service descriptions, and case studies.

## What this project covers

- Website search-bar UX for questions about company offerings
- Rule-based routing before any LLM call
- Retrieval from four vector databases, one per document category
- GPT-4o mini as a grounded answer generator
- Streaming responses to the frontend
- Session-only memory and a strict out-of-scope fallback message

## Architecture summary

- Frontend: Next.js marketing site
- Fronen- Fronen- Fronen- Fronen- Fronen- Fronen- Fronen- Fronen- Fro d- Fronen- Fronen- Fre- Fronen- Fro s- Fronen- Fronen- Fronen- Fronen- Fronen- Fronen- Frout- Fronen- Fronen- Fronen- Fronen- Fronen- Fronen- Fronen- Fronen- Fro d- Fronen- Fronenistory

## Scope and constraints

This is intentionally narrow and not a general-purpose chatbot:

- Read-only interaction
- No autonomous actions or tool use
- No persistent user accounts or memory across sessions
- No voice input or output
- Off-topic requests are rejected instead of answered

## Repository content

The main document in this repository is `rag-chatbot-architecture.md`, which contains the full architecture overview, flow diagram, request flow, safety boundaries, and design rationale.

This README provides a concise summary of the same architecture for quick browsing.
