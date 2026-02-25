# AWS Face Comparison - The Ultimate Plugin Guide

After reviewing Bubble's architecture for your specific request, **we do not need to build a visual "Element" at all.**

In Bubble, if a plugin doesn't need to display anything visually on the screen (like a video player or a button), forcing it into the "Elements" tab leads to confusing UI bugs. 

The absolute best, native way to build this in Bubble is as a **Server-Side Action**. This creates an independent, universal "tool block" that you can drop into **any** workflow in your app. 

### What data types can it handle?
This single powerful action can handle **all three** major data types simultaneously:
1.  **Direct Base64:** Accepts raw code strings like the `reference_image_base64` output from the Liveness Camera.
2.  **Public URLs:** Accepts standard Bubble Image URLs (e.g., photos uploaded by users and saved in your database).
3.  **AWS S3 Buckets:** For extreme security, it accepts raw AWS S3 Bucket names and Object Keys if your images are locked down.

### Where is the input given?
You provide the inputs directly inside the **Workflow Editor**, not the Design tab. When you click a button on your app and open a workflow, you add this action. A clean property box will appear asking you for the Source Image and Target Image right there!

---

## Step 1: Initialize the Plugin & Global Settings
1. In your Bubble Dashboard, go to your Plugins page and click **New Plugin**. Name it "AWS Face Comparison".
2. Go to the **Shared** tab. 
3. Under "Additional Keys", you must create three keys exactly like this:
    *   Name: `AWS Access Key`, Private: **Checked**
    *   Name: `AWS Secret Key`, Private: **Checked**
    *   Name: `AWS Region`, Private: **Unchecked**

---

## Step 2: Create the Universal Action

This is the only thing we need to build for the plugin to work!

1. Go to the **Actions** tab of your plugin. (Ignore the Elements tab completely).
2. Click **Add a new action**. Name it `Compare Faces API`.
3. Set "Action type" to **Server side**.
4. **Create these 7 Input Fields:** (These will appear in your workflow editor)
    *   `source_image_url_base64` (Type: `text`, Editor: `Dynamic value`)
    *   `source_s3_bucket` (Type: `text`, Editor: `Dynamic value`)
    *   `source_s3_key` (Type: `text`, Editor: `Dynamic value`)
    *   `target_image_url_base64` (Type: `text`, Editor: `Dynamic value`)
    *   `target_s3_bucket` (Type: `text`, Editor: `Dynamic value`)
    *   `target_s3_key` (Type: `text`, Editor: `Dynamic value`)
    *   `similarity_threshold` (Type: `number`, Editor: `Dynamic value`, Default: `80`)
5. **Create these 3 Returned Values:** (These will be output back into the workflow)
    *   `is_match` (Type: `yes/no`)
    *   `similarity_score` (Type: `number`)
    *   `error_message` (Type: `text`)
6. **Code Dependencies:** Scroll down. Check **"This action uses node modules"**. 
    *   In the `package.json` box, paste: `{ "@aws-sdk/client-rekognition": "^3.0.0" }`
7. **The Code:** Open the file `bubble_server_actions/compare_faces_universal.js` in your project folder. Copy all the code inside and paste it into the large **Action Code** box at the bottom.

---

## Step 3: How to use it in your App!

You are done building the plugin! Now, how do you actually use it in your Bubble app?

1. Install your new plugin into your app and put your AWS keys in the settings.
2. Go to your app's **Workflow** tab. 
3. Whenever you want to compare faces (e.g., `When a Button "Verify Identity" is clicked`), click to add an action.
4. Go to **Plugins -> Compare Faces API**.
5. The action box will appear. You just fill in the dynamic data!
    *   *Example:* For `source_image_url_base64`, you insert dynamic data to equal `Current User's SavedIDPhoto's URL`.
    *   *Example:* For `target_image_url_base64`, you insert whatever new photo you just captured (from a Picture Uploader element, or from the `reference_image_base64` of the Liveness plugin!).
    *   *Note: If you are using URL/Base64, just leave the S3 Bucket/Key fields completely empty!*
6. **Get the Results:** Add a final step to your workflow (like "Show Success Popup"). Set it to *only run when* **Result of step X (Compare Faces API)'s is_match is "yes"**. You can also display the text of `Result of step X's similarity_score` on the screen!
