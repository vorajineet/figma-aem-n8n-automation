Now I can see the full workflow. I can identify all the nodes clearly. Here's a detailed `README.md` you can use: [localhost](http://localhost:5678/workflow/I6QiyGvguMwQySC2)

***

```markdown
# Figma Design to AEM Content Fragment — n8n Workflow

An automated n8n workflow that bridges **Figma** and **Adobe Experience Manager (AEM)**. It reads design files from a Figma project, extracts content and assets, generates AI-powered SEO tags via OpenAI, and publishes everything as structured **Content Fragments** and **Assets** in AEM — with zero manual copying.

---

## Table of Contents

- [Overview](#overview)
- [Workflow Architecture](#workflow-architecture)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
  - [1. n8n Setup](#1-n8n-setup)
  - [2. Figma Setup](#2-figma-setup)
  - [3. AEM Setup](#3-aem-setup)
  - [4. OpenAI Setup](#4-openai-setup)
  - [5. Configure Credentials in n8n](#5-configure-credentials-in-n8n)
- [Workflow Nodes Explained](#workflow-nodes-explained)
- [Running the Workflow](#running-the-workflow)
- [Environment Variables](#environment-variables)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

This workflow automates a content pipeline that:

1. Fetches all **[Ready]** tagged files from a Figma project
2. Retrieves the Figma file content and parses it
3. Generates **SEO tags** using OpenAI (GPT)
4. Fetches and downloads **image assets** from Figma
5. Uploads those assets to the **AEM DAM**
6. Creates a **Content Fragment** in AEM with the parsed content
7. Loads all data (text + asset references) into the Content Fragment

---

## Workflow Architecture

```
Execute Workflow
      │
      ▼
Figma - Get Files (GET /v1/files)
      │
      ▼
Fetch files with [Ready] state
      │
      ▼
Figma - Get File (GET /v1/files/:key)
      │
      ▼
Parse content from Figma
      │
      ├──────────────────────────────────┐
      ▼                                  ▼
Generate SEO tags (OpenAI)     Figma - Get Asset URL
      │                                  │
      ▼                                  ▼
Merge into JSON                   Download Asset
      │                                  │
      ▼                                  ▼
Clean the JSON                  AEM - Create Asset (DAM)
      │                                  │
      ├──────────────────────────────────┘
      ▼
AEM - Create CF (Content Fragment)
      │
      ▼
Get CF name
      │
      ▼
Merge all data to JSON
      │
      ▼
AEM - Load Data to CF
```

---

## Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| [n8n](https://n8n.io) | ≥ 1.30.0 | Self-hosted or cloud |
| Adobe Experience Manager | 6.5+ or AEM as a Cloud Service | With Content Fragments enabled |
| Figma Account | Any plan | With API access |
| OpenAI Account | GPT-3.5 or GPT-4 | For SEO tag generation |
| Node.js | ≥ 18.x | If running n8n locally |

---

## Setup

### 1. n8n Setup

#### Option A — Docker (Recommended)

```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

#### Option B — npm

```bash
npm install n8n -g
n8n start
```

Once running, open **http://localhost:5678** in your browser.

#### Import the Workflow

1. In n8n, go to **Workflows → Import from file**
2. Select the `figma-design-to-aem-cf.json` file from this repo
3. Click **Import**

---

### 2. Figma Setup

#### Get a Figma Personal Access Token

