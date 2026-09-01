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