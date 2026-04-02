# API Documentation

This document describes all available endpoints for the **AI Knowledge Indexing System**.

## 1. Authentication Endpoints

These endpoints are strictly for generating JSON Web Tokens (JWTs) representing logged-in users.

### `POST /api/auth/register`
**Description**: Registers a new user. The password gets hashed (bcrypt) and persisted.
- **Request Body**:
  ```json
  {
    "username": "user1",
    "password": "securepassword123"
  }
  ```
- **Response** `200 OK`:
  ```json
  {
    "access_token": "eyJhbGciOiJIUzI1...",
    "token_type": "bearer"
  }
  ```

### `POST /api/auth/login`
**Description**: Authenticates an existing user and retrieves a JWT valid for 7 days.
- **Request Body**:
  ```json
  {
    "username": "user1",
    "password": "securepassword123"
  }
  ```
- **Response** `200 OK`: Same as register.

---

## 2. Chat & Document Upload Endpoints

> **Note**: All endpoints below require an `Authorization` header containing a valid Bearer token.
>
> `Authorization: Bearer <your-access-token>`

### `POST /api/chats/upload-pdf`
**Description**: Accepts a raw `.pdf` file upload as `multipart/form-data`, extracts the text, slices it via semantic chunks, stores the metadata into SQL, and ingests the Vector embeddings directly into Qdrant.
- **Request Body** (FormData):
  - `file`: The `.pdf` file.
- **Response** `200 OK`:
  ```json
  {
    "message": "File uploaded and indexed successfully",
    "file_id": 14
  }
  ```

### `GET /api/chats/`
**Description**: Retrieves a list of all active chat sessions belonging to the logged-in user.
- **Response** `200 OK`:
  ```json
  [
    {
      "id": 1,
      "title": "Discussion about Law Case A",
      "file_id": 14,
      "created_at": "2026-04-02T22:45:00Z"
    }
  ]
  ```

### `POST /api/chats/`
**Description**: Instantiates a fresh chat context specifically tied to a previously uploaded `file_id`.
- **Request Body**:
  ```json
  {
    "title": "Discussion about Law Case A",
    "file_id": 14
  }
  ```
- **Response** `200 OK`:
  ```json
  {
    "chat_id": 1
  }
  ```

### `GET /api/chats/{chat_id}`
**Description**: Retrieves the properties of a specific active chat session. 
- **Response** `200 OK`:
  ```json
  {
    "id": 1,
    "user_id": 4,
    "title": "Discussion about Law Case A",
    "file_id": 14,
    "created_at": "2026-04-02T22:45:00Z"
  }
  ```

### `DELETE /api/chats/{chat_id}`
**Description**: Deletes a specific chat session. Since cascades are active on the database, this securely destroys message history concurrently.

---

## 3. Inference / Messaging Endpoints

### `GET /api/chats/{chat_id}/messages`
**Description**: Rebuilds the entire exact message history associated with a chat session (including user prompts and LLM-generated assistant responses).
- **Response** `200 OK`:
  ```json
  [
    {
      "id": 12,
      "role": "user",
      "content": "What is the verdict mentioned?",
      "created_at": "2026-04-02T22:50:00Z"
    },
    {
      "id": 13,
      "role": "assistant",
      "content": "Based on the provided context, the verdict was GUILTY.",
      "created_at": "2026-04-02T22:50:04Z"
    }
  ]
  ```

### `POST /api/chats/{chat_id}/messages`
**Description**: Appends a new user message to the conversation, dynamically loads the last 10 historical conversation turns for semantic tracking, searches the Qdrant Vector database for appropriate chunks matching the `file_id` constraints, and prompts GPT-4o-mini to render an accurate response with references mapped.
- **Request Body**:
  ```json
  {
    "content": "Who is the primary suspect?"
  }
  ```
- **Response** `200 OK`:
  ```json
  {
    "message": "Based on the internal context mapping, John Doe was listed as the primary suspect.",
    "references": [
      { "text": "...John Doe was brought precisely into custody at 9 PM..." }
    ]
  }
  ```
