# How to Run: Cloud Based Intelligent Portfolio 🚀

A complete, step-by-step guide to deploying this serverless portfolio on AWS. 
Includes all the errors we hit and exactly how to fix them — so you don't have to.

---

## 🗂️ Architecture Overview

```
Website Visitor → AWS Amplify (Website)
                        ↓
                 API Gateway (Endpoint)
                        ↓
              AWS Lambda (Python Brain)
               /          |          \
     DynamoDB         AWS SES      Comprehend
   (Save Data)    (Send Email)    (AI Analysis)
```

---

## PART 1: STEP-BY-STEP DEPLOYMENT

---

### ✅ Step 1 — Fork & Configure the GitHub Repository

1. Fork the project repository to your own GitHub account.
2. Make it **Private** (Settings → Change Visibility).
3. Clone it locally:
   ```
   git clone https://github.com/YOUR_USERNAME/Cloud-based-Intelligent-Portfolio.git
   ```
4. Keep this open — you will push changes to it throughout the project.

---

### ✅ Step 2 — Host Website with AWS Amplify

1. Go to **AWS Amplify** in the AWS Console.
2. Click **New App → Host Web App**.
3. Click **GitHub** and authorize AWS to access your repositories.
4. Select your forked repository and choose the `main` branch.
5. Leave all build settings as default and click **Save and Deploy**.
6. Wait for the deployment to complete. Note the live URL (e.g., `https://main.xxxx.amplifyapp.com`).

> **CI/CD is automatic**: Any future `git push` will trigger a redeployment.

---

### ✅ Step 3 — Create a DynamoDB Table

1. Go to **AWS DynamoDB** in the console.
2. Click **Create table**.
3. Fill in:
   - **Table name**: `my-portfolio-data-table` *(exact, case-sensitive)*
   - **Partition key (Primary Key)**: `ResponsesID` → Select type **String**
4. Leave all other settings on default.
5. Click **Create table**.
6. *(Optional)* Click **Explore table items → Create item** and add a test record.

---

### ✅ Step 4 — Create an IAM Role for Lambda

1. Go to **AWS IAM → Roles → Create role**.
2. Trusted entity: **AWS service → Lambda**. Click Next.
3. Attach these **managed policies**:
   - `AWSLambdaBasicExecutionRole`
   - `AmazonSESFullAccess`
   - `ComprehendFullAccess`
4. Name the role: `Portfolio-Lambda-Role`. Click **Create role**.
5. Open the newly created role and click **Add permissions → Create inline policy**.
6. Switch to **JSON** editor and paste:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": "dynamodb:PutItem",
         "Resource": "arn:aws:dynamodb:ap-south-1:*:table/my-portfolio-data-table"
       }
     ]
   }
   ```
7. Name the policy `DynamoDBPutItem` and click **Create policy**.

---

### ✅ Step 5 — Create the Lambda Function

1. Go to **AWS Lambda → Create function**.
2. Select **Author from scratch**.
3. Fill in:
   - **Function name**: `portfolio-backend`
   - **Runtime**: Python 3.10
   - **Execution role**: Use existing role → `Portfolio-Lambda-Role`
4. Click **Create function**.

**Add the Numpy Layer:**
1. Scroll down to **Layers → Add a layer**.
2. Select **Specify an ARN** and enter:
   ```
   arn:aws:lambda:ap-south-1:770693421928:layer:Klayers-p310-numpy:17
   ```
3. Click **Verify**, then **Add**.

**Increase Timeout & Memory (CRITICAL):**
1. Click the **Configuration** tab → **General configuration** → **Edit**.
2. Set **Memory** to `512 MB`.
3. Set **Timeout** to `30 seconds`.
4. Click **Save**.

---

### ✅ Step 6 — Set Up API Gateway

1. In your Lambda function, click **Add trigger**.
2. Select **API Gateway**.
3. Choose **Create a new API → HTTP API**.
4. Set **Security** to **Open**.
5. Click **Add**.
6. Note the **API endpoint URL** — you will need it in the next step.

**Enable CORS in API Gateway:**
1. Go to **API Gateway console** and click your new API.
2. In the left menu, click **CORS → Configure**.
3. Set:
   - **Access-Control-Allow-Origin**: `*`
   - **Access-Control-Allow-Methods**: `POST`, `OPTIONS`
   - **Access-Control-Allow-Headers**: `content-type`
4. Click **Save**.

---

### ✅ Step 7 — Connect the Website to the API

1. Open `assets/mail/contact_me.js` in your local repository.
2. Find line 27 and replace the placeholder URL with your API Gateway endpoint:
   ```javascript
   url: "https://YOUR_API_ID.execute-api.ap-south-1.amazonaws.com/default/portfolio-backend",
   ```
3. Save, commit, and push:
   ```bash
   git add .
   git commit -m "Update API Gateway endpoint"
   git push
   ```
4. Wait for Amplify to redeploy (~2 minutes).

---

### ✅ Step 8 — Verify Your Email in SES

1. Go to **Amazon SES → Identities → Create identity**.
2. Select **Email address**.
3. Enter your personal email and click **Create identity**.
4. Check your inbox and click the **verification link**.
5. *(For project submission)* Repeat for `edsa.predicts@explore-ai.net`.

---

### ✅ Step 9 — Deploy the Final Lambda Code (AI + Email + Database)

Paste this into your Lambda's `lambda_function.py` and click **Deploy**:

```python
import boto3
import json
import base64
import random

