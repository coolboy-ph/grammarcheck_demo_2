# AWS S3 + CloudFront + Aws Certificate Manager + Route 53 + Firebase

Here’s a step-by-step guide to set up the following stack:
- index.html hosted on S3
- Served via CloudFront
- HTTPS via AWS Certificate Manager
- Route 53 for DNS
- Firebase Oauth 2.0 for authentication

## 🧾 Step-by-Step Setup
### ✅ 1. Prepare index.html File
Create your `index.html` file locally. This will be uploaded to S3.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Grammar Checker with Firebase Auth</title>
  <script type="module">
    // Firebase SDK imports
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js";
    import {
      getAuth,
      signInWithPopup,
      GoogleAuthProvider,
      signOut,
      onAuthStateChanged
    } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-auth.js";

    // Replace with your Firebase config
    const firebaseConfig = {
      apiKey: "YOUR_API_KEY",
      authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
      projectId: "YOUR_PROJECT_ID",
      storageBucket: "YOUR_STORAGE_BUCKET",
      messagingSenderId: "YOUR_SENDER_ID",
      appId: "YOUR_APP_ID",
      measurementId: "YOUR_MEASUREMENT_ID"
    };

    const app = initializeApp(firebaseConfig);
    const auth = getAuth(app);
    const provider = new GoogleAuthProvider();

    const loginBtn = document.getElementById("loginBtn");
    const logoutBtn = document.getElementById("logoutBtn");
    const userInfo = document.getElementById("userInfo");
    const userName = document.getElementById("userName");
    const userEmail = document.getElementById("userEmail");
    const textInput = document.getElementById("textInput");
    const checkBtn = document.getElementById("checkBtn");
    const outputDiv = document.getElementById("output");

    loginBtn.addEventListener("click", () => {
      signInWithPopup(auth, provider)
        .then(result => {
          const user = result.user;
          userName.textContent = user.displayName;
          userEmail.textContent = user.email;
        })
        .catch(err => alert("Login failed: " + err.message));
    });

    logoutBtn.addEventListener("click", () => {
      signOut(auth);
    });

    onAuthStateChanged(auth, user => {
      if (user) {
        userInfo.style.display = "flex";
        loginBtn.style.display = "none";
        userName.textContent = user.displayName;
        userEmail.textContent = user.email;
        textInput.disabled = false;
        checkBtn.disabled = false;
      } else {
        userInfo.style.display = "none";
        loginBtn.style.display = "inline-block";
        textInput.disabled = true;
        checkBtn.disabled = true;
        textInput.value = "";
        outputDiv.innerText = "Your corrected text will appear here.";
      }
    });

    // Grammar check logic
    window.checkGrammar = async function () {
      const text = textInput.value;
      outputDiv.innerText = "Checking grammar...";

      const params = new URLSearchParams();
      params.append("text", text);
      params.append("language", "en-US");

      try {
        const res = await fetch("https://api.languagetoolplus.com/v2/check", {
          method: "POST",
          headers: { "Content-Type": "application/x-www-form-urlencoded" },
          body: params.toString()
        });

        const data = await res.json();

        if (data.matches.length === 0) {
          outputDiv.innerHTML = "<b>No grammar issues found!</b>";
          return;
        }

        let result = text;
        data.matches.reverse().forEach(match => {
          const replacement = match.replacements[0]?.value || "";
          if (replacement) {
            const start = match.offset;
            const end = match.offset + match.length;
            result = result.slice(0, start) +
              `<span class="highlight" title="${match.message}">${replacement}</span>` +
              result.slice(end);
          }
        });

        outputDiv.innerHTML = `<b>Suggested Correction:</b><br>${result}`;
      } catch (err) {
        outputDiv.innerText = "Error: " + err.message;
      }
    };
  </script>
  <style>
    body { font-family: sans-serif; padding: 2rem; background: #f9fafb; }
    .container { max-width: 720px; margin: auto; background: white; border-radius: 16px; padding: 2rem; box-shadow: 0 0 20px #ccc; }
    .output { margin-top: 2rem; background: #f3f4f6; padding: 1rem; border-radius: 12px; }
    .highlight { background: #fde68a; padding: 0 4px; border-radius: 4px; border-bottom: 2px dotted #f59e0b; }
    .login-btn, .logout-btn, button { background: #2563eb; color: white; padding: 0.75rem 1.5rem; border: none; border-radius: 8px; cursor: pointer; margin-top: 1rem; }
    .logout-btn { background: #ef4444; }
    .user-info { display: none; justify-content: space-between; align-items: center; margin: 1rem 0; padding: 1rem; border: 1px solid #ccc; border-radius: 12px; background: #f0fdf4; }
    textarea { width: 100%; padding: 1rem; margin-top: 1rem; border-radius: 12px; border: 1px solid #ddd; resize: vertical; }
    .login-info { background: #e0f2fe; border: 1px solid #90cdf4; padding: 1rem; border-radius: 12px; margin-bottom: 1rem; font-size: 1rem; text-align: center; }
    .title-box { text-align: center; }
  </style>
</head>
<body>
  <div class="container">
    <div class="title-box">
      <h2>📝 English Grammar Checker</h2>
    </div>

    <div class="login-info">
      🔐 <strong>You can sign in with your Google account</strong><br>
      to access the Grammar Checker.
    </div>

    <button id="loginBtn" class="login-btn">Sign in with Google</button>

    <div id="userInfo" class="user-info">
      <div>
        <div id="userName" style="font-weight: bold;"></div>
        <div id="userEmail" style="font-size: 0.9rem; color: #6b7280;"></div>
      </div>
      <button id="logoutBtn" class="logout-btn">Logout</button>
    </div>

    <textarea id="textInput" placeholder="Example: She go to school everyday." disabled></textarea>
    <button onclick="checkGrammar()" id="checkBtn" disabled>Check Grammar</button>

    <div class="output" id="output">Your corrected text will appear here.</div>
  </div>
</body>
</html>
```

### ✅ 2. Create and Configure S3 Bucket
a. Create S3 Bucket
  - Go to S3 Console
  - Click Create bucket
  - Name: `your-domain-name`
  - Uncheck "Block all public access"
  - Keep static website hosting disabled (use CloudFront instead)

b. Upload index.html
  - Open the bucket
  - Click Upload
  - Upload your index.html

c. Set Object Permissions
  - Go to the uploaded file
  - Click Permissions > Edit
  - let CloudFront handle access (recommended)

### ✅ 3. Create an SSL Certificate (ACM)
a. Go to AWS Certificate Manager
  - Request a certificate
  - Add domain: `your-domain-name` and `www.your-domain-name` (optional)

b. DNS Validation via Route 53
  - ACM will suggest a CNAME
  - Click Create record in Route 53

c. Wait until status is "Issued"

### ✅ 4. Set Up CloudFront Distribution
a. Create a Distribution
  - Origin domain: Select your S3 bucket (select bucket name, not website endpoint)
  - Viewer Protocol Policy: Redirect HTTP to HTTPS
  - Origin Access Control (OAC): Create a new one (recommended)
  - Restrict Bucket Access: Yes (use OAC)
  - Default Root Object: index.html
  - Alternate Domain Name (CNAME): `your-domain-name`
  - Custom SSL Certificate: Choose the one from ACM

b. Update S3 Bucket Policy (automatically done with OAC)

### ✅ 5. Setup Route 53 DNS
a. Go to Route 53 > Hosted Zones
  - Create a Hosted Zone (if not already exists) for `your-domain-name`

b. Create Record
  - Type: A or AAAA
  - Alias: Yes
  - Alias Target: Choose your CloudFront distribution
  - Save

### ✅ 6. Setup Firebase
a. Create a Firebase Project
  - Go to [Firebase Console](https://console.firebase.google.com/).
  - Click `Add project`.
  - Enter a project name → click `Continue`.
  - (Optional) Enable Google Analytics → click `Create project`.
  - Wait for it to be ready, then click `Continue`.

b. Register Your App in Firebase
  - Depending on your app platform, pick the appropriate setup:
  🔵 For Web App:
  - In your Firebase project dashboard, click `</> Web` icon to register a web app.
  - Enter an app nickname → click `Register app`.
  - Firebase gives you your config snippet (we’ll use this later).
  - Click `Continue to Console` (you can skip Firebase Hosting for now).

c. Enable Google Authentication
  - In the Firebase console, go to `Authentication`.
  - Click `Get started`.
  - Under the `Sign-in method` tab:
    - Click `Google` provider.
    - Click `Enable`.
    - (Optional) Provide a project support email.
    - Click `Save`.
  - Under the `Setting` tab:
    - Click `Add domain` and add the domain(s) you're using.
    For example:
      - `localhost` (for local development)
      - `127.0.0.1`
      - `your-app.web.app` (if using Firebase Hosting)
      - `your-custom-domain.com` (if using a custom domain)
    - Click Add after entering each one.

### ✅ 7. Test the Setup
  - Open `https://YOUR_DOMAIN_NAME`
  - Confirm:
    - HTTPS is working
    - Content from index.html is loading
    - Firebase OAuth 2.0 is protecting access
    - You can check the logged-in user under the Firebase's Authentication Users tab.

![image](https://github.com/user-attachments/assets/669ed2de-e790-4bba-93a9-5ca10009ca50)

![image](https://github.com/user-attachments/assets/ef779d21-ec4c-41df-bd50-e5d1cd740d8c)


