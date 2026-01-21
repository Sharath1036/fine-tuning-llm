# Fine-Tuning Gemini 2.5 Flash-Lite on Vertex AI for Ingredient Extraction

This guide walks you through setting up an IAM user, preparing a Google Cloud project, uploading a dataset with images, and fine-tuning the Gemini 2.5 Flash-Lite model on Vertex AI to extract product information and flagged ingredients from images based on dietary preferences.

## Prerequisites

- A Google account with access to Google Cloud.
- Basic familiarity with the Google Cloud Console and storage concepts.
- Local machine with a browser and text editor (e.g., VS Code).

## Step 1: Create an IAM User

1. **Sign In to Google Cloud**  
   Go to https://console.cloud.google.com and sign in with your Google account.

2. **Create a Project**  
   Click the project dropdown → **New Project**  
   Name it `ingredicheck-426912` (or your preferred name) → **Create**

3. **Enable Billing**  
   Go to **Billing** in the sidebar → create/link a billing account (required for Vertex AI).

4. **Set Up IAM User**  
   Go to **IAM & Admin > IAM**  
   Click **Grant Access**, add your email, assign:  
   - `roles/aiplatform.user`  
   - `roles/storage.objectAdmin`  
   Click **Save**

5. **Initialize Google Cloud SDK** (optional but recommended)  
   Install from https://cloud.google.com/sdk  
   Run `gcloud init` → select project `ingredicheck-426912` → authenticate

## Step 2: Set Up Google Cloud Storage

1. **Create a Bucket**  
   Go to **Cloud Storage > Buckets > Create Bucket**  
   Name: `ingredicheck-finetune-426912` (globally unique)  
   Location: `us-central1` (recommended for Gemini tuning)  
   Storage class: `Standard`  
   Click **Create**

2. **Verify Access**  
   Make sure you can see the bucket in the console.  
   (CLI check: `gsutil ls gs://ingredicheck-finetune-426912/` — run as Administrator if permission error occurs)

## Step 3: Prepare the Dataset

### Option A – Single training file (most common for small experiments)

1. Create `extractor.vertex.jsonl` with the correct Gemini tuning format:  
   ```json
   {"messages": [
     {"role": "user",   "content": "prompt text or multimodal content"},
     {"role": "model",  "content": "expected completion"}
   ]}
   ```

   or (newer multimodal style)

   ```json
   {"contents": [
     {"role": "user",   "parts": [{"text": "..."}, {"fileData": {...}}]},
     {"role": "model",  "parts": [{"text": "..."}]}
   ]}
   ```

2. Collect images and upload them to `gs://ingredicheck-finetune-426912/images/`

3. Update `fileUri` in the JSONL to point to Cloud Storage:  
   `"fileUri": "gs://ingredicheck-finetune-426912/images/image1.jpg"`

4. Upload the JSONL file to the bucket root (or any path you like).

→ Vertex AI will automatically create an internal validation split (~10–20%).

### Option B – You already have separate train and validation sets (recommended when you want to control evaluation)

1. Prepare two JSONL files with **exactly the same format**:

   - `train.jsonl` → only training examples  
   - `valid.jsonl` or `val.jsonl` → validation / hold-out examples

   Example structure (both files must use this format):

   ```json
   {"messages": [
     {"role": "user",   "parts": [{"text": "Dietary preferences: avoid sugar"}, {"fileData": {"mimeType": "image/jpeg", "fileUri": "gs://.../images/image1.jpg"}}]},
     {"role": "model",  "parts": [{"text": "Product info: ... \nFlagged ingredients: [...]"}]}
   ]}
   ```

2. Upload both files to the bucket:

   ```
   gs://ingredicheck-finetune-426912/train.jsonl
   gs://ingredicheck-finetune-426912/valid.jsonl
   ```

   (You can also put them in subfolders if you prefer, e.g. `data/train.jsonl`)

3. **Important**: Make sure image `fileUri` paths in both files point to the correct location in your bucket (usually `gs://ingredicheck-finetune-426912/images/...`).

→ When you create the tuning job, you will point Vertex AI to **both** files.

## Step 4: Fine-Tune the Model on Vertex AI

1. Go to **Vertex AI > Training > Tuning**

2. Click **Create**

3. **Model**: Select `gemini-2.5-flash-lite` (or latest stable version)

4. **Dataset**:

   - **Training dataset** → browse or paste  
     `gs://ingredicheck-finetune-426912/train.jsonl`  
     (or `extractor.vertex.jsonl` if using single file)

   - **Validation dataset** → browse or paste  
     `gs://ingredicheck-finetune-426912/valid.jsonl`  
     (skip if using single file)

5. **Tuned model display name**: `ingredicheck-38-tuned-2.5-flash-lite-v1`

6. **Hyperparameters**:
   - Epochs: 2–3 (start with 2 for small dataset)
   - Adapter size: `SMALL`
   - Region: `us-central1`

7. Click **Start Training**

8. **Monitor**:
   - Go to **Tuning Jobs**
   - Refresh every 5–10 min
   - Expected time: 20–90 min (longer with images)

## Step 5: Deploy and Test the Tuned Model

1. When status = **SUCCEEDED**:
   - Go to **Vertex AI > Models**
   - Find your tuned model → **Deploy & Test** or **Deploy to endpoint**

2. Create endpoint:
   - Name: `ingredicheck-endpoint-v1`
   - Machine type: `n1-standard-2` (cheapest)
   - Deploy

3. Test in console:
   - Go to the endpoint → **Test tab**
   - Send sample request with text + image URI

## Step 6: Clean Up (to avoid charges)

- **Delete endpoint** when not using → **Endpoints > Delete**
- **Delete tuned model** (optional) → **Models > Delete**
- **Delete bucket contents** if finished → **Cloud Storage > Delete**

## Troubleshooting

- Job fails with “inaccessible URL” → make sure all `fileUri` are valid `gs://` paths
- “Invalid dataset format” → check JSONL is correctly formatted (use single-line JSON objects)
- Very long training time → reduce epochs to 1–2 or remove some images
