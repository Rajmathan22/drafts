# AWS Face Comparison - Bubble Setup Guide

This guide will walk you through creating the new standalone **AWS Face Comparison** plugin in Bubble and connecting it to your existing Face Liveness check.

## Step 1: Create the Plugin
1. Go to your Bubble Dashboard Plugins page and click **New Plugin**. Name it "AWS Face Comparison".
2. Go to the **Shared** tab. 
3. Under "Additional Keys", create three keys:
    *   Name: `AWS Access Key`, Private: **Checked**
    *   Name: `AWS Secret Key`, Private: **Checked**
    *   Name: `AWS Region`, Private: **Unchecked**

## Step 2: Create Action 1 (Direct Upload / Base64)

This is the most common action. It compares two images provided directly via URLs or Base64 strings.

1. Go to the **Actions** tab.
2. Click **Add a new action**. Name it `Compare Faces (Direct Upload / Base64)`.
3. Set "Action type" to **Server side**.
4. Create 3 **Fields**:
    *   `source_image` (Type: `text`, Editor: `Dynamic value`)
    *   `target_image` (Type: `text`, Editor: `Dynamic value`)
    *   `similarity_threshold` (Type: `number`, Default: `80`, Editor: `Dynamic value`)
5. Create 3 **Returned Values**:
    *   `is_match` (Type: `yes/no`)
    *   `similarity_score` (Type: `number`)
    *   `error_message` (Type: `text`)
6. Scroll down to the action code. Check **"This action uses node modules"**.
7. In the modules package.json box, paste:
```json
{ "@aws-sdk/client-rekognition": "^3.0.0" }
```
8. In the big **Action Code** box, paste the code from the `bubble_server_actions/compare_faces_base64.js` file provided!

## Step 3: Create Action 2 (S3 Buckets)

This action is for high-security environments where the images are already safely locked in an AWS S3 Bucket.

1. Still on the **Actions** tab, click **Add a new action**. Name it `Compare Faces (S3 Bucket)`.
2. Set "Action type" to **Server side**.
3. Create 5 **Fields**:
    *   `source_bucket` (Type: `text`)
    *   `source_filename` (Type: `text`)
    *   `target_bucket` (Type: `text`)
    *   `target_filename` (Type: `text`)
    *   `similarity_threshold` (Type: `number`, Default: `80`)
4. Create 3 **Returned Values**:
    *   `is_match` (Type: `yes/no`)
    *   `similarity_score` (Type: `number`)
    *   `error_message` (Type: `text`)
5. Code dependencies: Check the node modules box and add `{ "@aws-sdk/client-rekognition": "^3.0.0" }`.
6. Use the code from the `bubble_server_actions/compare_faces_s3.js` file!

---

## Step 4: Using it in your Bubble App!

Here is how you wire your new Comparison Plugin together with your original Liveness Plugin:

1. In your Bubble App, install the newest "AWS Face Comparison" plugin and fill out your AWS Keys.
2. Assuming you already have the **Face Liveness Camera** element on your page, find your Workflow where you call **Get Liveness Results** (Workflow 2 from the previous guide).
3. Right after the **Get Liveness Results** step, click *Click here to add an action*.
4. Go to **Plugins** -> **Compare Faces (Direct Upload / Base64)**.
    *   **source_image:** Usually, this is the user's saved ID photo. (e.g. `Current User's ID_Photo's URL`)
    *   **target_image:** This is the brand new liveness photo! Click "Insert dynamic data" -> **Result of step X (Get Liveness Results)** -> **reference_image_base64**.
    *   **similarity_threshold:** `80` (or `90` for strict security).
5. **Add a condition:** Now you can add a final step (like "Log the user in" or "Update User status to Verified"). Only allow this final step to occur *only when* **Result of Get Liveness Results's is_live is "yes"** AND **Result of Compare Faces' is_match is "yes"**!
