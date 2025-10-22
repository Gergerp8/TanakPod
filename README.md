# 🌳 TanakPod AI - Field Intelligence Console

**TanakPod AI** is an agricultural monitoring system that combines IoT sensor data, weather forecasting, and AI-powered decision-making to provide real-time recommendations for tree/crop management via WhatsApp notifications.

## 📦 Prerequisites

Before you begin, you'll need:

- ✅ AWS Account with billing enabled
- ✅ Firebase account 
- ✅ Twilio account (free tier works for sandbox)
- ✅ Basic understanding of AWS Lambda, IAM roles, and environment variables
- ✅ Command-line access (macOS, Linux, or Windows with Git Bash/WSL)

---

## 🚀 Setup Instructions

### 1. AWS Console Setup

#### Step 1.1: Select Your Region

1. Log in to [AWS Console](https://console.aws.amazon.com/)
2. In the top-right navigation bar, select your preferred region (e.g., `ap-southeast-1`)

---

#### Step 1.2: Amazon Bedrock Configuration

1. Navigate to **Amazon Bedrock** service
2. Go to **Model access** → **Enable specific models**
3. Browse the **Model Catalog** and enable your preferred model
   - **Default recommendation**: `Claude 3 Haiku` (fast and cost-effective)
   - Alternatives: Claude 3 Sonnet, Claude 3 Opus, or other available models

---

#### Step 1.3: Create Lambda Functions

You need to create **2 Lambda functions**: `ingest_fn` and `decision_fn`.

##### 📥 **Function 1: Ingest Function**

1. Navigate to **AWS Lambda** → **Functions** → **Create function**
2. **Configuration**:
   - Function name: `ingest` (or your preferred name)
   - Runtime: Python 3.x
3. Click **Create function**
4. **Upload code**:
   - Click **Upload from** → **.zip file**
   - Upload your `ingest_fn.zip` file

##### ⚙️ **Environment Variables** (5 required):

| Variable Name | Description | Example |
|--------------|-------------|---------|
| `decision_fn_arn` | ARN of your Decision Lambda function |`arn:aws:lambda:ap-southeast-1:123456789012:function:decision` |
| `FIREBASE_CREDENTIALS_JSON_BASE64` | Base64-encoded Firebase service account key | (See instructions below) |
| `FIREBASE_DATABASE_URL` | Your Firebase Realtime Database URL | `https://your-project.firebaseio.com` |
| `FIREBASE_PROJECT_ID` | Your Firebase project ID | `your-project-id` |
| `FIREBASE_SA_SECRET` | Name of your secret in AWS Secrets Manager | `firebase-secret` |

**How to get `FIREBASE_CREDENTIALS_JSON_BASE64`:**

1. Go to [Firebase Console](https://console.firebase.google.com/)
2. Select your project → **Project Settings** → **Service Accounts**
3. Click **Generate new private key** → Download `serviceAccountKey.json`
4. Run this command in your terminal:
   ```bash
   base64 -w 0 serviceAccountKey.json
   ```
5. Copy the output and paste it as the environment variable value

**Example `serviceAccountKey.json` structure:**
```json
{
  "type": "service_account",
  "project_id": "<YOUR_PROJECT_ID>",
  "private_key_id": "<YOUR_PRIVATE_KEY_ID>",
  "private_key": "-----BEGIN PRIVATE KEY-----\n<YOUR_PRIVATE_KEY>\n-----END PRIVATE KEY-----\n",
  "client_email": "<YOUR_CLIENT_EMAIL>",
  "client_id": "<YOUR_CLIENT_ID>",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token",
  "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
  "client_x509_cert_url": "<YOUR_CERT_URL>",
  "universe_domain": "googleapis.com"
}
```

##### ⚙️ **General Configuration**:

- **Configuration** → **General configuration** → **Edit**
- Set **Timeout**: `30 seconds`

##### 🌐 **Function URL** (Enable CORS):

1. Go to **Configuration** → **Function URL** → **Create function URL**
2. **Auth type**: `NONE` (for public access)
3. **Configure cross-origin resource sharing (CORS)**:
   - ✅ Enable CORS
   - **Allow origin**: `*`
   - **Allow methods**: `POST`
   - **Allow headers**: `content-type`
4. **Save** and copy the Function URL (you'll need this for the HTML file)

---

##### 🎯 **Function 2: Decision Function**

1. Navigate to **AWS Lambda** → **Functions** → **Create function**
2. **Configuration**:
   - Function name: `decision` (or your preferred name)
   - Runtime: Python 3.x
3. Click **Create function**
4. **Upload code**:
   - Click **Upload from** → **.zip file**
   - Upload your `decision_fn.zip` file

##### ⚙️ **Environment Variables** (8 required):

| Variable Name | Description | Example |
|--------------|-------------|---------|
| `FIREBASE_CREDENTIALS_JSON_BASE64` | Same value as ingest function | (Base64 string) |
| `FIREBASE_DATABASE_URL` | Your Firebase Realtime Database URL | `https://your-project.firebaseio.com` |
| `FIREBASE_PROJECT_ID` | Your Firebase project ID | `your-project-id` |
| `FIREBASE_SA_SECRET` | Name of your secret in AWS Secrets Manager | `firebase-secret` |
| `TWILIO_ACCOUNT_SID` | Your Twilio Account SID | `ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |
| `TWILIO_AUTH_TOKEN` | Your Twilio Auth Token | `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |
| `TWILIO_FROM_NUMBER` | Twilio sandbox WhatsApp number | `whatsapp:+14155238886` |
| `TWILIO_TO_NUMBER` | Your phone number | `whatsapp:+60123456789` |

##### ⚙️ **General Configuration**:

- **Configuration** → **General configuration** → **Edit**
- Set **Timeout**: `15 seconds`
- Set **Memory**: `512 MB` (adjust as needed)

---

#### Step 1.4: IAM Roles and Permissions

Both Lambda functions need proper IAM roles with the following policies:

1. Navigate to **IAM** → **Roles**
2. Find the role automatically created for each Lambda function
3. Click **Add permissions** → **Attach policies**
4. Add these managed policies:
   - ✅ `AWSLambdaBasicExecutionRole` (CloudWatch logging)
   - ✅ `AmazonBedrockFullAccess` (AI model access)
5. Create a custom inline policy named `AllowBedrockInvoke`:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "bedrock:InvokeModel",
           "bedrock:InvokeModelWithResponseStream"
         ],
         "Resource": "*"
       }
     ]
   }
   ```
6. Create a custom inline policy named `FirebaseSecretAccess`:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "secretsmanager:GetSecretValue"
         ],
         "Resource": "arn:aws:secretsmanager:YOUR_REGION:YOUR_ACCOUNT_ID:secret:YOUR_SECRET_NAME*"
       }
     ]
   }
   ```