def lambda_handler(event, context):
    try:
        body = event.get('body', '{}')
        if event.get('isBase64Encoded', False):
            dec_dict = json.loads(base64.b64decode(body))
        else:
            dec_dict = json.loads(body)
            
        visitor_name = dec_dict.get('name', 'Guest')
        visitor_email = dec_dict.get('email', '')
        message_content = dec_dict.get('message', '')
        rid = str(random.randint(1, 1000000000))
        
        dynamodb = boto3.resource('dynamodb')
        ses = boto3.client('ses')
        comprehend = boto3.client('comprehend')
        table = dynamodb.Table('my-portfolio-data-table')
        
        db_response = table.put_item(Item={
            'ResponsesID': rid,
            'Name': visitor_name,
            'Email': visitor_email,
            'Cell': str(dec_dict.get('phone', 'N/A')),
            'Message': message_content
        })

        # AI with Fail-Safe (in case AWS subscription is pending)
        sentiment = 'NEUTRAL'
        phrases = []
        try:
            sentiment_res = comprehend.detect_sentiment(Text=message_content, LanguageCode='en')
            sentiment = sentiment_res['Sentiment']
            key_phrases_res = comprehend.detect_key_phrases(Text=message_content, LanguageCode='en')
            phrases = [p['Text'].lower() for p in key_phrases_res['KeyPhrases']]
        except Exception as ai_err:
            print(f"AI Skipped (Subscription issue): {str(ai_err)}")

        SENDER = 'your-verified-email@example.com'  # <-- UPDATE THIS
        YOUR_NAME = 'Your Name'                     # <-- UPDATE THIS
        
        CV_Text = 'I see that you mentioned my C.V in your message. I am happy to forward you my C.V in response. If you have any other questions or C.V related queries please do get in touch. '
        Project_Text = 'The projects I listed on my site only include those not running in production. I have several other projects that might interest you. '
        Article_Text = 'In your message you mentioned my blog posts and data science articles. I have several other articles published in academic journals. Please do let me know if you are interested - I am happy to forward them to you. '
        Negative_Text = f'I see that you are unhappy in your response. Can we please set up a session to discuss why you are not happy, be it with the website, my personal projects or anything else. \n\nLooking forward to our discussion. \n\nKind Regards, \n\n{YOUR_NAME}'
        Neutral_Text = f'Thank you for your email. Let me know if you need any additional information.\n\nKind Regards, \n\n{YOUR_NAME}'
        Farewell_Text = f'\n\nAgain, Thank you for your email.\n\nIf there is anything else I can assist you with please let me know and I will set up a meeting for us to meet in person.\n\nKind Regards, \n\n{YOUR_NAME}'

        email_body = f"Good day {visitor_name},\n\n"
        if sentiment == 'NEGATIVE':
            email_body += Negative_Text
        elif sentiment == 'NEUTRAL':
            email_body += Neutral_Text
        else:
            matched = False
            phrases_str = ' '.join(phrases)
            if any(w in phrases_str for w in ['cv', 'resume', 'c.v']):
                email_body += CV_Text + "\n\n"; matched = True
            if any(w in phrases_str for w in ['blog', 'article', 'post']):
                email_body += Article_Text + "\n\n"; matched = True
            if any(w in phrases_str for w in ['github', 'git', 'project', 'portfolio']):
                email_body += Project_Text + "\n\n"; matched = True
            if not matched:
                email_body += "Thank you for your very kind words about my work! "
            email_body += Farewell_Text

        ses.send_email(
            Destination={'ToAddresses': [visitor_email]},
            Message={
                'Body': {'Text': {'Charset': "UTF-8", 'Data': email_body}},
                'Subject': {'Charset': "UTF-8", 'Data': f"RE: {visitor_name} - Data Science Enquiry"},
            },
            Source=SENDER,
        )
        return {
            'statusCode': 200,
            'headers': {'Access-Control-Allow-Origin': '*'},
            'body': json.dumps({'message': 'Success!'})
        }
    except Exception as e:
        print(f"General Error: {str(e)}")
        return {
            'statusCode': 500,
            'headers': {'Access-Control-Allow-Origin': '*'},
            'body': json.dumps({'error': str(e)})
        }
