# AWS Face Comparison - Zero to Hero Guide

This guide explains how to build a brand new, fully independent **AWS Face Comparison** Bubble Plugin from scratch. 

This plugin is extremely powerful. It provides you with a single visual element (`Face Comparer`) that you can drop onto any page. It accepts **any** image format: standard Bubble Image URLs, raw Base64 code (like the selfies from the Face Liveness camera), or even images hidden inside AWS S3 Buckets!

---

## Step 1: Initialize the Plugin & Global Settings
1. In your Bubble Dashboard, go to your Plugins page and click **New Plugin**. Name it "AWS Face Comparison".
2. Go to the **Shared** tab. 
3. Under "Additional Keys", you must create three keys exactly like this:
    *   Name: `AWS Access Key`, Private: **Checked**
    *   Name: `AWS Secret Key`, Private: **Checked**
    *   Name: `AWS Region`, Private: **Unchecked**

*(Your users will fill these out with their AWS IAM User credentials when they install the plugin into their app).*

---

## Step 2: Create the Backend Engine (Server Action)

We need a completely secure script running on Bubble's servers to talk to AWS Rekognition.

1. Go to the **Actions** tab of your plugin.
2. Click **Add a new action**. Name it `Compare Faces API`.
3. Set "Action type" to **Server side**.
4. **Create these exact 7 Fields:** (pay attention to the naming!)
    *   `source_image_url_base64` (Type: `text`, Editor: `Dynamic value`)
    *   `source_s3_bucket` (Type: `text`, Editor: `Dynamic value`)
    *   `source_s3_key` (Type: `text`, Editor: `Dynamic value`)
    *   `target_image_url_base64` (Type: `text`, Editor: `Dynamic value`)
    *   `target_s3_bucket` (Type: `text`, Editor: `Dynamic value`)
    *   `target_s3_key` (Type: `text`, Editor: `Dynamic value`)
    *   `similarity_threshold` (Type: `number`, Editor: `Dynamic value`, Default: `80`)
5. **Create these 3 Returned Values:**
    *   `is_match` (Type: `yes/no`)
    *   `similarity_score` (Type: `number`)
    *   `error_message` (Type: `text`)
6. **Code Dependencies:** Scroll down. Check **"This action uses node modules"**. Check **"Publish item to bubble plugin"** (CRITICAL).
    *   In the `package.json` box, paste: `{ "@aws-sdk/client-rekognition": "^3.0.0" }`
7. **The Code:** Open the file `bubble_server_actions/compare_faces_universal.js` in your project folder. Copy all the code inside and paste it into the large **Action Code** box.

---

## Step 3: Create the Visual UI Element

Now we create the physical element that you will actually drag onto the page in your app.

1. Go to the **Elements** tab. Click **Add a new element**. Name it `Face Comparer`.
2. **Fields (Properties):** Add exactly the same 7 fields you added in Step 2:
    *   `source_image_url_base64` (Type: `text`, Editor: `Dynamic value`)
    *   `source_s3_bucket` (Type: `text`, Editor: `Dynamic value`)
    *   `source_s3_key` (Type: `text`, Editor: `Dynamic value`)
    *   `target_image_url_base64` (Type: `text`, Editor: `Dynamic value`)
    *   `target_s3_bucket` (Type: `text`, Editor: `Dynamic value`)
    *   `target_s3_key` (Type: `text`, Editor: `Dynamic value`)
    *   `similarity_threshold` (Type: `number`, Editor: `Dynamic value`, Default: `80`)
3. **Exposed States:** Add these 4 states:
    *   `is_match` (Type: `yes/no`)
    *   `similarity_score` (Type: `number`)
    *   `error_message` (Type: `text`)
    *   `is_comparing` (Type: `yes/no`)
4. **Events:** Add two events:
    *   `Comparison Finished`
    *   `Comparison Failed`