1. Log in to [figma.com](https://figma.com)
2. Go to **Account Settings → Security → Personal Access Tokens**
3. Click **Generate new token**, give it a name, and copy it

#### Figma File Convention

This workflow uses a naming convention to identify files ready for processing:

- Files must be tagged with `[Ready]` in their name  
  e.g. `[Ready] Homepage Banner Design`
- The workflow will **only pick up files** that match this pattern
- Once processed, you should rename them to `[Done]` to avoid reprocessing

#### Get your Figma Project ID

1. Open your Figma project in the browser
2. The URL will look like: `https://www.figma.com/files/project/YOUR_PROJECT_ID/...`
3. Copy the `YOUR_PROJECT_ID` value

---

### 3. AEM Setup

#### AEM Requirements

- **Content Fragment Model** must be created before running the workflow
- The **AEM Assets HTTP API** must be enabled
- A service user or admin account with permissions to:
  - Create/update Assets in DAM (`/content/dam/...`)
  - Create/update Content Fragments (`/content/dam/.../content-fragments/...`)

#### AEM Content Fragment Model

Create a Content Fragment Model in AEM with (at minimum) the following fields:

| Field Label | Field Type | Property Name |
|-------------|-----------|---------------|
| Title | Single-line text | `title` |
| Description | Multi-line text | `description` |
| SEO Tags | Tags / Text | `seoTags` |
| Image | Content Reference | `image` |

> Navigate to: **Tools → Assets → Content Fragment Models** to create the model.

#### AEM DAM Path

Decide on a DAM folder path where assets and CFs will be stored, e.g.:
```
/content/dam/figma-imports/
```
Create this folder in AEM before running the workflow.

#### AEM Base URL

If running AEM locally via Docker:
```
http://host.docker.internal:4502
```
If running on AEM Cloud:
```
https://author-pXXXXX-eYYYYY.adobeaemcloud.com
```

---

### 4. OpenAI Setup

1. Go to [platform.openai.com](https://platform.openai.com)
2. Navigate to **API Keys → Create new secret key**
3. Copy the key — you will need it for the n8n credential

---

### 5. Configure Credentials in n8n

In n8n, go to **Settings → Credentials** and create the following:

#### Figma API

| Field | Value |
|-------|-------|
| Credential Type | `Header Auth` |
| Name | `Figma API` |
| Header Name | `X-Figma-Token` |
| Header Value | `<your Figma personal access token>` |

#### AEM (Basic Auth)

| Field | Value |
|-------|-------|
| Credential Type | `HTTP Basic Auth` |
| Name | `AEM Admin` |
| Username | `admin` (or your service account) |
| Password | `<your AEM password>` |

#### OpenAI

| Field | Value |
|-------|-------|
| Credential Type | `OpenAI API` |
| Name | `OpenAI` |
| API Key | `<your OpenAI API key>` |

After creating credentials, open the imported workflow and assign each credential to its respective node.

---

## Workflow Nodes Explained

| Node | Purpose |
|------|---------|
| **Execute Workflow** | Trigger node — starts the pipeline manually or on a schedule |
| **Figma - Get Files** | Calls `GET /v1/projects/:projectId/files` to list all files in the project |
| **Fetch files with [Ready] state** | Filters the file list to only process files tagged `[Ready]` |
| **Figma - Get File** | Fetches the full file content using `GET /v1/files/:fileKey` |
| **Parse content from Figma** | Extracts text layers (title, body, etc.) from the Figma JSON structure |
| **Generate SEO tags** | Sends parsed content to OpenAI GPT to generate relevant SEO metadata |
| **Merge into JSON** | Combines the parsed content and SEO tags into a single object |
| **Clean the JSON** | Removes null/empty fields and normalises the data structure |
| **Figma - Get Asset URL** | Fetches the export/render URL for image assets in the Figma file |
| **Download Asset** | Downloads the binary image from the Figma render URL |
| **AEM - Create Asset** | Uploads the image to the AEM DAM via `POST /api/assets/:path` |
| **AEM - Create CF** | Creates an empty Content Fragment in AEM using the CF model |
| **Get CF name** | Extracts the generated CF path/name from AEM's response |
| **Merge all data to JSON** | Combines CF path, asset reference, and content into one payload |
| **AEM - Load Data to CF** | Patches the Content Fragment with all field values via the AEM CF API |

---

## Running the Workflow

1. Open the workflow in n8n
2. Ensure all credentials are assigned to nodes
3. Verify your Figma project has at least one file named with the `[Ready]` prefix
4. Click **Execute Workflow**
5. Monitor each node's output by clicking on it after execution
6. In AEM, navigate to your DAM folder to verify the asset and Content Fragment were created

---

## Environment Variables

If you are running n8n via Docker Compose, you can set these as environment variables:

```env
# Figma
FIGMA_PROJECT_ID=your_figma_project_id

# AEM
AEM_BASE_URL=http://host.docker.internal:4502
AEM_DAM_PATH=/content/dam/figma-imports
AEM_CF_MODEL_PATH=/conf/global/settings/dam/cfm/models/your-model-name

# OpenAI
OPENAI_MODEL=gpt-4
```

> These need to be referenced inside the relevant n8n nodes via expressions, e.g. `{{ $env.AEM_BASE_URL }}`.

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| No files returned from Figma | No files tagged `[Ready]` | Rename at least one Figma file to include `[Ready]` |
| 401 error on AEM nodes | Wrong credentials | Check AEM username/password in n8n credentials |
| 404 on AEM - Create CF | CF Model path incorrect | Verify the model path in **Tools → Assets → Content Fragment Models** |
| OpenAI node fails | Invalid/expired API key | Regenerate key at platform.openai.com |
| Asset upload fails | DAM folder doesn't exist | Create the target folder in AEM DAM first |
| Docker networking issue | AEM unreachable from n8n container | Use `host.docker.internal` instead of `localhost` for AEM URL |

---

## Contributing

Pull requests are welcome. For major changes, please open an issue first to discuss what you'd like to change.

---

## License

MIT License — see [LICENSE](LICENSE) for details.