# SIH S02: AI-Powered Grievance Portal - Master Integration Plan

## 1. Architecture & Flow Overview
1. **Frontend (Next.js):** User selects a department (optional hint), writes a complaint (supports colloquial/Hinglish text like *"Road number bara par khadda hai"*), uploads an image, and shares location.
2. **Next.js API Route (`/api/ai/analyze`):** 
   - Extracts EXIF GPS from the image buffer (if available).
   - Normalizes/Translates colloquial text into a professional description (*"Pothole spotted at Road 12"*).
   - Classifies Department + Sub-category using hardcoded taxonomy.
   - Generates Google AI embeddings for duplicate detection.
3. **Firebase Firestore:** Persists the structured record and updates cache collections.
4. **Response:** Returns a strict JSON envelope to the frontend for immediate UI rendering.

---

## 2. Taxonomy & Department Mapping (Hardcoded Samples)
To guide the AI model accurately, use this strict mapping inside your classification prompt:

```json
{
  "Roads & Infrastructure": ["Potholes", "Broken Footpath", "Road Cave-in", "Damaged Bridge"],
  "Water Supply & Drainage": ["Pipeline Leakage", "Contaminated Water", "Overflowing Drain", "No Water Supply"],
  "Electricity & Streetlights": ["Streetlight Not Working", "Sparking Wire", "Open Transformer", "Power Outage"],
  "Waste Management & Sanitation": ["Uncleaned Garbage Dump", "Dead Animal Removal", "Blocked Sewer", "Public Toilet Issue"]
}
```
## 3. Firebase Firestore Schema Mapping
Ensure your AI pipeline outputs a JSON structure that maps directly to this Firestore document schema:

Collection: complaints

Document ID: Auto-generated (e.g., cmp_98af21...)

Fields:

description (String): Original user text.

normalizedText (String): Cleaned/translated text (e.g., "Potholes spotted at Road 12").

department (String): e.g., "Roads & Infrastructure".

incidentSubCategory (String): e.g., "Potholes".

priority (String): "High" | "Medium" | "Low".

latitude (Number): Geotagged coordinate.

longitude (Number): Geotagged coordinate.

embedding (Array of Numbers): 384-dim vector for semantic matching.

isDuplicate (Boolean): True if matches an active nearby complaint.

parentComplaintId (String | null): ID of the original complaint if duplicate.

status (String): "Active" | "Marked as Duplicate" | "Resolved".

createdAt (Timestamp)