```

---

## PART 2: ERRORS WE HIT & EXACT FIXES

---

### ❌ Error 1 — "Mail server not responding"
- **When**: After first submitting the form.
- **Root Cause**: API Gateway was missing CORS headers.
- **How to Diagnose**: Open browser DevTools (F12) → Network tab → Click the failed request → Check Status Code.
- **Fix**: Add CORS to API Gateway (see Step 6 above).

---

### ❌ Error 2 — `Type mismatch for key ResponsesID expected: S actual: N`
- **When**: After CORS was fixed, a 500 error appeared.
- **Root Cause**: DynamoDB table has `ResponsesID` as String (S), but the Lambda was sending an integer (N).
- **How to Diagnose**: AWS CloudWatch Logs → Lambda function → Monitor tab → View CloudWatch logs.
- **Fix**: Change the Lambda code from:
  ```python
  'ResponsesID': random.randint(...)
  ```
  To:
  ```python
  'ResponsesID': str(random.randint(...))
  ```

---

### ❌ Error 3 — `SubscriptionRequiredException` from `DetectSentiment`
- **When**: After integrating the AI (Comprehend) code.
- **Root Cause**: AWS Comprehend requires full account verification before allowing any calls, even on the free tier.
- **Fix**: Wrap the AI calls in a `try-except` block (Fail-Safe). If the AI subscription is blocked, default to `'NEUTRAL'` and continue sending the email normally.

---

### ❌ Error 4 — `{"message": "Internal Server Error"}`
- **When**: After adding the AI Fail-Safe code, the website still showed "mail server not responding".
- **Root Cause**: Lambda default timeout is only 3 seconds. Adding SES email sending + Comprehend calls made the function take longer than 3 seconds, causing a timeout.
- **How to Diagnose**: Browser DevTools (F12) → Network → Response tab → The response says `{"message":"Internal Server Error"}` (generic AWS timeout message, not our custom error).
- **Fix**: Go to Lambda → **Configuration → General configuration → Edit** → Set **Memory to 512 MB** and **Timeout to 30 seconds**.

---

## ✅ Final Verification Checklist

- [ ] Submit form → Website shows **"Your message has been sent"** ✅
- [ ] Go to **DynamoDB → Explore table items** → New row visible ✅
- [ ] Check your email inbox → **Automated "Thank You" reply received** ✅

**Project Status: Fully Functional 🎉**
