## ✅ 1. WebSocket JSON Payloads (`/call/:id`)

### 🎙️ User → Backend (via WebSocket)

**Text message:**

```json
{
  "type": "text",
  "content": "I've had chest pain for two days."
}
```

---

### 🤖 Backend → Frontend

**Response from AI model (to speak via avatar):**

```json
{
  "type": "ai",
  "content": "How long have you had the chest pain?",
  "meta": {
    "source": "model 4b",
    "timestamp": "2025-07-04T08:44:32.127Z"
  }
}
```

---

## ✅ 2. Redis Conversation Log Format

Each session is stored as a list under `chat:<session_id>`, where each item is:

```json
{
  "role": "user" | "ai",
  "content": "The message content",
  "timestamp": "2025-07-04T08:44:32.127Z",
  "s3_doc_url": null | "https://s3.amazonaws.com/bucket/file.pdf"
}
```

Optional document links (when user uploads a file).

---

## ✅ 3. Document Upload API (REST)

**POST `/upload`**
Frontend sends a document (PDF, image, etc.) to backend:

```json
{
  "session_id": "abc123",
  "user_id": "user001",
  "filename": "blood_test.pdf",
  "content_base64": "<base64-encoded-content>"
}
```

---

##### 📤 Response from backend:

```json
{
  "status": "success",
  "s3_url": "https://firebase.storage/bucket/abc123/blood_test.pdf",
  "message": "Document uploaded successfully"
}
```

Backend saves this document URL into Redis using:

```json
{
  "role": "user",
  "content": "[Document Uploaded: blood_test.pdf]",
  "timestamp": "2025-07-04T08:45:00Z",
  "s3_doc_url": "https://firebase.storage/bucket/abc123/blood_test.pdf"
}
```

---

## ✅ 4. Final Gemini 27B Report Generation Input

When call ends, backend fetches Redis conversation and sends to Gemini 27B:

```json
{
  "conversation": [
    { "role": "user", "content": "I feel dizzy", "timestamp": "..." },
    { "role": "ai", "content": "Have you eaten today?", "timestamp": "..." },
    ...
  ],
  "documents": [
    {
      "name": "blood_test.pdf",
      "url": "https://firebase.storage/bucket/abc123/blood_test.pdf"
    }
  ]
}
```

---

### 🧠 Output from Gemini 27B (Markdown Report)

```md
## Patient Summary

- **Symptoms**: Dizziness, chest pain
- **Onset**: 2 days ago
- **Additional Notes**: Patient uploaded recent blood test.

## Suggested Next Steps

- Consult cardiologist
- Perform ECG and blood panel
```

---

## ✅ 5. Markdown-to-PDF Conversion

No specific JSON needed unless you expose an endpoint like:

```json
{
  "session_id": "abc123",
  "markdown": "<Gemini-generated-markdown>"
}
```

And backend responds with:

```json
{
  "pdf_url": "https://firebase.storage/bucket/abc123/report.pdf"
}
```

---

## ✅ 6. Final Report Metadata in Firestore for the Doctor Interface

```json
{
  "session_id": "abc123",
  "user_id": "user001",
  "created_at": "2025-07-04T09:00:00Z",
  "markdown": "...",
  "pdf_url": "https://firebase.storage/bucket/abc123/report.pdf"
}
```

---

## ✅ Summary Table of JSON Interfaces

| Interface                 | Direction | Type     | Purpose                     |
| ------------------------- | --------- | -------- | --------------------------- |
| `/call/:id` WebSocket     | ⬅️➡️      | JSON     | Live text/audio interaction |
| Redis                     | internal  | JSON     | Store convo session memory  |
| `/upload`                 | ➡️        | JSON     | Upload documents            |
| `/upload` Response        | ⬅️        | JSON     | Returns S3 URL              |
| Gemini Input              | ➡️        | JSON     | Full transcript + doc refs  |
| Gemini Output             | ⬅️        | Markdown | Health summary report       |
| Final PDF Endpoint (opt.) | ➡️        | JSON     | Convert MD to PDF           |
| Firestore                 | internal  | JSON     | Archive session + report    |