5. **Element Actions:** Under "Element Actions", click "Add a new action". Name it `Run Face Comparison`. In its code box, paste this:
```javascript
function(instance, properties, context) {
    if (instance.data.runComparison) instance.data.runComparison();
}
```

### The Javascript Glue
Now we paste the Javascript that connects the UI element properties to the Backend Action.

**6. The `update` Script:** At the bottom, paste this into the `update` box:
```javascript
function(instance, properties, context) {
    // Cache the latest user inputs in the element
    instance.data.props = {
        source_image_url_base64: properties.source_image_url_base64,
        source_s3_bucket: properties.source_s3_bucket,
        source_s3_key: properties.source_s3_key,
        target_image_url_base64: properties.target_image_url_base64,
        target_s3_bucket: properties.target_s3_bucket,
        target_s3_key: properties.target_s3_key,
        similarity_threshold: properties.similarity_threshold || 80
    };
}
```

**7. The `initialize` Script:** Paste this into the `initialize` box. **WARNING**: You must change `'USER_SSA_NAME'` on line 17 to match the exact internal name Bubble generated for the Server Action you created in Step 2!
```javascript
function(instance, context) {
    const container = document.createElement('div');
    container.style.display = 'none';
    instance.canvas.append(container);
    instance.data.props = {};
    
    instance.data.runComparison = function() {
        const p = instance.data.props;
        
        // We either need a URL/Base64 OR an S3 Bucket + Key for the source
        const hasSource = p.source_image_url_base64 || (p.source_s3_bucket && p.source_s3_key);
        const hasTarget = p.target_image_url_base64 || (p.target_s3_bucket && p.target_s3_key);

        if (!hasSource || !hasTarget) {
            instance.publishState('error_message', 'You forgot to provide either a URL/Base64 string OR an S3 Bucket and Key for both images!');
            instance.triggerEvent('Comparison Failed');
            return;
        }

        instance.publishState('is_comparing', true);
        instance.publishState('error_message', null);

        // TRIGGER THE SECURE SERVER ACTION!
        // Replace 'YOUR_SSA_NAME' with the actual action name from Step 2!
        context.requestAction('YOUR_SSA_NAME', p, function(err, result) {
            instance.publishState('is_comparing', false);
            
            if (err) {
                instance.publishState('error_message', err.message || 'Comparison failed.');
                instance.triggerEvent('Comparison Failed');
            } else {
                instance.publishState('is_match', result.is_match);
                instance.publishState('similarity_score', result.similarity_score);
                if(result.is_match) {
                    instance.triggerEvent('Comparison Finished');
                } else {
                    instance.publishState('error_message', "Faces don't match.");
                    instance.triggerEvent('Comparison Failed');
                }
            }
        });
    };
}
```

---

## Step 4: Connecting the Face Liveness Output!

Now the plugin is built! Here is how you use it in your app to verify the Liveness selfie against an ID card!

### The Setup
1. On your page, drag the new **Face Comparer** element onto the screen.
2. In its properties, fill out the **Source** section. (For example, set `source_image_url_base64` to `Current User's Saved_ID_Card's URL`).
3. Leave the **Target** section completely blank for now.

### The Workflows
1. You already have a workflow for: `When Face Liveness Camera Check Finished`.
2. Inside that workflow, you run your **Get Liveness Results** action.
3. Add a new step right after that: **Element Actions -> Set State**. Save the `Result of Get Liveness Results's reference_image_base64` into a custom state on your page called `Liveness_Camera_Output`.
4. Go back to your **Face Comparer** element properties on the Design screen. Now you can set `target_image_url_base64` to that `Liveness_Camera_Output` custom state!
5. In your workflow, add one final step: **Element Actions -> Face Comparer -> Run Face Comparison**.

### The Final Result
1. Make a brand new workflow: `When Face Comparer Comparison Finished`.
2. Add an action to "Log User In" or "Show Success Alert".
3. Only allow it to run when **This Face Comparer's is_match is "yes"**!
