
# 🌱 AgriFert AI — Enhanced (Fixed v4)

## ⚠️ What Was Actually Wrong (Previous "v3 Fix")

The old README in this project claimed the chatbot and voice assistant were fixed by making them call the Anthropic API directly from the browser.

That caused the following problems:

- `FarmerChatbot.jsx` called the Anthropic API directly from React without an API key. Browser applications cannot safely store secret API keys. Requests returned 401 errors and silently fell back to keyword matching.
- `VoiceAssistant.jsx` did not call any AI or backend endpoint. It used keyword matching regardless of language or phrasing.

The app appeared AI-powered but was primarily rule-based. A working server-side Claude integration already existed in `server/server.js` but was not being used by the chat and voice interfaces.

## ✅ What I Fixed

1. **Chatbot backend integration**
   - Uses `POST /api/chat/message`.
   - The backend holds `ANTHROPIC_API_KEY` securely.
   - Sends free-form questions to Claude.
   - Stores conversational history in MongoDB for follow-up questions.
   - Uses keyword matching only as a last-resort fallback.

2. **Voice Assistant backend integration**
   - Uses `POST /api/voice/process`.
   - Uses the browser Web Speech API for speech transcription.
   - Sends the transcript to the backend.
   - Claude interprets the question and replies in the same language.
   - SpeechSynthesis reads the response aloud.
   - The voice endpoint is public because the assistant is available on the public Home page and does not access per-user data.

3. **Claude model configuration**
   - Updated the server-side Claude model configuration.
   - Verify that the configured model ID is currently supported by your Anthropic account before deployment.

4. **Multer upgrade**
   - Upgraded Multer from the deprecated/vulnerable 1.x line to 2.x.
   - Existing `memoryStorage()`, `.single()`, and `fileFilter` APIs are retained.

5. **Verification**
   - Checked backend syntax.
   - Checked JWT and bcrypt authentication logic.
   - Tested fertilizer and disease rule-engine behavior with different inputs.

## 🚀 Running the App

### Backend

```bash
cd server
npm install
```

Edit the `.env` file and configure a valid MongoDB connection string.

Optionally configure your Anthropic API key for AI-powered chat, voice, and vision.

```bash
npm run dev
```

Use `npm start` if you want to start the server without nodemon.

### Frontend

Open another terminal:

```bash
cd client
npm install
npm start
```

Open the application:

http://localhost:3000

The frontend communicates with the backend through `REACT_APP_API_URL`, which defaults to:

http://localhost:5000/api

## 🔑 Environment Variables

### server/.env

| Variable | Description |
|---|---|
| PORT | Backend port (default: 5000) |
| MONGO_URI | MongoDB connection string |
| JWT_SECRET | Secret used to sign authentication tokens |
| JWT_EXPIRE | Token lifetime, for example `30d` |
| ANTHROPIC_API_KEY | Enables Claude-powered chat, voice, and vision |
| NODE_ENV | Application environment |
| CLIENT_URL | Allowed frontend origin for CORS |

### client/.env

| Variable | Description |
|---|---|
| REACT_APP_API_URL | Backend API URL |
| REACT_APP_APP_NAME | Application name |

**Security:** Never commit real API keys, passwords, database credentials, or `.env` files to GitHub. Use placeholders in `.env.example`.

## 📋 How Each Feature Behaves

| Feature | Backend and MongoDB | Anthropic API Key |
|---|---|---|
| Login / Register | Stores real accounts in MongoDB with JWT authentication | Not required |
| Chatbot | Stores chat history per session | Enables Claude natural-language answers |
| Chatbot fallback | Uses rule-based replies if AI is unavailable | Not required |
| Voice Assistant | Processes voice requests through the backend | Enables Claude natural-language answers |
| Disease Detection | Saves detection history | Enables Claude Vision image analysis |
| AI Recommendation | Saves recommendation history | Adds a Claude-generated explanation |
| Dashboard | Aggregates saved user history | Not required |

## 🧪 What Was Verified

The following checks were performed in the development environment:

- `node -c server.js` — no syntax errors.
- Backend routes loaded without runtime errors.
- Invalid MongoDB connection handling was checked.
- bcrypt hash and compare operations were tested.
- JWT sign and verify operations were tested.
- Fertilizer and disease rule engines were tested with varied inputs.
- Client production build completed without errors.

A live MongoDB end-to-end test could not be completed in the sandbox because network access to MongoDB download and Atlas servers was unavailable.

Before production deployment, test the complete workflow:

1. Register a new account.
2. Log in.
3. Generate a fertilizer recommendation.
4. Test the chatbot.
5. Test the voice assistant.
6. Upload an image for disease detection.
7. Log out.

Use a valid MongoDB URI and your own Anthropic API key.

## ⚠️ Remaining Limitations

### 1. Disease Detection

Disease detection is not based on a separately trained CNN image-classification model.

- The project does not contain a PlantVillage-trained CNN or a separate Python/Flask ML service.
- When an image is uploaded and the Anthropic API key is configured, Claude Vision analyzes that image.
- Without an image or API key, the system uses a rule-based disease table and soil-related heuristics.
- Genuine trained image classification would require training and hosting a CNN model and creating a dedicated prediction endpoint.

### 2. Fertilizer Recommendation

Fertilizer recommendation is rule-based rather than a trained machine-learning model.

- It uses N/P/K values, pH, and soil type.
- Recommendations vary according to the supplied inputs.
- Claude can generate explanations when the API key is configured.

### 3. Demo / Offline Authentication

Browser-local demo authentication is intended only for UI testing when the backend is unavailable.

Other features require the backend and/or database and will not work normally when those services are unavailable.

### 4. API Rate Limiting

The project uses generic `express-rate-limit` protection.

For public deployment, add per-user quotas for Claude API requests to prevent excessive API usage and unexpected costs.

## 🛠️ Technology Stack

- **Frontend:** React.js
- **Backend:** Node.js and Express.js
- **Database:** MongoDB
- **Authentication:** JWT and bcryptjs
- **AI Integration:** Anthropic Claude API
- **Voice:** Web Speech API and SpeechSynthesis
- **File Upload:** Multer
- **Environment Configuration:** dotenv

---

**AgriFert AI — Smart Fertilizer Recommendations and AI-Based Crop Assistance.**
