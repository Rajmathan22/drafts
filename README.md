# Laveyoo AWS Face Liveness - Technical Architecture & Stack Documentation

## 1. High-Level Overview
Integrating AWS Rekognition Face Liveness into a no-code visual builder like Bubble.io presents a unique challenge: the AWS Amplify Liveness Face Detector is a complex React library that relies heavily on modern JavaScript bundling, whereas Bubble expects generic HTML/JavaScript web elements and separated Node.js backend execution.

To solve this, the **Laveyoo AWS Face Liveness Plugin** utilizes a **"Split Full-Stack" architecture**. 
1.  **The Frontend:** We compile the complex React application down into a single, native HTML Custom Element (a "Web Component") using Vite. Bubble simply treats it as an HTML tag: `<aws-face-liveness>`.
2.  **The Backend:** We use Bubble's Server-Side Actions (which execute in a secure Node.js 18+ environment) to communicate with the AWS Rekognition API securely.

---

## 2. The Frontend Technology Stack (`plugin-src`)

The camera UI requires rendering the `@aws-amplify/ui-react-liveness` component. We achieve this using a specific modern frontend build pipeline.

### Core Stack
*   **React 18:** The underlying UI library rendering the AWS components.
*   **AWS Amplify UI (`@aws-amplify/ui-react`):** Renders the Face Liveness detector and handles the WebRTC video streaming.
*   **TypeScript:** Used for type safety during development.
*   **Vite:** The incredibly fast build tool and bundler used to package the React code.

### The Problem with React inside Bubble
Bubble does not run React. If you just copy/paste React code into Bubble, the browser will crash. We need to bridge the gap.

### The Solution: Web Components (`react-to-webcomponent`)
We use the NPM package `react-to-webcomponent` to wrap our central React component (`LaveyooFaceLiveness.tsx`) into a native browser **Custom Element**.
*   **How it works:** It maps React `props` to HTML `attributes`. When Bubble executes `document.createElement('aws-face-liveness')`, the browser natively understands it. When Bubble updates an attribute like `session-id="123"`, the Web Component pipeline automatically translates that into a React prop update, causing the React component inside to re-render.
*   **Shadow DOM:** We initialize the Web Component with `shadow: 'open'`. This creates an isolated DOM tree. This is crucial because it ensures that the host website's CSS (Bubble styles) does not bleed into and break the highly sensitive AWS UI layout, and vice-versa.

### The CSS Problem: (`vite-plugin-css-injected-by-js`)
Web Components present a massive styling problem. AWS Amplify uses thousands of lines of CSS. If we bundle a `.css` file separately, it becomes very difficult to inject into a Shadow DOM dynamically from Bubble.
*   **Our Fix:** We use `vite-plugin-css-injected-by-js`. This Vite plugin takes the compiled AWS CSS and turns it into a giant Javascript string. When the Web Component boots up `connectedCallback()`, it dynamically builds a `<style>` tag containing all the AWS formatting and injects it directly into its own isolated Shadow DOM.

### The Final Output
Vite's configuration (`vite.config.ts`) is heavily modified to build in **"UMD Format"** with **no code-splitting**. This forces the entire React engine, AWS Amplify, our custom logic, and all CSS into **one single `.js` file** (`laveyoo-face-liveness.js`). This makes it incredibly easy for a Bubble user to upload to their "Shared Assets" and import with a single `<script>` tag.

---

## 3. The Backend Technology Stack (Server-Side Actions)

Face Liveness requires a high degree of security. You absolutely **cannot** put your AWS Access Keys in the browser, or malicious users will steal them and start fake Face Liveness sessions on your dime.

### Core Stack
*   **Bubble Server-Side Actions (SSAs):** Bubble provides a secured Node.js execution environment where custom code can run safely away from the user's browser.
*   **AWS SDK for JavaScript v3 (`@aws-sdk/client-rekognition`):** The modern, modular AWS SDK used to interact with Rekognition.

### The Actions
1.  **`start_liveness_session.js`**: Runs in Node.js. Uses the hidden AWS Access/Secret keys to call `CreateFaceLivenessSessionCommand`. It returns a temporary UUID (`SessionId`) to the frontend.
2.  **`get_liveness_results.js`**: After the video is finalized, this action calls `GetFaceLivenessSessionResultsCommand` using the `SessionId`. Because it runs on the server, the final score cannot be tampered with by the user in the browser.

### Reference Image Buffer Processing
When the results are fetched, AWS does not return a simple image URL. It returns raw Machine Code bytes (`Uint8Array`). Our backend Node.js script intercepts these bytes, loads them into a Node `Buffer`, and encodes them into a **Base64 String** (`data:image/jpeg;base64,...`). This is the only format Bubble can natively assign to an Image element via dynamic data fields without needing a separate S3 bucket upload workflow.

---

## 4. The Security Model

This plugin uses a **Dual-Credential Authorization** method.

### 1. Server Authentication (Permanent Keys)
Your permanent `AWS Access Key` and `AWS Secret Key` are stored in Bubble's "Shared" Plugin settings. These keys are heavily restricted via IAM policies to *only* allow `rekognition:CreateFaceLivenessSession` and `rekognition:GetFaceLivenessSessionResults`. These keys never leave the Bubble servers.

### 2. Browser Authentication (Temporary Guest Access)
The React camera component still needs permission to stream video directly to AWS servers. To do this without exposing your permanent keys, we utilize an **Amazon Cognito Identity Pool**.
*   The Identity Pool is configured for **Unauthenticated Guest Access**.
*   The React component receives the public `identity-pool-id`.
*   Behind the scenes, `Amplify.configure()` talks to Cognito, trades the Pool ID for temporary, severely restricted STS (Security Token Service) credentials that expire shortly and only have permission to stream video for that specific `Session ID`.

---

## 5. End-to-End Sequence Flow

1. **User Clicks "Start Scan"** in Bubble.
2. Bubble executes the **Start Liveness Session** backend action securely.
3. Node.js backend contacts AWS -> AWS returns `SessionId 123`.
4. Bubble updates the custom state on the page, passing `SessionId 123` to the Frontend Web Component.
5. The **Web Component** (`<aws-face-liveness>`) wakes up. It uses the Cognito Identity Pool ID to get temporary credentials for the browser.
6. The user completes the facial scan video. The Web Component streams this directly to AWS.
7. AWS finishes processing. The Web Component dispatches a DOM Event (`liveness_complete`).
8. The Bubble Element catches the event and triggers a Bubble Workflow.
9. Bubble executes the **Get Liveness Results** backend action securely, passing `SessionId 123`.
10. Node.js backend contacts AWS -> AWS returns the real `ConfidenceScore` and `ReferenceImage` byte array.
11. Backend translates bytes to Base64 and returns the data to Bubble.
12. Bubble UI updates to show the Score and Image.
