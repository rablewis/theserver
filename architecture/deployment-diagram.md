# Deployment Diagram

How Take-off is deployed in AWS. The service choices are recorded in
[`decisions.md`](decisions.md).

```mermaid
flowchart LR
    user["User's browser<br/>(Take-off SPA)"]

    subgraph idps["External identity providers"]
        google["Google"]
        facebook["Facebook"]
    end

    subgraph aws["AWS"]
        cf["CloudFront<br/>(HTTPS, SPA fallback)"]
        siteBucket["S3 bucket<br/>(SPA build output, private)"]
        cognito["Cognito User Pool<br/>(managed login, email accounts)"]
        apigw["API Gateway HTTP API<br/>(JWT authorizer)"]
        lambda["API Lambda<br/>(Python)"]
        dataBucket[("S3 data bucket<br/>users/{sub}/…")]
        proc["Processing Lambda<br/>(Python, PDF → SVG)"]
        failQueue[["SQS queue<br/>(failed processing runs)"]]
        failLambda["Failure-handler Lambda<br/>(Python)"]
    end

    user -- "HTTPS: load app" --> cf
    cf -- "Origin Access Control" --> siteBucket
    user -- "sign in (OAuth 2.0 code + PKCE)" --> cognito
    cognito -- "federated sign-in" --> idps
    user -- "HTTPS API calls<br/>+ Cognito JWT" --> apigw
    apigw -. "validate JWT<br/>(JWKS)" .-> cognito
    apigw --> lambda
    lambda -- "read / write under<br/>caller's sub only" --> dataBucket
    lambda -. "issue presigned<br/>upload / download URLs" .-> user
    user -- "PDF upload / SVG download<br/>(presigned URLs)" --> dataBucket
    dataBucket -- "PDF uploaded event" --> proc
    proc -- "write SVGs<br/>and status.json" --> dataBucket
    proc -. "on failure<br/>(crash / timeout)" .-> failQueue
    failQueue --> failLambda
    failLambda -- "write status.json<br/>(failed)" --> dataBucket
```

## Elements

| Element | Responsibility |
| --- | --- |
| CloudFront | Serves the SPA over HTTPS. Maps unknown paths to `index.html` so client-side routes work. |
| S3 site bucket | Holds the SvelteKit `adapter-static` build. Private; readable only by CloudFront. |
| Cognito User Pool | Stores email accounts, federates to Google and Facebook, and issues ID and access tokens to the SPA. |
| API Gateway HTTP API | Public API endpoint. Its JWT authorizer rejects requests without a valid Cognito token. |
| API Lambda (Python) | Manages projects and JSON data, and issues presigned URLs for PDF uploads and image downloads. Takes the user's identity from the verified `sub` claim. |
| S3 data bucket | Private, encrypted, versioned storage. Each user's objects live under `users/{sub}/`. Layout in [`data-design.md`](data-design.md). |
| Processing Lambda (Python) | Triggered when a PDF is uploaded. Generates SVG images and records the PDF's processing status in a `status.json` beside them. Invoked with no automatic retries. |
| SQS queue | On-failure destination for the processing Lambda. Receives a record of each run that crashed or timed out. |
| Failure-handler Lambda (Python) | Reads the queue and marks the affected PDF `failed` in its `status.json`, so it never stays in `processing`. |
