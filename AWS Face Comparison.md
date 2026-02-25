# AWS Face Comparison

**Plugin details**
Perform highly secure biometric identity verification by utilizing intelligent Face Comparison technology. Compare faces between official ID documents, profile pictures, selfies, or images stored in S3 to verify identity.
This plugin seamlessly integrates with outputs from Liveness checks or standard Bubble picture uploaders.

The following actions are provided:
- COMPARE FACES API (BACK-END)

The plugin accepts an image in multiple formats: public Image URLs (like Bubble database uploads), raw Base64 strings, or secure AWS S3 Bucket locations. It compares a Source image with a Target image, returning a match boolean and a similarity score.

---

## 📖 Instructions

1️⃣: COMPARE FACES API (BACK-END)
================================

📋ACTION DESCRIPTION
--------------------------------
The COMPARE FACES API backend action compares two faces to determine if they are the same person. It intelligently accepts an image in multiple formats: public Image URLs (like Bubble database uploads), raw Base64 strings, or secure AWS S3 Bucket locations.
This is heavily utilized to verify that a person in a selfie or liveness check is truly the owner of the account or ID card on file.

🔧 STEP-BY-STEP SETUP
--------------------------------
0) Sign-up for an AWS Account and access the IAM console: https://console.aws.amazon.com/iam/home

1) Create your AWS REKOGNITION ACCESS KEY & SECRET ACCESS KEY. Create an IAM user and attach a policy allowing the `rekognition:CompareFaces` action.
If you intend to use AWS S3 buckets for the images, ensure your policy also has `s3:GetObject` permissions for the bucket.

2) Enter in the **Shared** tab of your Bubble PLUGIN SETTINGS the `AWS Access Key`, `AWS Secret Key`, and the `AWS Region` (e.g., `us-east-1` or `us-west-2`). 

3) Set up the action "COMPARE FACES API" action in the workflow where you want to perform the comparison.

   Input Fields:
      - SOURCE_IMAGE_URL_BASE64: An image URL (https://) or Base64 string of the first face (e.g., the user's ID Card).
      - SOURCE_S3_BUCKET / KEY: If not using a URL, the AWS S3 location of the first image.
      - TARGET_IMAGE_URL_BASE64: An image URL (https://) or Base64 string of the second face (e.g., a selfie or output from a camera).
      - TARGET_S3_BUCKET / KEY: If not using a URL, the AWS S3 location of the second image.
      - SIMILARITY_THRESHOLD: Specifies the minimum confidence level for a match. AWS REKOGNITION doesn't return a match if lower than this specified value. Minimum value of 0. Maximum value of 100. Default value is 80 if not provided.

   Output Fields:
      - IS_MATCH: A Yes/No boolean stating if the AWS algorithm determined the faces belong to the same person.
      - SIMILARITY_SCORE: The exact confidence level (0-100) of the match.
      - ERROR_MESSAGE: Populated if AWS could not detect a face, or if the images were improperly formatted.


🔍INTEGRATION WORKFLOW EXAMPLE
======================
1. User captures a selfie via Picture Uploader or a Liveness plugin.
2. User provides an ID card image.
3. In your workflow, run **Compare Faces API**.
4. Set the *Source* of the comparison to the uploaded ID card image URL. Set the *Target* to the selfie image (URL or Base64).
5. If `is_match` is "Yes", log the user in or mark their account as Verified!


ℹ️ADDITIONAL INFORMATION
======================
> Compare Faces details: https://docs.aws.amazon.com/rekognition/latest/dg/faces-comparefaces.html
> AWS services availability per region: https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/


⚠️TROUBLESHOOTING
================
Any plugin related error will be posted to the Logs tab, "Server logs" section of your App Editor.
Make sure that "Plugin server side logging" and "Plugin client side logging" are selected in "Show Advanced".

Always check the ERROR MESSAGE state returned by backend actions to provide a better user experience.


⚡PERFORMANCE CONSIDERATIONS
===========================

GENERAL
-------------
The `Compare Faces API` automatically analyzes the input type provided. If a Bubble Image URL is provided, the backend immediately utilizes Node.js `fetch` to stream the image directly into an `ArrayBuffer` in memory, completely skipping heavy Base64 string conversions for drastically improved speed.

⏱️BACK-END ACTION START DELAY
-----------------------------------------------
Each time a server-side action is called, Bubble initializes a small virtual machine to execute the action. If the same action is called shortly after, the caching mechanism kicks in, resulting in faster execution on subsequent calls.
