
# S3 Multipart Upload: Client-Side and Backend Implementation

This README provides an overview of how the client-side and backend work together to enable efficient, resumable uploads to Amazon S3 using multipart upload.

## Overview
Amazon S3's multipart upload allows large files to be uploaded in smaller parts, improving reliability and performance. This implementation divides responsibilities between the client-side and the backend:

- **Client-Side**: Handles file splitting, part uploads, and progress tracking.
- **Backend**: Interacts with AWS S3 to initiate uploads, generate presigned URLs, track parts, and finalize uploads.

---

## Client-Side Workflow

The client-side application is implemented using HTML, CSS, and JavaScript. 

**Client-side code is found in the client-side folder: "index.html".** 

Its main responsibilities include:

1. **File Selection**:
   - The user selects a file for upload.

2. **Splitting the File**:
   - The file is divided into smaller parts (e.g., 5 MB each).
   - Each part is assigned a sequential part number starting from 1.

3. **Requesting Presigned URLs**:
   - For each part, the client sends a request to the backend to get a presigned URL.

4. **Uploading Parts**:
   - Each part is uploaded directly to S3 using the presigned URL.
   - The response from S3 includes an `ETag` for the uploaded part, which is stored for later.

5. **Tracking Progress**:
   - A progress bar visually updates as parts are uploaded.
   - Upload state (e.g., `UploadId`, part numbers, and `ETag`s) is saved in `localStorage` to enable resuming after interruptions.

6. **Completing the Upload**:
   - Once all parts are uploaded, the client sends a request to the backend with the `UploadId` and part details (part numbers and `ETag`s) to finalize the upload.

7. **Error Handling**:
   - Failed parts are retried automatically.
   - If the connection is interrupted, the client uses `localStorage` and the backend to resume from the last successful part.

---

## Backend Workflow

The backend is implemented using FastAPI and Boto3. It serves as the intermediary between the client and S3. 

**Back-end code is found in the back-end/lambda folder: "lambda_function.py".** 

Upload lambda_function.py as zip. You can just copy and paste the code in the code editor with the lambda' user interface.

Make sure to **Deploy** lambda function if you copy and paste the code or make any changes to the code.

Its main responsibilities include:

1. **Starting Multipart Upload**:
   - The backend handles requests from the client to initiate a multipart upload by calling S3's `CreateMultipartUpload` API.
   - It returns the `UploadId` and object key to the client.

2. **Generating Presigned URLs**:
   - For each part, the client requests a presigned URL from the backend.
   - The backend generates the URL using S3's `GeneratePresignedUrl` API with the appropriate `UploadId` and `PartNumber`.

3. **Listing Uploaded Parts**:
   - If the client resumes after an interruption, it requests the list of uploaded parts.
   - The backend uses S3's `ListParts` API to retrieve already uploaded parts and returns them to the client.

4. **Completing Multipart Upload**:
   - After all parts are uploaded, the client sends the `UploadId`, part numbers, and `ETag`s to the backend.
   - The backend calls S3's `CompleteMultipartUpload` API to finalize the upload.

---

## Error Handling

### Client-Side
- Retries failed parts using stored upload state.
- Alerts the user if an upload cannot proceed.

### Backend
- Validates API requests and ensures the required parameters are present.
- Handles S3 errors (e.g., invalid `UploadId`, missing parts) gracefully.

---

## Configuration
1. **CORS Permission for S3 Bucket**:
   
   Ensure the S3 Bucket's CORS policy allows required methods and headers within S3 Bucket's permissions.
  ```json
  [
    {
        "AllowedHeaders": [
            "*"
        ],
        "AllowedMethods": [
            "GET",
            "PUT",
            "POST",
            "DELETE",
            "HEAD"
        ],
        "AllowedOrigins": [
            "*"
        ],
        "ExposeHeaders": [
            "Content-Type",
            "ETag"
        ],
        "MaxAgeSeconds": 3000
    }
]
  ```
2. **Lambda Function's IAM Policy**:
   
   Create an IAM policy for your lambda function to allow CloudWatch to create log groups. The log groups help tacking down bugs that may occur with your lambda function.
 ```json
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "logs:CreateLogGroup",
            "Resource": "arn:aws:logs:<enter reagon>:<enter account id>:*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": [
                "arn:aws:logs:<enter reagon>:<enter account id>:log-group:/aws/lambda/<enter lambda function's name>:*"
            ]
        }
    ]
}
  ```
3. **Lambda Function's IAM Role**:
   
   Create an IAM role for your lambda function. Include the following IAM policies in your lambda function's role:

- **AmazonS3FullAccess**: AWS managed policy.
- **IAM Policy you created earlier**: Customer managed policy.

4. **Setup Lambda Function's URL**

    Configure your lambda function's URL.

    - Go to your lambda function you created. 
    - Click on the Configuration tag.
    - On the left hand side, click on Function URL.
    - On the right hand side, click on Create function URL button.
    - Configure the following URL settings:
        - **Auth Type:** None
        - **Additional Settings:**
            - **Invoke Mode:** BUFFERED (default)
            - **Allow Origin:** *
            - **Expose headers:** content-type
            - **Allow headers:** content-type
            - **Allow methods:** *
        - Disregard other settings.
        - Click on the Save button.

5. **Create Lambda Layer**

    A lambda layer is required because the lambda function has dependencies.
    
    - Download Git Bash if you do not have it yet.
    - Open Git Bash.
    - Given that you cloned this repository, run the following Git Bash prompt:

        ```
        cd s3_multi_part_upload_python_guide/back-end/lambda_layer
        ```
        ```
        ./create_lambda_layer.sh
        ```
    - Follow the instructions in the prompt.
---

## Security Considerations
1. **Authentication and Authorization**:
   - Use AWS IAM roles and policies to restrict access to the S3 bucket.
   - Implement user authentication on the backend (e.g., JWT).

2. **Presigned URL Expiry**:
   - Set a short expiry time (e.g., 15 minutes) for presigned URLs.

---
## Resumable Upload Workflow Considerations
1. The client tracks upload progress and stores the state (`UploadId`, parts, etc.) in `localStorage`.
2. On reconnecting, the client fetches the list of already uploaded parts from the backend.
3. The client resumes uploading remaining parts.
4. Once all parts are uploaded, the backend finalizes the upload.

## Conclusion
This implementation leverages S3's multipart upload for efficient, reliable, and resumable uploads. By distributing responsibilities between the client-side and backend, it ensures scalability and fault tolerance while keeping the system secure.


