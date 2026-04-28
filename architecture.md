FAISS-Based Outlet Matching System (Azure-Centric Architecture)
Overview

This system implements a hybrid outlet matching pipeline using vector similarity search. High-confidence matches are automatically resolved using FAISS, while lower-confidence matches are routed to a human-in-the-loop interface for manual verification.

The architecture is designed to:

Minimize manual matching effort
Scale efficiently with large datasets
Integrate with existing Snowflake data infrastructure
Leverage Azure-native services for compute-heavy workloads
High-Level Architecture
                ┌──────────────────────────────┐
                │  Azure Container App         │
                │  (FAISS + FastAPI Service)   │
                └──────────────┬───────────────┘
                               │
                        Vector Search API
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
        ▼                      ▼                      ▼
┌──────────────┐     ┌──────────────────┐    ┌──────────────────┐
│ Streamlit UI │     │ Embedding Service│    │ Batch Processing │
│ (Snowflake)  │     │ (Azure)          │    │ (ADF / Functions)│
└──────┬───────┘     └─────────┬────────┘    └─────────┬────────┘
       │                        │                       │
       └──────────────┬─────────┴──────────────┬────────┘
                      ▼                        ▼
               ┌────────────────────────────────────┐
               │          Snowflake DB              │
               │ (Source of truth + storage layer)  │
               └────────────────────────────────────┘
Core Components
1. FAISS Vector Search Service (Azure)

Deployment: Azure Container Apps / AKS
Tech Stack: FastAPI + FAISS

Responsibilities:

Maintain in-memory FAISS index
Perform nearest neighbor search
Return top-K similar matches with similarity scores

Endpoints:

POST /build_index
POST /add_vectors
POST /query
2. Embedding Service (Azure)

Tech Stack: Sentence Transformers

Responsibilities:

Convert outlet names → vector embeddings
Encode new data only (avoid recomputation)
Return embeddings for storage or indexing
3. Batch Processing Layer

Deployment Options:

Azure Functions
Azure Data Factory (ADF)
Scheduled container jobs

Responsibilities:

Detect new/unembedded records
Generate embeddings
Update FAISS index incrementally
4. Snowflake (Data Layer)

Role: Source of truth

Tables (suggested):

OUTLETS_RAW
OUTLETS_EMBEDDINGS
MATCH_RESULTS
MANUAL_REVIEW_QUEUE

Responsibilities:

Store outlet data
Store embeddings (optional but recommended)
Store match results
Feed Streamlit UI
5. Streamlit UI (Snowflake)

Deployment: Streamlit in Snowflake

Responsibilities:

Display outlet matching interface
Show FAISS recommendations
Allow manual matching
Write confirmed matches back to Snowflake
Data Flow
Step 1: Data Ingestion
New outlet data is inserted into Snowflake (OUTLETS_RAW)
Step 2: Embedding Generation
Batch job identifies new records
Calls embedding service
Stores embeddings in OUTLETS_EMBEDDINGS
Step 3: FAISS Index Update
New embeddings are sent to FAISS service
FAISS index updated incrementally (index.add())
Step 4: Matching (Query Flow)

When a match is needed:

Encode query outlet name
Call FAISS /query
Retrieve top-K matches
Apply confidence thresholds
Step 5: Decision Logic
Confidence Score	Action
> 0.90	Auto-match
0.75 – 0.90	Review queue
< 0.75	Manual matching
Step 6: Human-in-the-Loop
Streamlit displays:
Suggested matches
Similarity scores
User confirms or edits match
Results written to MATCH_RESULTS
Key Design Principles
1. Encode Once, Use Many Times
Avoid re-encoding existing data
Store embeddings persistently
2. Incremental Index Updates
Use FAISS index.add()
Avoid rebuilding index from scratch
3. Separation of Concerns
Embedding ≠ Search ≠ UI
Each component scales independently
4. Hybrid Matching Strategy
Automate high-confidence matches
Retain human validation for edge cases
5. Observability & Feedback Loop
Log:
Similarity scores
User corrections
Use data to improve thresholds/model later
Deployment Strategy
Phase 1 (MVP)
FAISS + FastAPI in Azure Container Apps
Basic embedding pipeline
Streamlit UI for manual review
Phase 2 (Scale)
Add autoscaling for FAISS service
Introduce queueing (e.g., Azure Service Bus)
Optimize embedding throughput
Phase 3 (Optimization)
Tune similarity thresholds
Evaluate alternative embedding models
Add active learning / retraining loop
Why Azure-Centric?
Advantages over Snowflake-only:
Better support for ML libraries (FAISS, transformers)
Faster container builds and startup
More flexible scaling
Cleaner architecture for production systems
Summary

This system combines:

FAISS (fast vector search)
Embeddings (semantic understanding)
Snowflake (data + UI)
Azure (compute + scalability)

to create a high-performance, scalable, and human-aware matching system.