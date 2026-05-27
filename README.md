# Gym Review REST API

A complete, tested, and secure REST API for gym reviews built with Express.js, Firebase authentication, React frontend, and comprehensive test coverage.

**📦 Deployed**: 
- Frontend: `[Add your Vercel URL here after deployment]`
- Backend: `[Add your Render URL here after deployment]`

## Table of Contents

- [Setup](#setup)
- [Testing](#testing)
- [Docker & Containerization](#docker--containerization)
- [Deployment](#deployment-to-cloud-platforms)
- [Authentication](#authentication)
- [Security Decisions](#security-decisions)
- [Reflections](#reflections)
- [Reflections](#reflections)

## Setup

### Prerequisites

- Node.js 22.x or higher
- npm or yarn
- A Firebase project with authentication enabled

### Clone the Repository

```bash
git clone <your-repo-url>
cd Building-and-securing-tested-REST-API
```

### Install Dependencies

#### Backend

```bash
cd backend
npm install
```

#### Frontend

```bash
cd frontend
npm install
```

### Configure Environment Variables

Copy the `.env.example` files and update them with your actual values:

#### Backend (`backend/.env`)

```bash
cp .env.example .env
```

Then edit `backend/.env`:

```env
FIREBASE_PROJECT_ID=your_project_id
FIREBASE_PRIVATE_KEY=your_private_key
FIREBASE_CLIENT_EMAIL=your_client_email
PORT=5001
NODE_ENV=development
FRONTEND_URL=http://localhost:5173
```

#### Frontend (`frontend/.env`)

```bash
cp .env.example .env
```

Then edit `frontend/.env`:

```env
VITE_FIREBASE_API_KEY=your_api_key
VITE_FIREBASE_AUTH_DOMAIN=your_auth_domain
VITE_FIREBASE_PROJECT_ID=your_project_id
VITE_FIREBASE_STORAGE_BUCKET=your_storage_bucket
VITE_FIREBASE_MESSAGING_SENDER_ID=your_sender_id
VITE_FIREBASE_APP_ID=your_app_id
VITE_API_URL=http://localhost:5001
```

### Run the Project Locally

#### Terminal 1: Start the Backend

```bash
cd backend
npm run dev
```

The server will run on `http://localhost:5001`

#### Terminal 2: Start the Frontend

```bash
cd frontend
npm run dev
```

The frontend will run on `http://localhost:5173`

## Testing

### Backend Test Structure

The backend uses **Vitest** with **Supertest** for integration testing and includes mocking of the Firebase Admin SDK:

#### Mock Files (`backend/__mocks__/`)

The `__mocks__/` directory contains mock implementations of external dependencies:

- **`firebase-admin.js`**: Mocks the Firebase Admin SDK to avoid requiring real Firebase credentials during testing
  - Provides mock implementations of `auth()`, `initializeApp()`, and credential methods
  - Allows tests to mock token verification without making real Firebase API calls
  - Tests can control token validation behavior using `vi.fn()` and `.mockResolvedValue()` / `.mockRejectedValue()`

**Why Mocking Firebase?**
- Real Firebase requires valid credentials and makes external API calls (slow tests)
- Mocking allows tests to run in isolation and control all scenarios (valid tokens, expired tokens, invalid tokens)
- Makes tests deterministic and doesn't rely on external service availability

### Run Tests Locally

#### Backend Integration Tests

```bash
cd backend
npm test
```

#### Frontend Unit Tests

```bash
cd frontend
npm test
```


## GitHub Actions Pipeline

The project includes a GitHub Actions workflow (`.github/workflows/test.yml`) that:

1. **Triggers on every push and pull request to main** -  all code changes are tested
2. **Installs dependencies** for both backend and frontend
3. **Runs all tests** - Both integration and unit tests
4. **Uses GitHub Secrets** for sensitive Firebase credentials

### Setting Up GitHub Secrets

Add the following secrets to your GitHub repository:

```
FIREBASE_PROJECT_ID
FIREBASE_PRIVATE_KEY
FIREBASE_CLIENT_EMAIL
VITE_FIREBASE_API_KEY
VITE_FIREBASE_AUTH_DOMAIN
VITE_FIREBASE_PROJECT_ID
VITE_FIREBASE_STORAGE_BUCKET
VITE_FIREBASE_MESSAGING_SENDER_ID
VITE_FIREBASE_APP_ID
```

## Authentication

### Provider Choice: Firebase

We chose **Firebase Authentication** because:

1. **Easy Integration**: Firebase Admin SDK provides straightforward token verification
2. **Token-Based**: Naturally supports API authentication via bearer tokens
3. **Security**: Managed by Google, handles hashing and best practices
4. **Scalability**: No need to manage user databases ourselves
5. **Cross-Platform**: Firebase works seamlessly across web, mobile, and backend

### How Authentication is Implemented

#### Frontend (`src/context/AuthContext.jsx`)

- **Firebase Initialization**: Sets up Firebase with configuration from environment variables
- **Auth Context**: Provides user state and authentication methods globally
- **Login**: Uses `signInWithPopup` with Google Provider
- **Token Management**: Automatically gets ID token after login
- **Logout**: Clears user state

#### Backend (`src/middleware/auth.js`)

- **Middleware**: `verifyToken` middleware checks Bearer tokens in `Authorization` headers
- **Token Verification**: Uses Firebase Admin SDK to verify tokens server-side
- **Protected Routes**: All POST routes and `/profile` require valid tokens
- **Error Handling**: Returns 401 Unauthorized for missing or invalid tokens

#### Routes

| Method | Route | Access | Returns |
|--------|-------|--------|---------|
| GET | /gyms | Public | 200 + gym list |
| GET | /gyms/:id | Public | 200 + gym details or 404 |
| POST | /gyms | Protected | 201 + new gym or 401/400 |
| POST | /gyms/:id/reviews | Protected | 201 + review or 401/404/400 |
| GET | /profile | Protected | 200 + user profile or 401 |

## Security Decisions

### 1. **No Secrets in Repository**

**What**: All sensitive values (Firebase credentials, API keys) are stored in `.env` files which are gitignored.

**Why**: If secrets are committed, anyone with repository access can compromise your Firebase project. Using environment variables ensures secrets are only available on secure servers and CI/CD pipelines.

**How**: 
- `.env` is in `.gitignore`
- `.env.example` documents required variables without values
- GitHub Actions uses GitHub Secrets for sensitive values

### 2. **Protected Routes Return 401**

**What**: All POST endpoints and the `/profile` endpoint require valid Firebase tokens. Requests without tokens return `401 Unauthorized`.

**Why**: This prevents unauthorized access to sensitive operations like creating gyms or viewing user profiles. Tests verify this behavior explicitly.

**How**:
- `verifyToken` middleware on all protected routes
- Firebase Admin SDK validates bearer tokens
- Invalid/missing tokens immediately return 401

### 3. **CORS Restricted to Specific Origin**

**What**: CORS is configured to only accept requests from `http://localhost:5173` (frontend origin), not `*`.

**Why**: A wildcard CORS policy allows requests from ANY origin, enabling attackers to make requests on behalf of users. Restricting to a specific origin prevents cross-origin attacks.

**How**:
```javascript
const corsOptions = {
  origin: process.env.FRONTEND_URL || 'http://localhost:5173',
  credentials: true
};
app.use(cors(corsOptions));
```

### 4. **Credentials in All Authenticated Requests**

**What**: Frontend includes `credentials: 'include'` in all authenticated API calls and sets headers with bearer tokens.

**Why**: This ensures cookies (if used) are sent with requests, and the server knows these requests are from authenticated users. It prevents token-based attacks.

**How**:
```javascript
fetch(`${API_URL}/gyms`, {
  headers: { 'Authorization': `Bearer ${token}` },
  credentials: 'include'
})
```

### 5. **Tokens Are NOT Stored in localStorage**

**What**: Firebase authentication tokens are stored only in React state and are retrieved directly from Firebase Authentication when needed.

**Why**: Storing tokens in localStorage increases the risk of token theft through XSS (Cross-Site Scripting) attacks because JavaScript running in the browser can access localStorage values. Keeping tokens only in memory reduces this risk.

**How**:
- Tokens are managed using React state
- Tokens are refreshed directly through Firebase Authentication
- No authentication tokens are persisted in browser localStorage

### 6. **Bearer Token in Authorization Header**

**What**: Tokens are sent in the `Authorization: Bearer <token>` header, not in URL or body.

**Why**: Headers are not logged by default in servers/proxies, and URL parameters can be cached or logged. The Authorization header is the standard secure location for credentials.

**How**:
```javascript
const token = authHeader.substring(7); // Extract token from "Bearer {token}"
const decodedToken = await admin.auth().verifyIdToken(token);
```

### 7. **Input Validation**

**What**: All endpoints validate required fields before processing.

**Why**: Prevents invalid or malicious data from being stored.

**How**:
```javascript
if (!name || !location) {
  return res.status(400).json({ error: 'Name and location are required' });
}
```

### 8. **In-Memory Database for Testing**

**What**: The API uses an in-memory array for storing gyms and reviews.

**Why**: For this assignment, an in-memory database is sufficient and keeps focus on testing and security, not database design.

## Reflections

### Implementation Choices

1. **Express.js Backend**: Simple, well-documented framework perfect for RESTful APIs
2. **React + Context API**: Minimal frontend with no complex state management needed
3. **Firebase**: Chose token-based auth for better API security patterns
4. **In-Memory Database**: Kept focus on testing and security
5. **Vitest**: Modern test runner with excellent mocking and React support
6. **Docker**: Multi-stage builds minimize image size and improve security
7. **Vercel + Render**: Industry-standard platforms with excellent DX and free tiers

### Challenges & Solutions

**Challenge 1: Firebase Admin SDK Initialization in Tests**
- **Problem**: Firebase Admin requires credentials, but tests need to mock authentication
- **Solution**: Mocked `firebase-admin` entirely using `vi.mock()`, allowing tests to control verification

**Challenge 2: CORS and Credentials**
- **Problem**: Credentials require specific CORS configuration, not just a wildcard
- **Solution**: Set `FRONTEND_URL` as environment variable and use in `corsOptions`

**Challenge 3: Frontend-Backend Token Flow**
- **Problem**: Needed to ensure tokens are correctly extracted from Firebase and sent to backend
- **Solution**: Created `AuthContext` to manage token lifecycle and add tokens to all requests

**Challenge 4: Docker Environment Variables**
- **Problem**: Secrets cannot be built into images, but apps need configuration
- **Solution**: Use Docker's `-e` flag and cloud platform environment variable management

**Challenge 5: Production vs Development URLs**
- **Problem**: Frontend and backend URLs change between local, staging, and production
- **Solution**: Use environment variables for all URLs (API_URL, FRONTEND_URL)

### What We'd Do Differently

1. **Database**: Use MongoDB with encryption at rest for production
2. **Rate Limiting**: Add middleware to prevent brute force attacks
3. **Logging**: Implement structured logging to audit authentication
4. **Refresh Tokens**: Implement refresh token rotation for longer sessions

### Why Vercel + Render?

- **Vercel**: Optimized for React, automatic git deployments, edge locations, generous free tier
- **Render**: Excellent Docker support, managed PostgreSQL, reliable uptime, free tier includes HTTPS
---

## Docker & Containerization

### Build and Run Locally with Docker

#### Build Images
```bash
# Build backend image
docker build -t gym-api-backend:latest ./backend

# Build frontend image
docker build -t gym-api-frontend:latest ./frontend
```

#### Run with Docker Compose
```bash
# Create .env file from examples
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env

# Edit .env files with your credentials
# nano backend/.env
# nano frontend/.env

# Start both services
docker-compose up -d

# View logs
docker-compose logs -f

# Stop services
docker-compose down
```

### Key Docker Decisions

1. **Multi-Stage Builds**: Reduces image size by removing build dependencies (Node.js, npm packages)
2. **Alpine Linux**: Smaller base images (node:22-alpine) for faster deployments
3. **Health Checks**: Ensure containers are ready before routing traffic
4. **No Hardcoded Secrets**: Environment variables passed at runtime, not built into image

---

## Deployment to Cloud Platforms

### Recommended Stack: Vercel (Frontend) + Render (Backend)

This is the recommended setup for this project. Both platforms offer generous free tiers and excellent performance.

#### Backend Deployment on Render

1. **Create Render Account**: Go to [https://render.com](https://render.com) and sign up

2. **Create New Web Service**:
   - Click "New +"  → "Web Service"
   - Connect your GitHub repository
   - Select the repository

3. **Configure Service**:
   - **Name**: `gym-api` (or your choice)
   - **Environment**: `Docker`
   - **Build Command**: (Render auto-detects from Dockerfile)
   - **Start Command**: (Render auto-detects)

4. **Add Environment Variables**:
   - Click "Environment" in the dashboard
   - Add all variables from `backend/.env.example`:
     ```
     FIREBASE_PROJECT_ID=your_project_id
     FIREBASE_PRIVATE_KEY=your_private_key
     FIREBASE_CLIENT_EMAIL=your_client_email
     FRONTEND_URL=https://your-frontend.vercel.app
     NODE_ENV=production
     PORT=10000
     ```
   - **Important**: Set `FRONTEND_URL` to your Vercel frontend URL

5. **Deploy**:
   - Push to GitHub main branch
   - Render auto-deploys on push
   - Backend URL will be: `https://gym-api-{random}.onrender.com` (shown on dashboard)

#### Frontend Deployment on Vercel

1. **Create Vercel Account**: Go to [https://vercel.com](https://vercel.com) and sign up

2. **Import Project**:
   - Click "Add New..." → "Project"
   - Import your GitHub repository

3. **Configure Build Settings**:
   - **Framework**: React
   - **Build Command**: `cd frontend && npm run build`
   - **Output Directory**: `frontend/dist`
   - **Install Command**: `cd frontend && npm install`

4. **Add Environment Variables** (in project settings):
   ```
   VITE_FIREBASE_API_KEY=your_api_key
   VITE_FIREBASE_AUTH_DOMAIN=your_auth_domain
   VITE_FIREBASE_PROJECT_ID=your_project_id
   VITE_FIREBASE_STORAGE_BUCKET=your_storage_bucket
   VITE_FIREBASE_MESSAGING_SENDER_ID=your_sender_id
   VITE_FIREBASE_APP_ID=your_app_id
   VITE_API_URL=https://gym-api-{random}.onrender.com
   ```
   - **Important**: Update `VITE_API_URL` with your actual Render backend URL

5. **Deploy**:
   - Click "Deploy"
   - Vercel builds and deploys automatically
   - Frontend URL: `https://your-project.vercel.app`

#### Update Firebase Configuration for Production

1. **Go to Firebase Console** → Your Project → Settings → Authorized Domains
2. **Add deployed URLs**:
   - `your-project.vercel.app` (frontend)
3. **Update Render deployment**:
   - Backend must know the frontend URL for CORS
   - Re-deploy Render after updating `FRONTEND_URL` env var

### Testing the Deployed Application

```bash
# Test backend API
curl https://gym-api-{random}.onrender.com/gyms

# Expected response:
# [{"id":"1","name":"FitnessFirst Downtown",...}, ...]
```

Visit `https://your-project.vercel.app` in browser:
- Should see login button
- Click to login with Google
- After login, should see your profile and gym list
- Should be able to add reviews to gyms

## Production Deployment Checklist

Before deploying to production, verify all of these:

- [ ] **1. No Secrets Committed**: Run `git log -p --all -S"FIREBASE_PRIVATE_KEY"` to verify no secrets in history
- [ ] **2. CORS Restricted**: `FRONTEND_URL` in backend points to deployed frontend, not `*`
- [ ] **3. Tokens Secure**: Tokens sent in `Authorization: Bearer` header, NOT in URL or localStorage
- [ ] **4. withCredentials Set**: Frontend requests include credentials flag
- [ ] **5. Docker Image Clean**: No `.env` files, `.git`, or `node_modules` in Docker image
- [ ] **6. Backend Uses HTTPS**: Deployed backend is served over HTTPS (most platforms auto-enable)
- [ ] **7. Auth Redirects**: Firebase redirects point to deployed frontend URL, not localhost
- [ ] **8. Environment Variables**: All required vars set in cloud platform (not in code)
- [ ] **9. Tests Pass**: Run `npm test` in both frontend and backend - all tests pass
- [ ] **10. CI Pipeline Works**: GitHub Actions runs successfully on push

---

---

## Project Structure

```
.
├── .github/workflows/test.yml
├── backend/
│   ├── __mocks__/
│   │   └── firebase-admin.js
│   ├── src/
│   │   ├── __tests__/integration.test.js
│   │   ├── middleware/auth.js
│   │   ├── routes/gyms.js
│   │   ├── routes/profile.js
│   │   ├── app.js
│   │   └── db.js
│   ├── index.js
│   ├── .env.example
│   └── package.json
├── frontend/
│   ├── src/
│   │   ├── __tests__/components.test.jsx
│   │   ├── components/
│   │   ├── context/AuthContext.jsx
│   │   ├── styles/
│   │   ├── App.jsx
│   │   └── main.jsx
│   ├── .env.example
│   └── package.json
└── README.md
```

### Screenshots 

<img width="1458" height="797" alt="github actions" src="https://github.com/user-attachments/assets/df4b38ec-4294-4a3b-a6da-dd5a5d50defc" />


<img width="1206" height="752" alt="frontend test" src="https://github.com/user-attachments/assets/d0c0fd06-88e4-4437-9ed5-79ce9b620b38" />



<img width="1224" height="779" alt="backend test" src="https://github.com/user-attachments/assets/a632008b-cbd1-4221-815f-949cbf79d871" />
