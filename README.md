
# RAG AI Digital Twin with n8n & Supabase

A Retrieval-Augmented Generation (RAG) system built with **n8n**, **Supabase Vector Store**, and **Google Gemini Models**.

This repository contains two main workflows:

1. **ETL Data Pipeline (Ingestion Workflow)**: Automatically fetches structured records from a database, formats them into semantically rich documents, chunks them using recursive text splitting, and populates the Supabase Vector Store.
2. **AI Agent Assistant (RAG Query Workflow)**: A conversational AI agent equipped with short-term memory and real-time retrieval capabilities to answer queries accurately using facts stored in the knowledge base.

---

## 🛠 Tech Stack

* **Workflow Orchestration**: n8n
* **Database & Vector Store**: Supabase (PostgreSQL with `pgvector`)
* **LLM & Embeddings**: Google Gemini (`models/gemini-3.5-flash-lite` & `models/gemini-embedding-2`)
* **Data Processing**: LangChain Integration (Recursive Text Splitters, Memory Buffer, Retrieval Tools)

---

## ⚙️ Workflows Overview

### Workflow 1: Dynamic Data Ingestion Pipeline (ETL)

This workflow extracts unstructured or structured user profile data from a Supabase database, cleans/formats it into LLM-friendly text, chunks it cleanly, and generates vector embeddings for storage.

![ETL Data Pipeline](ETL%20Data%20Pipeline.png)

```
[ Manual Trigger ] ➔ [ Supabase (Get) ] ➔ [ JavaScript Transformation ]
                                                      │
                                           [ Supabase Vector Store ]
                                                      │
                                           [ Default Data Loader ]
                                                      │
                                    [ Recursive Text Splitter (1000/100) ]

```

#### Key Components:

* **Supabase Fetch Node**: Pulls raw records dynamically from your source table (e.g., `profiles`).
* **JavaScript Transformation Node**: Formats raw JSON key-value pairs into readable Markdown text sections to prevent fragmented or lost context during vector search.
* **Recursive Character Text Splitter**: Splits long text dynamically (Recommended: `Chunk Size: 1000`, `Chunk Overlap: 100`).
* **Embeddings & Vector Store**: Converts text chunks into high-dimensional vectors via Gemini Embeddings and saves them into the `knowledgebase` table.

---

### Workflow 2: RAG AI Agent Assistant

An interactive AI Digital Twin that uses real-time vector search to answer user questions about a user's skills, background, or experience.

📹 **Demo Video**:

<video width="100%" controls autoplay muted loop>
  <source src="AI Agent Assistant (RAG Query Workflow).mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

```
[ Chat Trigger ] ➔ [ AI Agent (Gemini Flash) ] ➔ [ Response to User ]
                        ├── [ Simple Memory (Window) ]
                        └── [ Supabase Vector Store Tool ]

```

#### Key Components:

* **Chat Trigger**: Receives incoming user inquiries.
* **AI Agent**: Powered by Gemini LLM with system prompts instructing it to enforce grounding via tool usage.
* **Vector Store Tool**: Grants the AI Agent full semantic search capabilities across the vector database.
* **Window Memory**: Retains conversation history for coherent follow-up responses.

---

## 📦 Setup & Installation

### 1. Prerequisites

* An active **n8n** instance (Self-hosted or Cloud).
* A **Supabase** project with the `pgvector` extension enabled.
* A **Google Gemini API Key**.

### 2. Database Schema Setup

Run the following SQL query in your Supabase SQL Editor to set up the vector store table and match function:

```sql
-- Enable the pgvector extension
create extension if not exists vector;

-- Create knowledgebase table
create table knowledgebase (
  id uuid primary key default gen_random_uuid(),
  content text,
  metadata jsonb,
  embedding vector(768) -- Matches Gemini Embedding dimensions
);

-- Create match function for vector similarity search
create or replace function match_documents (
  query_embedding vector(768),
  match_count int default 5,
  filter jsonb default '{}'
) returns table (
  id uuid,
  content text,
  metadata jsonb,
  similarity float
) language plpgsql as $$
begin
  return query
  select
    knowledgebase.id,
    knowledgebase.content,
    knowledgebase.metadata,
    1 - (knowledgebase.embedding <=> query_embedding) as similarity
  from knowledgebase
  where knowledgebase.metadata @> filter
  order by knowledgebase.embedding <=> query_embedding
  limit match_count;
end;
$$;

```

---

## 💻 Customizing the Code Node (Data Transformation)

To convert generic database JSON inputs into clean Markdown without hardcoding personal data, place code similar to the template below into the **Code Node** in Workflow 1:

```javascript
// Get raw item from Supabase
const rawItem = $input.first().json;
const profile = rawItem.data || rawItem;

// Helper function to format arrays
const fmt = (arr) => Array.isArray(arr) ? arr.join(', ') : (arr || 'N/A');

// Build structured Markdown sections
const formattedText = `
# Candidate Profile: ${profile.full_name || 'Candidate'}

## Core Overview
- **Role**: ${profile.current_role || 'N/A'}
- **Location**: ${profile.location || 'N/A'}
- **Email**: ${profile.email || 'N/A'}

## Technical Skills & Stack
- **Technologies**: ${fmt(profile.tech_stack)}
- **Skills Matrix**: ${typeof profile.skills === 'object' ? JSON.stringify(profile.skills) : fmt(profile.skills)}

## Summary & Experience
- **Summary**: ${profile.summary || 'N/A'}
- **Background**: ${profile.bio || 'N/A'}
`.trim();

// Output as standard document item for LangChain Vector Store
return [
  {
    json: {
      pageContent: formattedText
    }
  }
];

```

> **Tip for LLM Prompting**: If you want an AI to generate custom JavaScript for your specific database fields, use the prompt below:
> *"Write a JavaScript code snippet for an n8n Code Node that accepts an input object `rawItem.data`, iterates over its key-value pairs, formats nested objects/arrays into clean Markdown headings and bullet points, and returns an array of objects containing `{ json: { pageContent: markdownString } }`."*


