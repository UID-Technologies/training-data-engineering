
#  **Lab 14: Building a Serverless REST API with AWS API Gateway & Lambda**

---

##  **Lab Objective**

By the end of this lab, you will:

- Create a Lambda function using Python or Node.js
- Configure AWS API Gateway to trigger that Lambda
- Test the REST API endpoint from Postman or curl
- Deploy and secure your API

---

##  **Prerequisites**

* AWS Account (free tier is fine)
* IAM user with permissions: **Lambda**, **API Gateway**, and **CloudWatch Logs**
* AWS CLI (optional, for extra steps)
* Postman or curl for testing

---

## **Step 0: Setup Environment**

1. Log in to **AWS Management Console** → go to **Lambda Service**.
2. Ensure you are in region `us-east-1` (or your preferred region).

---

##  **Step 1: Create a Lambda Function**

1. Click **Create function** → select **Author from scratch**
2. Fill details:

   * **Function name:** `HelloWorldLambda`
   * **Runtime:** Choose **Python 3.9** or **Node.js 18.x**
   * **Permissions:** Select *Create a new role with basic Lambda permissions*
3. Click **Create Function**

---

###  Example: Python Code

```python
import json

def lambda_handler(event, context):
    name = event.get("queryStringParameters", {}).get("name", "Guest")
    message = f"Hello, {name}! Welcome to AWS Lambda + API Gateway."
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({"message": message})
    }
```

###  Example: Node.js Code

```javascript
exports.handler = async (event) => {
    const name = event.queryStringParameters?.name || "Guest";
    const message = `Hello, ${name}! Welcome to AWS Lambda + API Gateway.`;
    
    return {
        statusCode: 200,
        headers: {"Content-Type": "application/json"},
        body: JSON.stringify({ message })
    };
};
```

4. Click **Deploy** to save your code.
5. Click **Test** → create a new test event with name `testEvent` and this content:

```json
{
  "queryStringParameters": {"name": "Varun"}
}
```

6. Run the test → You should see:

```json
{
  "statusCode": 200,
  "body": "{\"message\": \"Hello, Varun! Welcome to AWS Lambda + API Gateway.\"}"
}
```

---

##  **Step 2: Create API in API Gateway**

1. Go to **Amazon API Gateway → Create API**
2. Choose **REST API** → “Build” (not HTTP API for now)
3. **API name:** `HelloWorldAPI`
4. Click **Create API**

---

##  **Step 3: Create Resource and Method**

1. Under your API → select **Actions → Create Resource**

   * Resource Name: `hello`
   * Resource Path: `/hello`
   * Check  “Enable API Gateway CORS”
2. Select `/hello` → click **Actions → Create Method** → choose `GET`
3. Integration Type → **Lambda Function**
4. Region: (same as Lambda region)
5. Lambda Function: `HelloWorldLambda`
6. Click **Save** → Confirm permission popup → click **OK**

---

##  **Step 4: Test the API Inside Console**

1. Select the **GET** method under `/hello`
2. Click **Test** tab
3. Add Query String:

   ```
   name=Varun
   ```
4. Click **Test**
5. Response:

   ```json
   {
     "message": "Hello, Varun! Welcome to AWS Lambda + API Gateway."
   }
   ```

---

##  **Step 5: Deploy the API**

1. Click **Actions → Deploy API**
2. Create a **New Stage** → Name: `dev`
3. Click **Deploy**

You’ll now get an **Invoke URL**, e.g.

```
https://abcd1234.execute-api.us-east-1.amazonaws.com/dev/hello
```

---

##  **Step 6: Test from Postman or curl**

### Using curl:

```bash
curl "https://abcd1234.execute-api.us-east-1.amazonaws.com/dev/hello?name=Varun"
```

### Output:

```json
{
  "message": "Hello, Varun! Welcome to AWS Lambda + API Gateway."
}
```

---

##  **Step 7: (Optional) Add API Key Security**

1. In API Gateway → go to **API Keys → Create API Key**
2. Name: `HelloWorldKey` → Auto-generate → Save
3. Go to **Usage Plans → Create**

   * Name: `BasicPlan`
   * Add the API stage (`HelloWorldAPI / dev`)
   * Link the API Key (`HelloWorldKey`)
4. Deploy again → Now you can require the API key in your requests:

   ```bash
   curl -H "x-api-key: <YourKey>" "https://abcd1234.execute-api.us-east-1.amazonaws.com/dev/hello"
   ```

---

##  **Step 8: Cleanup Resources**

After testing, delete:

* The **API** from API Gateway
* The **Lambda Function**
* The **IAM Role** (optional)

---

##  **Outcome**

- You created and deployed a **fully serverless REST API**
- It’s backed by **AWS Lambda** (compute) and **API Gateway** (exposure layer)
- You tested using console, curl, and Postman
- You understand how to add **CORS**, **API keys**, and **usage plans**

