# Take-off: High-Level Requirements (Server Side)

## 1. Context

Take-off is a static single-page application (Svelte 5, SvelteKit 2, `adapter-static`).
It runs entirely in the browser and talks to AWS-hosted services for authentication and
data.

Take-off is a **commercial product** that will **only ever run as a hosted service**. It is
never distributed for customers to install (no desktop or on-premises versions). This
constraint affects which third-party software licences are acceptable.

## 2. Functional requirements

### Hosting

- **HR-1** The Take-off SPA (static HTML, JS, CSS and assets) is hosted in AWS and served
  over HTTPS.
- **HR-2** Client-side routes resolve to the SPA entry point (SPA fallback), so that
  deep links and page refreshes work.

### Authentication

- **AU-1** Users must authenticate before using any part of the app beyond the sign-in page.
- **AU-2** Users can sign in with a **Google** account.
- **AU-3** Users can sign in with a **Facebook** account.
- **AU-4** Users can create a Take-off account with an **email address** and sign in with it.
  This includes email verification and password reset.
- **AU-5** Users can sign out.
- **AU-6** Each account has **exactly one sign-in method**. Accounts are not linked: signing in
  with Google and with an email account that use the same email address gives two separate
  accounts with separate data.

### API

- **AP-1** The SPA saves user data to the server through an API.
- **AP-2** The SPA retrieves the user's previously saved data through the API.
- **AP-3** Every API call is authenticated. Calls without a valid credential are rejected.

### Data

- **DA-1** Data a user enters is stored durably in AWS and is available to them in later
  sessions and on other devices.
- **DA-2** A user can read and change only their own data. No user can reach another
  user's data, whether through the app or by calling the API directly.
- **DA-3** A user's data is organised into **projects**. Each user keeps a list of their
  projects, each with a user-defined name, and the list references the S3 folder containing
  the files belonging to each project.
- **DA-4** Each project contains:
  - one or two **PDF files** uploaded by the user;
  - one or more **image files** (most likely SVG) generated on the server;
  - two or three **JSON files** of structured data produced by the web app from user input.
- **DA-5** Users can create projects, rename them, delete them (with all their files), and
  view their project list.
- **DA-6** An uploaded PDF is at most **20 MB** and has at most **20 pages**.
- **DA-7** The server generates the project's image files **from the uploaded PDF**: each page
  of the PDF becomes **one SVG image**.
- **DA-8** The web app downloads the generated image files using **authenticated access**:
  only the owner of a project can fetch its images.

## 3. Non-functional requirements

- **NF-1 Security:** all traffic uses TLS. Data is encrypted at rest. User identity on the
  server comes from a verified token, never from values the client supplies.
- **NF-2 Isolation:** per-user data isolation is enforced on the server side.
- **NF-3 Infrastructure as code:** AWS resources are defined in code (AWS CDK, Python) so environments can be
  reproduced.
- **NF-4 Processing time:** images for an uploaded PDF are generated within **2 minutes**
  of the upload completing.

## 4. Expected usage

Planning assumptions for sizing and cost estimates.

| Measure | Assumption | Derived figure |
| --- | --- | --- |
| Users | **1,000** active users | |
| PDF uploads | **10 PDFs** per user per month | 10,000 PDFs per month (about 330 per day) |
| PDF pages | **4 pages** per PDF on average | 40,000 pages (SVG images) generated per month |
| PDF file size | **2.5 MB** per PDF on average | about 25 GB of new PDFs per month (about 300 GB after a year if nothing is deleted) |
| Editing | **10 hours** per user per week, editing data saved in JSON files | about 43,000 editing hours per month; about 250 users editing at once if editing is spread over a 40-hour working week |
| JSON saves | The web app saves **at most once per minute** while a user is editing | at most about 2.6 million saves per month (about 4 per second on average during a 40-hour working week) |
| JSON file size | about **50 KB** each | up to about 130 GB of saved JSON per month, kept as old versions for 30 days |

## 5. Open questions

To be resolved before choosing a detailed design. Answers will be recorded here.

- Average SVG size per page (to be measured in the P-2 library tests), which decides SVG
  storage and download costs.
- Whether data ever needs to be shared between users in future.
- Custom domain, AWS regions and data-residency constraints.
- Environments needed (e.g. dev, test, prod).
- Cost targets, availability targets, backup and retention needs.
- Compliance needs (e.g. GDPR data export and account deletion).
