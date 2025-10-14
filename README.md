# AI Conversational Admissions Assistant

A conversational AI system designed to help university staff and students get quick, accurate answers to their questions about university services, policies, and procedures.

> **Private Repository Notice**
> This is an ongoing private project containing proprietary and confidential code. This repository is restricted to authorized personnel only and is not intended for public access or distribution.

<img src="public/images/chat-screen-1.png" width="600" alt="Chat Interface" />
<img src="public/images/chat-screen-b.png" width="600" alt="Chat Interface Features" />

## What This System Does

The chat assistant provides instant, helpful responses to questions about:
- Academic policies and procedures
- Student services and support
- IT systems and access
- Financial aid and fees
- Campus life and facilities
- And much more

## How It Works

### For Students and Staff
1. **Ask a question** - Type your question in plain English
2. **Get instant answers** - The system searches our knowledge base and provides focused, relevant answers
3. **Follow up naturally** - Ask follow-up questions like "tell me more about that" or "what was the first option?"
4. **Click suggestions** - When available, click suggested related questions for quick, focused answers

### Key Features
- **Real-time streaming** - Answers appear as they're generated for fast response
- **Conversation memory** - The system remembers your conversation context
- **Smart suggestions** - Get related questions to explore topics further
- **Reliable answers** - Built on comprehensive university knowledge base
- **Privacy-focused** - Conversation data expires automatically for your privacy

## Key Technologies

- **Backend**: Python, FastAPI, MySQL, Weaviate, Redis, OpenAI
- **Frontend**: Next.js, React, TypeScript, Tailwind CSS
- **AI/ML**: OpenAI GPT models, vector embeddings, hybrid search (vector + BM25)
- **Infrastructure**: Redis for session management

## Technical Documentation

For detailed technical information about how the system works end-to-end, see:
**[End-to-End System Overview](backend/system-docs/end-to-end-system-overview.md)**

## Project Structure

```
├── backend/                 # FastAPI backend service
│   ├── src/app/            # Main application code
│   ├── system-docs/        # System documentation
│   └── tests/              # Backend tests
├── frontend/               # Next.js frontend
│   ├── app/                # Main application pages
│   ├── components/         # React components
│   └── lib/                # Utility libraries
└── README.md               # This file
```

## Getting Started

### For End Users (Students & Staff)
Simply visit the chat interface and start asking questions. No login required - the system works immediately.

### For Developers
This project consists of:
- **Backend**: FastAPI-based chat service with vector search and LLM integration
- **Frontend**: Next.js chat interface with real-time streaming
- **Knowledge Base**: MySQL + Weaviate for FAQ storage and hybrid search (vector + BM25)

## Completed Enhancements

### Completed: Vector Database Migration
- **Status**: COMPLETED - Migrated from ChromaDB to Weaviate
- **Current**: Weaviate with validated hybrid search (vector + BM25)
- **Benefits Achieved**:
  - Hybrid search combining semantic understanding with keyword matching
  - 100% precision on validated test queries
  - ChromaDB remains available as fallback option

### Completed: Performance Evaluation System
- **Status**: COMPLETED - Comprehensive evaluation framework implemented
- **Features**:
  - Automated search quality evaluation (Precision@1, Hit Rate@5, MRR, similarity scores)
  - Answer quality validation (topic coverage, term compliance, answerability rate)
  - Real data testing using production FAQ database
  - Integration tests for CI/CD quality gates
  - Detailed reporting with JSON and Markdown outputs
- **Results**: System achieves 88.2% Precision@1 on full evaluation dataset (34 tests)

## Support

For technical issues or questions about the system:
- Check the [End-to-End System Overview](backend/system-docs/end-to-end-system-overview.md) for detailed technical information
- Contact the development team for system-specific issues

For questions about university policies or services:
- Use the chat assistant - it's designed to help with these questions!
- Contact relevant university departments for complex or specific inquiries

---

*This system is designed to provide quick, accurate answers to common university questions. For complex or sensitive matters, please contact the appropriate university department directly.*