## 4. Next.js API Route Implementation (app/api/ai/analyze-complaint/route.js)
```
import { NextResponse } from 'next/server';
import exifr from 'exifr';
import { invokeStructuredJSON } from '@/lib/groq-client';
import { generateEmbedding } from '@/lib/google-ai-client';
import { db } from '@/lib/firebase-admin';

// Helper for cosine similarity
function cosineSimilarity(vecA, vecB) {
  let dotProduct = 0, normA = 0, normB = 0;
  for (let i = 0; i < vecA.length; i++) {
    dotProduct += vecA[i] * vecB[i];
    normA += vecA[i] * vecA[i];
    normB += vecB[i] * vecB[i];
  }
  return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
}

export async function POST(req) {
  try {
    const formData = await req.formData();
    const rawDescription = formData.get('description');
    const userDeptHint = formData.get('departmentHint') || '';
    const deviceLat = parseFloat(formData.get('latitude'));
    const deviceLng = parseFloat(formData.get('longitude'));
    const imageFile = formData.get('image');

    // 1. Auto-Geotagging via Image EXIF
    let finalLat = deviceLat;
    let finalLng = deviceLng;
    if (imageFile) {
      const buffer = Buffer.from(await imageFile.arrayBuffer());
      const gps = await exifr.gps(buffer).catch(() => null);
      if (gps && gps.latitude && gps.longitude) {
        finalLat = gps.latitude;
        finalLng = gps.longitude;
      }
    }

    // 2. AI Classification, Normalization & Sub-categorization (Groq)
    const prompt = `
      You are a municipal AI assistant for citizen grievances. 
      Analyze this user complaint: "${rawDescription}".
      User Department Hint (optional): "${userDeptHint}".

      Taxonomy allowed:
      - Roads & Infrastructure: ["Potholes", "Broken Footpath", "Road Cave-in"]
      - Water Supply & Drainage: ["Pipeline Leakage", "Contaminated Water", "Overflowing Drain"]
      - Electricity & Streetlights: ["Streetlight Not Working", "Sparking Wire", "Open Transformer"]
      - Waste Management & Sanitation: ["Uncleaned Garbage Dump", "Dead Animal Removal", "Blocked Sewer"]

      Return a strict JSON object with:
      1. "normalizedText": Translate/normalize colloquial text (e.g., Hinglish "Road bara par khadda hai") into clean, standard English (e.g., "Potholes spotted at Road 12").
      2. "department": Must be one of the 4 categories above.
      3. "incidentSubCategory": Must be one of the sub-categories matching the department.
      4. "priority": "High", "Medium", or "Low" (Mark "High" if safety hazard like sparking wires or major pipe bursts).
      5. "summary": A brief 1-sentence description.
    `;

    const aiResult = await invokeStructuredJSON(prompt);

    // 3. Generate Embeddings for Duplicate Detection (Google AI)
    const currentEmbedding = await generateEmbedding(aiResult.normalizedText);

    // 4. Vector & Geospatial Cosine Search against Active Complaints in Firebase
    const snapshot = await db.collection('complaints')
      .where('department', '==', aiResult.department)
      .where('status', '==', 'Active')
      .limit(30)
      .get();

    let isDuplicate = false;
    let parentComplaintId = null;
    let maxSimilarity = 0;

    snapshot.forEach(doc => {
      const past = doc.data();
      if (past.embedding) {
        const similarity = cosineSimilarity(currentEmbedding, past.embedding);
        const latDiff = Math.abs(finalLat - past.latitude);
        const lngDiff = Math.abs(finalLng - past.longitude);

        // If similarity > 0.85 and within ~100 meters geographic proximity
        if (similarity > 0.85 && latDiff < 0.001 && lngDiff < 0.001) {
          if (similarity > maxSimilarity) {
            maxSimilarity = similarity;
            isDuplicate = true;
            parentComplaintId = doc.id;
          }
        }
      }
    });

    // 5. Firebase Batch Write
    const batch = db.batch();
    const newRef = db.collection('complaints').doc();

    const complaintDocData = {
      description: rawDescription,
      normalizedText: aiResult.normalizedText,
      department: aiResult.department,
      incidentSubCategory: aiResult.incidentSubCategory,
      priority: aiResult.priority,
      summary: aiResult.summary,
      latitude: finalLat,
      longitude: finalLng,
      embedding: currentEmbedding,
      isDuplicate,
      parentComplaintId,
      status: isDuplicate ? 'Marked as Duplicate' : 'Active',
      createdAt: new Date().toISOString()
    };

    batch.set(newRef, complaintDocData);

    // Cache entry for GIS Heatmap dashboard
    const insightRef = db.collection('ai_insights').doc(newRef.id);
    batch.set(insightRef, {
      department: aiResult.department,
      subCategory: aiResult.incidentSubCategory,
      lat: finalLat,
      lng: finalLng,
      priority: aiResult.priority,
      isDuplicate
    });

    await batch.commit();

    // 6. Return Structured JSON Envelope
    return NextResponse.json({
      success: true,
      error: null,
      data: {
        complaintId: newRef.id,
        ...aiResult,
        geospatial: { latitude: finalLat, longitude: finalLng },
        duplicateDetection: { isDuplicate, parentComplaintId, similarityScore: maxSimilarity }
      }
    });

  } catch (error) {
    console.error("AI Pipeline Error:", error);
    return NextResponse.json({ success: false, error: error.message, data: null }, { status: 500 });
  }
}
```
## 5. Frontend Integration Snippet (React / Next.js)
```
async function handleSubmit(event) {
  event.preventDefault();
  const formData = new FormData(event.target);

  const res = await fetch('/api/ai/analyze-complaint', {
    method: 'POST',
    body: formData
  });

  const result = await res.json();
  if (result.success) {
    alert(`Complaint filed successfully under: ${result.data.department} -> ${result.data.incidentSubCategory}`);
  }
}
```