---

#### Step 1.5: AWS Secrets Manager

1. Navigate to **AWS Secrets Manager**
2. Click **Store a new secret**
3. **Secret type**: `Other type of secret`
4. Add your Firebase credentials as key-value pairs or JSON
5. **Secret name**: `firebase-secret` (or match your `FIREBASE_SA_SECRET` env var)
6. Click **Store**

---

### 2. Firebase Setup

TanakPod writes AI decisions into **Firebase Realtime Database** from the **Decision Lambda** using the **Firebase Admin SDK**. Follow these steps to configure your Firebase project.

---

### 1. Create a Firebase Project

1. Go to the [Firebase Console](https://console.firebase.google.com/).
2. Click **Add project** → name your project → finish setup (Google Analytics optional).

---

### 2. Enable Realtime Database

1. In your Firebase project sidebar, go to **Build → Realtime Database**.
2. Click **Create database** → choose your region.
3. Start in **locked mode** (recommended).
4. Copy your database URL — it will look something like:

```
https://<PROJECT_ID>.firebaseio.com
```

or

```
https://<PROJECT_ID>-default-rtdb.asia-southeast1.firebasedatabase.app
```

You’ll use this later as your `FIREBASE_DATABASE_URL`.

---

### 3. Generate a Service Account Key

1. Go to **Project Settings (gear icon)** → **Service accounts** → **Firebase Admin SDK**.
2. Click **Generate new private key** to download `serviceAccountKey.json`.
3. Keep it private. You’ll encode it for AWS Lambda in the next step.

---

### 4. Base64-encode the Service Account JSON

Encode your `serviceAccountKey.json` file to a single-line Base64 string:

**macOS**

```bash
base64 serviceAccountKey.json | tr -d '\n' > firebase_sa.b64
```

**Linux**

```bash
base64 -w 0 serviceAccountKey.json > firebase_sa.b64
```

**Windows PowerShell**

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("serviceAccountKey.json")) > firebase_sa.b64
```

Open the `firebase_sa.b64` file and copy the entire single line (no spaces or newlines).

---

### 5. Add Environment Variables (Decision Lambda Only)

Open your **AWS Lambda → Decision Function → Configuration → Environment variables → Edit** and add:

| Key                                | Value                                         |
| ---------------------------------- | --------------------------------------------- |
| `FIREBASE_CREDENTIALS_JSON_BASE64` | *(paste the Base64 string from Step 4)*       |
| `FIREBASE_DATABASE_URL`            | *(paste your Firebase Realtime Database URL)* |

> Note: Only the **Decision Lambda** needs Firebase environment variables. The **Ingest Lambda** does not interact with Firebase.

---

### 6. Include the Firebase Admin SDK in Lambda

Your Decision Lambda imports `firebase_admin`. Make sure the dependency is available when you deploy.

**Option A: Package in zip file**

```bash
mkdir pkg && cd pkg
pip install firebase-admin -t .
cp ../decision_lambda.py .
zip -r decision_fn.zip .
```

Upload `decision_fn.zip` to your Decision Lambda function.

**Option B: Use a Lambda Layer**
Create a layer that includes `firebase-admin` and attach it to your Decision Lambda.

If you’ve already seen `✅ Successfully saved decision to Firebase.` in CloudWatch logs, this step is complete.

---

### 7. Optional: Secure Database Rules

Because the Admin SDK uses a service account, Firebase rules **don’t restrict** your server writes.
You can still lock the database for safety:

**Default secure rules:**

```json
{
  "rules": {
    ".read": false,
    ".write": false,
    "decision_logs": {
      ".read": false,
      ".write": false
    }
  }
}
```

**If you want authenticated client apps to read decisions:**

```json
{
  "rules": {
    "decision_logs": {
      ".read": "auth != null",
      ".write": false
    }
  }
}
```

---

### 8. Verify Connection

Run the Decision Lambda with a test event:

```json
{
  "tree_id": "Test-Tree-1",
  "soil_moisture": 0.12,
  "electrical_conductivity": 1.4,
  "soil_nutrient_index": 0.35,
  "spectral_idx": 0.74,
  "rain_mm_next_24h": 5.0
}
```

You should see this in **CloudWatch logs**:

```
✅ Successfully saved decision to Firebase.
```

In Firebase Console → **Realtime Database**, verify that you see a new entry:

```
decision_logs
  └── -Nh... (auto key)
      ├─ tree_id: "Test-Tree-1"
      ├─ actions: ["IRRIGATE"]
      ├─ rationale: "..."
      └─ timestamp: "2025-10-22T..."
