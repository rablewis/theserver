# Take-off: Development Plan (Server Side)

The order in which the server side will be built. Each stage lists the Python files written
or changed, the AWS resources created or configured, and the tests that show the stage is
done. It implements [`high-level-requirements.md`](high-level-requirements.md) and the
decisions in [`architecture/decisions.md`](architecture/decisions.md).

Each stage is deployed to **dev** and its tests pass there before the next stage starts.
The **test** and **prod** environments are set up in Stage 13 (AD-14: all three live in one
AWS account).

Decisions that must still be made, and the stage each one blocks, are listed in
[Decisions still needed](#decisions-still-needed).

## Assumptions

- **PDF rendering:** Poppler `pdftocairo -svg` (P-2 option 1), called from the Python processing
  Lambda, which runs as a container image. Stage 1 confirms or replaces this choice. P-2
  options 2 and 3 are Java and Rust, so choosing either would change AD-4 (Python for Lambda
  code) and Stages 9–10.
- **Region:** `eu-west-2` (London) for every environment (NF-6, AD-15). Set in
  `infra/takeoff_infra/config.py` and checked by tests.
- **Domain:** until a custom domain is chosen, each environment uses its CloudFront default
  domain (`https://dxxxx.cloudfront.net`) and its Cognito managed-login prefix domain.
- **Front end:** the SPA lives in another repository. This plan deploys a placeholder page
  and a script that uploads a real SPA build.
- **Runtime:** Python 3.13 on arm64 for all Lambdas (P-4).

## Repository layout

```text
pyproject.toml                    # dependencies, ruff, pytest settings
infra/
  app.py                          # CDK app: builds the stacks for one environment (-c env=dev|test|prod)
  cdk.json
  takeoff_infra/
    config.py                     # per-environment settings
    stacks/
      site.py                     # S3 site bucket + CloudFront
      data.py                     # S3 data bucket
      auth.py                     # Cognito User Pool, IdPs, app clients
      api.py                      # HTTP API, JWT authorizer, API Lambda
      processing.py               # processing Lambda, SQS queue, failure-handler Lambda
      monitoring.py               # alarms, SNS topic
lambdas/
  common/                         # shared by every Lambda: S3 key builders, IDs, status model
  api/                            # API Lambda
  processing/                     # processing Lambda + Dockerfile
  failure_handler/                # failure-handler Lambda
scripts/                          # operator and developer scripts
tests/
  conftest.py                     # --env option, stack outputs, test users
  infra/                          # CDK template assertions (no AWS needed)
  unit/                           # Lambda code tests with moto / botocore Stubber (no AWS needed)
  integration/                    # tests against a deployed environment
  fixtures/pdfs/                  # generated test PDFs
```

## How testing works

There are three layers of automated tests, all in Python with **pytest**:

| Layer | What it checks | Runs against | Tools |
| --- | --- | --- | --- |
| **Infra** | The synthesised CloudFormation template for each stack and each environment has the right resources and settings (encryption, public access blocks, IAM scope, names). | Nothing deployed | `aws_cdk.assertions` |
| **Unit** | Lambda code logic, including error paths that are hard to cause on purpose in AWS. | Nothing deployed | `pytest`, `moto`, `botocore.stub.Stubber` |
| **Integration** | Each deployed resource really behaves as required. | A deployed environment: `pytest tests/integration --env dev` | `boto3`, `requests` |

Rules:

- **Every deployed resource has at least one integration test.** `tests/infra/test_resource_coverage.py`
  synthesises every stack and fails if any resource in the template is missing from
  `tests/resource_coverage.py`. That file maps each resource to the integration test that
  covers it. CDK helper resources (such as the auto-delete custom resource) are mapped to
  their parent resource's test.
- Integration tests read resource names and URLs from the CloudFormation **stack outputs**,
  never from hard-coded values.
- Test users are created by a fixture (`admin_create_user`, named `test-<uuid>@example.com`)
  and deleted along with their S3 data after the test session.
- Tests sign in through a **test-only app client** with `ADMIN_USER_PASSWORD_AUTH`. That
  client exists in dev and test only.
- `conftest.py` **refuses to run the integration suite with `--env prod`**. Prod has its own
  non-destructive smoke tests (`tests/smoke/`, Stage 13).
- **UK residency (NF-6)** is checked at every stage: an infra test fails if any stack is not
  pinned to `eu-west-2`, or if any template names another region (in an ARN, a replication
  rule or a cross-region reference), apart from the global services allowed by DN-1.
- Every stage also runs `ruff check`, `pytest tests/infra tests/unit`, and
  `cdk synth -c env=<each env>` before deploying.

---

## Stage 0: Project set-up and account bootstrap

**Goal:** a working toolchain and an AWS account ready for CDK deployments (NF-3, NF-5).

**Python files**

- `pyproject.toml`: `aws-cdk-lib`, `constructs`, `boto3`, `pytest`, `moto`, `requests`, `ruff`.
- `infra/app.py`: reads `-c env=…`, refuses unknown names, sets stack names to `Takeoff-<env>-<Stack>`.
- `infra/takeoff_infra/config.py`: an `EnvConfig` dataclass per environment: region, name
  prefix (`takeoff-dev`), removal policy (dev/test destroy, prod retain), allowed CORS origins,
  whether the test app client is created.
- `tests/conftest.py`: `--env` option, prod guard, stack-output fixture.
- `tests/resource_coverage.py`, `tests/infra/test_resource_coverage.py`.
- `scripts/stack_outputs.py`: prints the outputs of every stack for one environment (used by
  scripts and tests).

**AWS resources**

- CDK bootstrap stack (`CDKToolkit`) in `eu-west-2` only, once for the shared account.
- An IAM user or SSO role for deployment. Developers use named AWS CLI profiles.

**Automated tests**

- Unit: `config.py` gives three environments with different name prefixes, all in `eu-west-2`; prod retains data;
  only dev and test enable the test app client; unknown environment names are rejected.
- Integration: the `CDKToolkit` stack exists in `eu-west-2` and is `CREATE_COMPLETE` or
  `UPDATE_COMPLETE`.

**Manual / sanity checks**

- `aws sts get-caller-identity` shows the expected account.
- `cdk synth -c env=dev` runs cleanly (no stacks yet).

---

## Stage 1: PDF-to-SVG rendering spike (local only)

**Goal:** settle P-2 and answer the open question on average SVG size before the processing
code is built. Nothing is deployed.

**Python files**

- `scripts/make_test_pdfs.py`: generates the fixtures in `tests/fixtures/pdfs/`: 1, 4, 20 and
  21 pages, one close to 20 MB, a corrupt file, a non-PDF renamed `.pdf`, and a
  password-protected PDF. Real sample drawings from the owner are added beside them.
- `lambdas/processing/render.py`: `render_page(pdf_path, page, out_path)` that wraps
  `pdftocairo -svg -f n -l n`, and `count_pages(pdf_path)`.
- `scripts/spike_render.py`: renders every fixture and prints time per page, SVG size per page,
  and peak memory.

**AWS resources:** none.

**Automated tests**

- Unit (`tests/unit/test_render.py`, needs Poppler installed or the container image): page
  count is correct; each page gives a well-formed SVG; corrupt and non-PDF files raise a clear
  `InvalidPdf` error.

**Manual / sanity checks**

- Visually compare SVG output with the original PDFs for the real sample drawings.
- Record the results (quality, time per page, average SVG size) in `decisions.md`, and move
  P-2 to a confirmed decision or to the fallback.

---

## Stage 2: SPA hosting

**Goal:** HR-1, HR-2.

**Python files**

- `infra/takeoff_infra/stacks/site.py`: private, encrypted site bucket with all public access
  blocked; CloudFront distribution with Origin Access Control, HTTPS only, and 403/404 mapped
  to `/index.html` with status 200; outputs `SiteBucketName`, `DistributionId`, `SiteUrl`.
- `scripts/upload_site.py`: uploads a SvelteKit build folder to the site bucket (long cache
  for hashed assets, no cache for `index.html`), writes `config.json` (API URL, Cognito
  details from stack outputs), and invalidates CloudFront.
- `site_placeholder/index.html`: a placeholder page used until the real SPA is deployed.

**AWS resources**

- S3 site bucket (`eu-west-2`), CloudFront distribution, Origin Access Control, bucket policy
  allowing only that distribution.
- CloudFront is a global service: it caches the public SPA files at edge locations outside
  the UK. It never handles user data, because API calls and file transfers go straight to
  `eu-west-2`. Whether this is acceptable under NF-6 is **DN-1**, which must be decided
  before this stage.

**Automated tests**

- Infra: bucket blocks public access and is encrypted; the distribution redirects HTTP to
  HTTPS; error responses map to `/index.html`; the bucket policy allows only the
  distribution's service principal.
- Integration (`tests/integration/test_site.py`):
  - `GET SiteUrl/` returns 200 and the placeholder HTML.
  - `GET SiteUrl/some/deep/link` returns 200 and the same HTML (SPA fallback).
  - `http://` is redirected to `https://`.
  - A direct request to the bucket's S3 URL is denied (403).

**Manual / sanity checks**

- Open the site URL in a browser; refresh on a deep link.

---

## Stage 3: Data bucket

**Goal:** durable, encrypted, versioned storage (DA-1, NF-1, AD-3, AD-12).

**Python files**

- `infra/takeoff_infra/stacks/data.py`: data bucket with SSE-S3 encryption, versioning, all
  public access blocked, a bucket policy that denies non-TLS requests, a lifecycle rule
  removing noncurrent versions after 30 days, and CORS allowing the site origin (and
  `http://localhost:5173` in dev only) for presigned POST and GET.
- `lambdas/common/keys.py`: builders for every S3 key in `data-design.md` (`projects_json(sub)`,
  `pdf_key(sub, project_id, file_id)`, …), with checks on ID formats.
- `lambdas/common/ids.py`: ULID generation and validation.

**AWS resources**

- S3 data bucket (`takeoff-<env>-data-<account>`) with its bucket policy, lifecycle rule and
  CORS rules.

**Automated tests**

- Infra: encryption, versioning, public access block, TLS-only policy, lifecycle rule (30 days
  noncurrent), CORS origins differ by environment.
- Unit: key builders produce exactly the layout in `data-design.md` and reject IDs containing
  `/`, `..` or other unexpected characters.
- Integration (`tests/integration/test_data_bucket.py`):
  - `get_bucket_versioning`, `get_bucket_encryption`, `get_public_access_block` and
    `get_bucket_lifecycle_configuration` match the design.
  - A request over plain HTTP is refused.
  - Writing an object twice keeps two versions.
  - `get_bucket_location` is `eu-west-2` and the bucket has no replication configuration (NF-6).

**Manual / sanity checks**

- Check the bucket in the S3 console shows "Bucket and objects not public".

---

## Stage 4: Authentication

**Goal:** AU-1 to AU-6 (AD-5).

**Python files**

- `infra/takeoff_infra/stacks/auth.py`: User Pool with email sign-in, email verification and
  password reset; managed-login domain `takeoff-<env>`; Google and Facebook identity providers
  with secrets read from Secrets Manager; SPA app client (authorization code + PKCE, no
  secret, callback and sign-out URLs from the site URL, plus localhost in dev); a test-only
  app client with `ADMIN_USER_PASSWORD_AUTH` (dev and test only); no account linking.
  Outputs `UserPoolId`, `SpaClientId`, `TestClientId`, `LoginDomain`.
- `scripts/set_idp_secrets.py`: stores the Google and Facebook client ID and secret for an
  environment in Secrets Manager (`takeoff/<env>/google`, `takeoff/<env>/facebook`). Run
  once per environment before deploying.
- `tests/conftest.py`: adds the `test_user` and `second_test_user` fixtures that create users,
  sign them in through the test client, and delete them afterwards.

**AWS resources**

- Cognito User Pool, managed-login domain, Google and Facebook identity providers, two app
  clients (one in prod), Secrets Manager secrets for each identity provider.
- Google OAuth client and Facebook app (outside AWS), each with the environment's Cognito
  redirect URL.

**Automated tests**

- Infra: the SPA client has no secret and allows only the code grant; callback URLs are
  correct per environment; the test client exists only in dev and test; Google and Facebook
  providers are attached to the SPA client.
- Integration (`tests/integration/test_auth.py`):
  - The test fixture can create a user and get ID and access tokens.
  - Tokens verify against the pool's JWKS and contain `sub`.
  - `/oauth2/authorize?identity_provider=Google…` redirects to `accounts.google.com`;
    the Facebook equivalent redirects to `facebook.com`.
  - The managed login page returns 200.
  - `describe_user_pool` shows email verification enabled.

**Manual / sanity checks**

- In a browser: sign up with email, receive the verification code, sign in, reset the
  password, sign out.
- Sign in with Google and with Facebook.
- Sign in with Google and with an email account that use the same address, and confirm that
  two separate users exist (AU-6).

---

## Stage 5: API skeleton with authentication

**Goal:** an authenticated API in place before any data features (AP-3, AD-2).

**Python files**

- `infra/takeoff_infra/stacks/api.py`: HTTP API with a JWT authorizer on the User Pool (SPA
  client audience, plus the test client in dev and test); CORS for the site origin; API Lambda
  (arm64, 512 MB) with IAM access only to `users/*` in this environment's data bucket; access
  logging; throttling limits. Output `ApiUrl`.
- `lambdas/api/handler.py`: entry point using AWS Lambda Powertools'
  `APIGatewayHttpResolver`, with structured logging.
- `lambdas/api/auth.py`: `caller_sub(event)` reads `sub` **only** from the authorizer's JWT
  claims.
- `lambdas/api/errors.py`: error responses in one consistent JSON shape.
- `lambdas/api/projects.py`: `GET /v1/projects`, which returns an empty list at this stage.

**AWS resources**

- API Gateway HTTP API, JWT authorizer, `$default` stage, access log group, API Lambda, its
  execution role and log group.

**Automated tests**

- Infra: every route has the JWT authorizer; the Lambda role can reach only the data bucket's
  `users/*` keys in this environment (no `*` resources, no other environment's bucket).
- Unit: `caller_sub` uses the JWT claims and ignores any `sub` in headers, query or body.
- Integration (`tests/integration/test_api_auth.py`):
  - No token returns 401; a malformed token returns 401; an expired token returns 401.
  - A token from another environment's User Pool returns 401.
  - A valid token returns 200 and `{"projects": []}`.
  - A CORS preflight from the site origin succeeds; one from another origin gets no
    `Access-Control-Allow-Origin` header.

**Manual / sanity checks**

- `curl $ApiUrl/v1/projects` returns 401.

---

## Stage 6: Projects

**Goal:** DA-3, DA-5, DA-2, AD-6.

**Python files**

- `lambdas/api/project_store.py`: reads and writes `projects.json` with `If-Match` on the ETag
  and retries on HTTP 412; creates the file on first use (`If-None-Match: *`).
- `lambdas/api/projects.py`: `GET`/`POST /v1/projects`, `GET`/`PATCH`/`DELETE /v1/projects/{id}`.
  Delete removes the list entry first, then every object under the project folder.
  `GET /v1/projects/{id}` lists the folder and reads `status.json` files (empty at this stage).
- `lambdas/api/validation.py`: project name rules (length, characters).

**AWS resources:** API routes added; no new resources.

**Automated tests**

- Unit:
  - Create, rename, delete and list with moto.
  - The 412 retry path, using `Stubber` to return 412 once.
  - Unknown project ID gives 404; invalid names give 400.
  - Delete removes every object under the prefix.
- Integration (`tests/integration/test_projects.py`):
  - Full lifecycle: create, list, rename, get, delete, list.
  - 10 parallel creates for one user all appear in the list (conditional writes work).
  - **Isolation (DA-2):** `second_test_user` gets 404 on `GET`, `PATCH` and `DELETE` of the first
    user's project, and the first user's list is unchanged.
  - Objects land under `users/{sub}/` for the caller's real `sub`.

**Manual / sanity checks**

- Inspect `projects.json` in the S3 console after the tests and check it matches `data-design.md`.

---

## Stage 7: JSON data files

**Goal:** AP-1, AP-2, DA-4 (JSON files), AD-11.

**Python files**

- `lambdas/api/data_files.py`: `GET` and `PUT /v1/projects/{id}/data/{name}`; rejects bodies over
  1 MB with 413; checks the name is allowed and the body is valid JSON matching the schema
  for that name.
- `lambdas/api/schemas/`: one JSON Schema per data file name. The schemas come from the
  front-end team; until then, the check is "valid JSON object".

**AWS resources:** API routes added.

**Automated tests**

- Unit: valid save and load; 1 MB + 1 byte gives 413; invalid JSON and schema failures give
  400; an unknown name gives 400; a path-traversal name (`../x`) gives 400.
- Integration (`tests/integration/test_data_files.py`):
  - Save and reload a 50 KB file; the content is identical.
  - Saving twice keeps the previous S3 version (DA-1, AD-12).
  - Oversized body gives 413.
  - The second test user cannot read or write the first user's data (404).

**Manual / sanity checks:** none beyond the automated tests.

---

## Stage 8: PDF upload

**Goal:** DA-4 (PDFs), DA-6 size limit, AD-8.

**Python files**

- `lambdas/api/pdfs.py`: `POST /v1/projects/{id}/pdfs` issues a presigned POST for
  `pdf/{fileId}.pdf` with conditions: exact key, `Content-Type: application/pdf`,
  `content-length-range` up to 20 MB, `x-amz-meta-filename`. Refuses a third PDF (a project
  holds at most 2).
- `lambdas/api/projects.py`: `GET /v1/projects/{id}` now lists PDFs and their original file names.

**AWS resources:** API route added; the API Lambda role gets `s3:PutObject` on `users/*/projects/*/pdf/*`.

**Automated tests**

- Unit: presigned POST conditions are exactly as listed; a third PDF is refused; a project
  belonging to someone else gives 404.
- Integration (`tests/integration/test_pdf_upload.py`):
  - Upload the 4-page fixture with the presigned POST; it appears in `GET /v1/projects/{id}`
    with its original name.
  - A file over 20 MB is rejected by S3.
  - Changing the key, or the content type, in the form is rejected by S3.
  - The presigned POST no longer works after it expires.

**Manual / sanity checks**

- Upload a PDF from the browser on `localhost` against dev to confirm CORS for presigned POST.

---

## Stage 9: Image generation

**Goal:** DA-7, DA-6 page limit, NF-4, AD-7, P-4.

**Python files**

- `lambdas/processing/Dockerfile`: AWS Python 3.13 arm64 base image plus Poppler.
- `lambdas/processing/handler.py`: handles the S3 `ObjectCreated` event; checks the key matches
  `users/{sub}/projects/{id}/pdf/{fileId}.pdf` (ignores anything else); writes
  `status: processing`; checks page count (over 20 pages or invalid PDF → `failed`, delete
  the PDF); renders one page at a time; stops and cleans up when the 2-minute budget runs out;
  writes `status: ready` with `pages`. Unexpected errors (for example a missing object) are
  **not** caught, so they reach the failure path in Stage 10.
- `lambdas/common/status.py`: the `status.json` model and writer.
- `infra/takeoff_infra/stacks/processing.py`: processing Lambda from the container image
  (arm64, 2 GB, 3-minute timeout, `retry_attempts=0`); S3 notification for keys ending in `.pdf`
  under `users/`; IAM access only to read `pdf/` and write `images/` in this environment's bucket.

**AWS resources**

- ECR image asset, processing Lambda and log group, S3 event notification, its IAM role.

**Automated tests**

- Infra: `retry_attempts` is 0; timeout 180 s; memory 2048 MB; arm64; the notification
  filter is correct; IAM is scoped to this environment's bucket.
- Unit (with moto and a fake clock):
  - A 4-page PDF writes 4 SVGs and `ready` with `pages: 4`.
  - 21 pages, corrupt or non-PDF: `failed`, PDF deleted, no SVGs.
  - Budget exhausted after page 2: SVGs deleted, `failed` with "took too long".
  - Keys outside the expected pattern are ignored.
- Integration (`tests/integration/test_processing.py`):
  - Upload the 4-page fixture; status reaches `ready` with 4 pages and 4 SVGs exist.
  - Upload the 20-page fixture; `ready` **within 120 seconds** of upload (NF-4).
  - Upload the 21-page and corrupt fixtures; status reaches `failed` and the PDF is gone.

**Manual / sanity checks**

- Open a few generated SVGs from the S3 console and compare with the PDF.
- Check the processing Lambda's logs show one line per page with timing.

---

## Stage 10: Processing failure handling

**Goal:** a PDF never stays in `processing` (AD-10).

**Python files**

- `lambdas/failure_handler/handler.py`: reads on-failure destination records from SQS,
  takes the S3 key from `requestPayload`, writes `status: failed` with an error message, and
  ignores keys that already have `ready` or `failed`.
- `infra/takeoff_infra/stacks/processing.py`: SQS queue (encrypted) as the processing
  Lambda's on-failure destination; failure-handler Lambda (256 MB) with an SQS event source;
  a dead-letter queue for the failure queue (after 3 receives).

**AWS resources**

- SQS failure queue, SQS dead-letter queue, failure-handler Lambda, its log group and IAM role,
  event invoke config on the processing Lambda.

**Automated tests**

- Infra: the processing Lambda's on-failure destination is the queue; the failure queue has a DLQ;
  the failure-handler role can write only `images/*/status.json`.
- Unit: a sample destination record gives the right `status.json`; an existing `ready` status
  is not overwritten; a malformed record raises (so it goes to the DLQ).
- Integration (`tests/integration/test_processing_failure.py`):
  - Invoke the processing Lambda **asynchronously** with an event for a PDF key that does not
    exist under a test user's project. The Lambda crashes, and within 2 minutes `status.json`
    shows `failed`.
  - The failure queue and DLQ are empty afterwards.

**Manual / sanity checks**

- Check the failure-handler's log line for the test record.

---

## Stage 11: Image download

**Goal:** DA-8, AD-9.

**Python files**

- `lambdas/api/images.py`: `GET /v1/projects/{id}/images/{fileId}/{n}` returns a presigned GET
  URL valid for 5 minutes, after checking the project is in the caller's list and the
  image exists.

**AWS resources:** API route added; the API Lambda role gets `s3:GetObject` on `users/*/projects/*/images/*`.

**Automated tests**

- Unit: ownership check; missing image gives 404; URL expiry is 300 seconds.
- Integration (`tests/integration/test_image_download.py`):
  - After processing the 4-page fixture, each page URL downloads a valid SVG.
  - The second test user gets 404 for the first user's images.
  - A direct, unsigned S3 request for the image is denied.
  - A URL used after expiry is refused (test with a short expiry setting in dev).

**Manual / sanity checks**

- In the browser on `localhost`, fetch an image URL from the SPA to confirm CORS for GET.

---

## Stage 12: Monitoring, limits and tuning

**Goal:** know when something goes wrong, and keep the shared account's environments from
affecting each other (AD-14).

**Python files**

- `infra/takeoff_infra/stacks/monitoring.py`: SNS alarm topic with an email subscription; alarms
  for API 5xx rate, Lambda errors and throttles, processing duration close to 2 minutes,
  messages in the failure queue's DLQ; log retention on every log group (for example
  30 days in dev and test, 1 year in prod); reserved concurrency for dev and test Lambdas
  so they can't use up prod's capacity; an AWS Budgets alert.
- `scripts/power_tune.py`: deploys and runs AWS Lambda Power Tuning against the processing
  Lambda using the fixture PDFs, and prints the recommended memory size (P-4).

**AWS resources**

- SNS topic and subscription, CloudWatch alarms, log retention settings, reserved concurrency
  (dev and test), budget alert. Power Tuning state machine (temporary, dev only).

**Automated tests**

- Infra: every log group has a retention period; every Lambda has an errors alarm; dev and
  test Lambdas have reserved concurrency, prod's do not.
- Integration (`tests/integration/test_monitoring.py`):
  - All expected alarms exist and are in `OK` or `INSUFFICIENT_DATA`.
  - `set_alarm_state` on one alarm to `ALARM` delivers a message to the SNS topic (checked
    via a test SQS subscription created and removed by the test).

**Manual / sanity checks**

- Confirm the alarm email subscription.
- Run `scripts/power_tune.py`, update the memory settings, and record the result in
  `decisions.md` (confirms P-4).

---

## Stage 13: test and prod environments

**Goal:** NF-5: three separate environments.

**Python files**

- `tests/smoke/test_prod_smoke.py`: non-destructive checks only: site and deep link return
  200; the API returns 401 without a token; the managed login page and JWKS respond; all
  alarms are not in `ALARM`.
- `tests/integration/test_environment_isolation.py`: compares the stack outputs of all
  environments and checks that no bucket, User Pool, API or Lambda is shared, and that each
  Lambda role cannot reach another environment's bucket (IAM policy simulator).
- `scripts/smoke_test.py`: runs the right suite for an environment (integration for dev and
  test, smoke for prod).
- `infra/takeoff_infra/config.py`: final settings for test and prod (retain policies, no test
  client in prod, Cognito email through SES for prod if a domain is available).

**AWS resources**

- Every stack from Stages 2–12 deployed as `Takeoff-test-*` and `Takeoff-prod-*`.
- Identity-provider secrets and Google/Facebook redirect URLs for test and prod.
- SES identity for prod emails (depends on the custom-domain decision); without it, Cognito's
  default email sender has a low daily limit.

**Automated tests**

- Run the full integration suite against **test**.
- Run the smoke suite against **prod**.
- Run `test_environment_isolation.py`.
- Run `tests/integration/test_residency.py` (all environments): every `Takeoff-*` stack is in
  `eu-west-2`; the Resource Groups Tagging API finds no Take-off-tagged resources in any other
  enabled region (except the global services allowed by DN-1); every log group, bucket,
  queue and User Pool is in `eu-west-2`.
- Infra: prod buckets and User Pool have `RETAIN` removal policies; prod has no test app client.

**Manual / sanity checks**

- In prod, sign in with each method (email, Google, Facebook), create a project, upload a
  PDF, see the images, save and reload JSON data, delete the project. Then delete the account.
- Check that the prod buckets and User Pool have deletion protection or `RETAIN` in the console.

---

## Decisions still needed

Each decision must be made before the stage shown. Where the plan already assumes an answer,
the assumption is given; confirm it, or change the plan. When a decision is made, record it in
[`architecture/decisions.md`](architecture/decisions.md) and remove it here.

| # | Decision | Needed by | Current assumption / options | Notes |
| --- | --- | --- | --- | --- |
| DN-1 | **How NF-6 (UK residency) applies to AWS global services.** These hold configuration and public files, not user data: **CloudFront** (caches the SPA at edge locations worldwide), **IAM**, **AWS Budgets**, and, with a custom domain, **Route 53** and the **ACM certificates** that CloudFront and Cognito custom domains must have in **`us-east-1`** | Stage 0 (before Stage 2) | Allow these as named exceptions, because no user data passes through or is stored in them. | The alternative to CloudFront is serving the SPA from a UK-only service (for example API Gateway or a load balancer in front of the S3 bucket). That costs more, is slower for users, and is more work. A `us-east-1` certificate also needs CDK bootstrapped in `us-east-1`. Once decided, write the exceptions into NF-6. |
| DN-2 | **Deployment access** to the shared account: who may deploy to each environment, and how (IAM Identity Center, roles per environment) | Stage 0 | Named CLI profiles for developers. | With one account (AD-14), stopping a dev deploy from touching prod relies on IAM and process, not account boundaries. |
| DN-3 | **PDF-to-SVG library** (P-2) | End of Stage 1 (before Stage 9) | Poppler `pdftocairo -svg`. | Poppler is GPL; running it as a separate program inside a hosted-only service is the basis for accepting it. Confirm this with whoever owns licensing. Java or Rust options would change AD-4. |
| DN-4 | **Custom domain** for the SPA, API and sign-in pages | Stage 2 (can be added later, at the cost of reconfiguration) | CloudFront and Cognito default domains. | Affects CORS origins, Cognito callback URLs, Google/Facebook redirect URLs, and SES (DN-6). Needs certificates in `us-east-1` (DN-1). Changing it later means updating all of those in every environment. |
| DN-5 | **ID format** for projects and files | Stage 3 | ULID. | `data-design.md` says "ULID/UUID". ULIDs sort by creation time, which helps when listing. |
| DN-6 | **Sign-up and reset email sender** | Stage 4 (prod: Stage 13) | Cognito default sender in dev and test; Amazon SES in prod. | Cognito's default sender has a low daily limit and can't use your own From address. SES needs a verified domain (DN-4). |
| DN-7 | **Password policy and MFA** for email accounts | Stage 4 | Cognito defaults (8+ characters, mixed case, digits, symbols); no MFA. | Not covered by the requirements. |
| DN-8 | **Google and Facebook apps**: one per environment, or one shared with several redirect URLs | Stage 4 | One per environment. | Separate apps keep prod credentials out of dev and test. Facebook needs app review before the general public can use prod. |
| DN-9 | **API throttling limits** (requests per second and burst) | Stage 5 | Around 50 rps steady, 100 burst per environment. | Expected peak is about 4 saves per second, plus polling and image requests. |
| DN-10 | **Project name rules** and **maximum projects per user** | Stage 6 | 1–100 characters, any printable characters; no project limit. | A limit keeps `projects.json` small and fast to rewrite. |
| DN-11 | **JSON data file names and schemas** | Stage 7 | Accept any JSON object up to 1 MB until schemas are supplied. | Must come from the front-end team. Without schemas the API checks only size and JSON syntax. |
| DN-12 | **Average SVG size** (open question) | End of Stage 1 | Measured in Stage 1. | Decides SVG storage and download costs, and whether AD-13 browser caching is enough. |
| DN-13 | **Lambda memory sizes** (P-4) | Stage 12 | 2 GB processing, 512 MB API, 256 MB failure handler. | Set from Power Tuning results. |
| DN-14 | **Reserved concurrency** for dev and test | Stage 12 | e.g. 10 per Lambda in dev and test. | Protects prod in the shared account (AD-14). Too low a value makes integration tests slow or flaky. |
| DN-15 | **Cost targets**, **budget alert amount** and **alarm recipients** | Stage 12 | None yet. | Cost targets are an open question in the requirements. |
| DN-16 | **Log retention** periods | Stage 12 | 30 days in dev and test, 1 year in prod. | May be driven by compliance needs (DN-18). |
| DN-17 | **Availability targets, backup and retention** beyond 30-day S3 versioning | Stage 13 | S3 versioning only; no cross-region copy; no Cognito backup. | Cognito user pools can't be restored from a backup, so losing the prod pool means users must sign up again. `RETAIN` and deletion protection lower that risk. |
| DN-18 | **Compliance**: GDPR data export, account deletion, privacy notice | Before prod launch | Not built. | Account deletion would delete the Cognito user and everything under `users/{sub}/`. That's a new API route and stage. |
| DN-19 | **Release process**: manual `cdk deploy` or a CI/CD pipeline, and how changes are promoted dev → test → prod | Stage 13 | Manual deploys from a developer machine. | A pipeline would run the infra and unit tests on every change and the integration suite before promotion. |

---

## Not covered by this plan

- A CI/CD pipeline. Deployments are run with `cdk deploy` from a developer machine.
- A custom domain, data-residency rules, GDPR export and account deletion, and backups
  beyond S3 versioning. These depend on open questions in `high-level-requirements.md`.
