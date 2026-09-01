Step 1: Install Required **NPM** Dependencies Run this command in your repository root to install the packages needed for **EXIF** image parsing, Firebase integration, and helper utilities:

Bash npm install exifr firebase-admin Step 2: Configure Environment Variables (.env.local) Create a .env.local file in your root directory to store your **API** keys and credentials securely:

Code snippet GROQ_API_KEY=your_groq_api_key_here GOOGLE_AI_API_KEY=your_google_ai_api_key_here FIREBASE_PROJECT_ID=your_project_id FIREBASE_CLIENT_EMAIL=your_client_email FIREBASE_PRIVATE_KEY=*-----**BEGIN** **PRIVATE** **KEY**-----\n...* Step 3: What Needs to Be in Your Codebase (The AI Checklist) To complete the AI pipeline, your file structure should look like this, ensuring all components plug cleanly into your Next.js **API** route:

Plaintext your-repo/ ├── app/ │   └── api/ │       └── ai/ │           └── analyze-complaint/ │               └── route.js       # The main endpoint (from our master plan) ├── lib/ │   ├── groq-client.js             # Wrapper for Groq **LLM** (Structured **JSON**) │   ├── google-ai-client.js        # Wrapper for Google AI (Embeddings) │   ├── firebase-admin.js          # Firebase Firestore instance setup │   ├── exif-parser.js             # Extracts **GPS** coordinates from uploaded image **EXIF** │   └── vector-math.js             # Cosine similarity calculation for duplicate detection └── .env.local Quick Reference Code for Missing Helper Files: lib/exif-parser.js

JavaScript
import exifr from 'exifr';
export async function extractImageGps(buffer) {
    try {
    const gps = await exifr.gps(buffer);
    if (gps && gps.latitude && gps.longitude) {
    return { latitude: gps.latitude, longitude: gps.longitude };
    }
    } catch (e) { console.log(*No **EXIF** data found*); }
    return null;
}
lib/vector-math.js

JavaScript
export function cosineSimilarity(vecA, vecB) {
    let dotProduct = 0, normA = 0, normB = 0;
    for (let i = 0; i < vecA.length; i++) {
    dotProduct += vecA[i] * vecB[i];
    normA += vecA[i] * vecA[i];
    normB += vecB[i] * vecB[i];
    }
    return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
}
Step 4: Connecting the Frontend UI
To complete the loop, your Next.js frontend page (e.g., app/submit/page.jsx) must send a FormData object to your newly created **API** route:

JavaScript
async function handleSubmit(e) {
    e.preventDefault();
    const formData = new FormData(e.target);

    const response = await fetch('/api/ai/analyze-complaint', {
    method: '**POST**',
    body: formData
    });

    const result = await response.json();
    if (result.success) {
    console.log(*Routed to:*, result.data.department);
    console.log(*Is Duplicate:*, result.data.duplicateDetection.isDuplicate);
    }
}
Step 5: Local Testing & Verification
Run npm run dev to start your Next.js local server.

Use Postman or your Frontend UI to submit a test grievance with colloquial text (e.g., *Road bara par khadda hai*).

Check your Firebase Firestore console to ensure:

A new document is successfully written to the complaints collection with the normalized text, sub-category, and vector embedding.

A corresponding cache document is created in the ai_insights collection for your geospatial dashboard map.