```

---

### 9. Troubleshooting

| Issue                                      | Cause / Fix                                                                         |
| ------------------------------------------ | ----------------------------------------------------------------------------------- |
| `Missing FIREBASE_CREDENTIALS_JSON_BASE64` | Environment variable not set or empty.                                              |
| `Invalid Database URL`                     | Use your exact Firebase Realtime Database URL (not Firestore).                      |
| `Permission denied`                        | Happens if testing with a locked database via client SDK — Admin SDK bypasses this. |
| `UnicodeDecodeError`                       | Your Base64 file contains line breaks — ensure it’s one continuous line.            |

---

Once complete, your Decision Lambda will automatically log every AI decision to Firebase in real time.

---

### 3. Twilio Setup

#### Step 3.1: Create Twilio Account

1. Sign up at [Twilio](https://www.twilio.com/try-twilio)
2. Complete verification (phone number required)

#### Step 3.2: WhatsApp Sandbox Setup

1. In the Twilio Console sidebar, go to **Develop** → **Messaging** → **Try it out**
2. Click **Send a WhatsApp message**
3. Follow the instructions to join the sandbox:
   - Send the provided join code (e.g., `join <word>-<word>`) to the Twilio WhatsApp number from your phone
4. You should receive a confirmation message

#### Step 3.3: Get Credentials

1. Navigate to **Account** → **Account Info**
2. Copy:
   - **Account SID** → Use for `TWILIO_ACCOUNT_SID`
   - **Auth Token** → Use for `TWILIO_AUTH_TOKEN`
3. Navigate to **Messaging** → **Try it out** → **Send a WhatsApp message**
4. Copy:
   - **From number** (e.g., `whatsapp:+14155238886`) → Use for `TWILIO_FROM_NUMBER`
   - Your phone number with country code (e.g., `whatsapp:+60123456789`) → Use for `TWILIO_TO_NUMBER`

---

### 4. Website Configuration

1. Open `Tanakpod.html` in a text editor
2. Find line 251:
   ```javascript
   const INGEST_URL = "https://irkk34opiwl5a3mepkpq5hgpgm0tvaml.lambda-url.ap-southeast-1.on.aws/";
   ```
3. Replace with your **Ingest Lambda Function URL** from Step 1.3
4. Save the file
5. Open `Tanakpod.html` in a modern web browser (Chrome, Firefox, Edge, Safari)

---

## 💡 Usage

### Quick Start Demo

1. Open `Tanakpod.html` in your browser
2. The weather preview will load automatically for the default coordinates
3. Enter test data:
   - **Tree ID**: `Plot7-Tree19`
   - **Soil moisture**: `0.12` (triggers irrigation recommendation)
   - **Electrical conductivity**: `1.40`
   - **Soil nutrient index**: `0.32` (triggers fertilizer recommendation)
   - **Drone spectral index**: `0.75` (with low rain, triggers harvest recommendation)
4. Optionally change **Latitude** and **Longitude** for your location
5. Click **🔎 Preview 24h Rain** to see weather forecast
6. Click **🚀 Send to AI** to dispatch data to AWS Lambda
7. Check your phone for a WhatsApp message with AI recommendations!

### Decision Logic Examples

| Condition | Recommendation |
|-----------|----------------|
| Soil moisture < 0.20 | 💧 **Irrigate** the tree |
| Soil nutrient index < 0.40 | 🌱 **Fertilize** the tree |
| Spectral index > 0.70 + low rain | 🌾 **Harvest** is recommended |

---

## 🔧 Troubleshooting

### Issue: CORS Error in Browser Console

**Solution**: Ensure your Ingest Lambda Function URL has CORS enabled:
- Allow origin: `*`
- Allow methods: `POST`
- Allow headers: `content-type`

---

### Issue: Lambda Timeout

**Solution**:
- Increase timeout in **Configuration** → **General configuration**
- Ingest function: 30 seconds
- Decision function: 15 seconds

---

### Issue: WhatsApp Message Not Received

**Checklist**:
- ✅ Joined Twilio sandbox (sent join code)
- ✅ Correct phone number format: `whatsapp:+[country code][number]`
- ✅ All Twilio environment variables set correctly
- ✅ Check Twilio console logs for errors

---

### Issue: Firebase Connection Error

**Checklist**:
- ✅ Service account key is correctly base64 encoded
- ✅ Firebase Database URL is correct (must end with `.firebaseio.com`)
- ✅ Firebase Realtime Database is enabled (not just Firestore)
- ✅ Database rules allow writes

---

### Issue: Bedrock Model Access Denied

**Solution**:
1. Go to **Amazon Bedrock** → **Model access**
2. Enable your chosen model (Claude 3 Haiku)
3. Wait for access to be granted (can take a few minutes)
4. Verify IAM role has Bedrock permissions

---

### Debug Mode

The website includes comprehensive console logging. To debug:

1. Open browser DevTools (F12 or Right-click → Inspect)
2. Go to **Console** tab
3. Click **🚀 Send to AI**
4. Review detailed logs showing:
   - Payload structure
   - Request/response status
   - Error messages

---

## 📄 License

This project is provided as-is for educational and demonstration purposes.

---

## 🤝 Support

For issues or questions:
1. Check the [Troubleshooting](#-troubleshooting) section
2. Review browser console logs
3. Check AWS CloudWatch logs for Lambda functions
4. Verify all environment variables are correctly set

---

**Built with ❤️ for smart agriculture**

