## >> [Link to MAIN AI Virtual Storyteller Repository](https://github.com/darthvandor13/CSC212-Virtual-Storyteller) <<

# ai-storyteller-agent-config

# AI Storyteller Conversational Agent

## Project Overview

This project implements a conversational AI agent using **Google Cloud Dialogflow CX (Conversational Agents)** designed to act as a virtual storyteller. The agent engages users in a conversation to gather preferences (protagonist name, story theme, moral) and then generates a unique short story (300-350 words).

The core mechanism is **Retrieval-Augmented Generation (RAG)**. The agent retrieves relevant context snippets from a predefined knowledge base of stories (hosted in Google Cloud Storage) and uses these snippets, along with the user's parameters, as strict inspiration to generate a new, original story. It is explicitly instructed *not* to use external knowledge or copy directly from the source material.

This project utilizes native Dialogflow CX Data Store features for RAG.

*(**Note on Initial Approach:** An earlier iteration attempted using ChromaDB via a webhook and parameter presets. This approach was paused due to platform issues encountered with parameter preset handling – specifically, the UI automatically adding quotes to `$webhookResponse` paths.)*

## Technology Stack

* **Conversational Platform:** Google Cloud Dialogflow CX / Conversational Agents
* **Knowledge Base Backend:** Google Cloud Storage (GCS)
* **Secrets Management:** Google Cloud Secret Manager
* **Version Control:** GitHub

## Core Concepts: RAG via GCS Data Store

1.  **Data Storage:** Source story texts (PDF, TXT, HTML formats supported) are stored in a GCS bucket.
2.  **Data Store Resource:** A Dialogflow "Data Store" resource is created within the Agent Builder environment, configured to point to the GCS bucket and index the unstructured content. *(Note: A workaround using "One time" sync instead of "Periodic" was needed during development due to a suspected UI issue preventing the data store from being selectable).*
3.  **Data Store Tool:** A "Data Store Tool" is created within the agent and linked to the specific Data Store resource.
4.  **Retrieval:** The agent's playbook logic calls this Data Store Tool, sending a query based on user input (e.g., theme, moral).
5.  **Generation:** The tool returns relevant text snippets found within the indexed documents. These snippets, along with user parameters, are used within a generative prompt executed by the agent's playbook to create the final story.

## Setup & Prerequisites (Start-to-Finish)

This guide assumes you have a Google Cloud project with billing enabled.

1.  **Enable APIs:**
    * In the Google Cloud Console, ensure the following APIs are enabled for your project:
        * Dialogflow API
        * Secret Manager API
        * Cloud Storage API
        * Vertex AI Agent Builder API (or Vertex AI Search and Conversation API)
    * *(Optional: Add link to GCP documentation on enabling APIs)*

2.  **Create GCS Bucket:**
    * Navigate to Cloud Storage -> Buckets in the Google Cloud Console.
    * Click "Create".
    * Choose a unique **Bucket name** (e.g., `[YOUR_PROJECT_ID]-story-data`).
    * Select **Location type: Region** and choose the **SAME REGION** as your Dialogflow agent (e.g., `us-central1`). This is critical.
    * Select **Storage class: Standard**.
    * Configure access control (Uniform is recommended).
    * Click "Create".
    * *(Optional: Add link to GCP documentation on creating GCS buckets)*

3.  **Upload Story Files:**
    * Navigate into the GCS bucket you just created.
    * Upload your source story files (e.g., `.pdf`, `.txt`). Organize into folders if desired.

4.  **Create GitHub Repository:**
    * Go to GitHub.
    * Create a new, **dedicated repository** for syncing this agent's configuration (e.g., `ai-storyteller-agent-config`). It can be private or public. Initialize it (e.g., with a LICENSE or empty README).
    * Ensure the primary branch you want to use exists (e.g., `main`).
    * *(Optional: Add link to GitHub documentation on creating repositories)*

5.  **Create GitHub Personal Access Token (PAT):**
    * In GitHub: Go to Settings -> Developer settings -> Personal access tokens -> Fine-grained tokens.
    * Click "Generate new token".
    * Give it a **Name** (e.g., `dialogflow-cx-sync`).
    * Set an **Expiration**.
    * Under "Repository access", select **"Only select repositories"** and choose the repository created in the previous step.
    * Under "Permissions" -> "Repository permissions", find **"Contents"** and set its access level to **"Read and write"**.
    * Click "Generate token".
    * **COPY THE TOKEN IMMEDIATELY** and store it securely temporarily.
    * *(Optional: Add link to GitHub documentation on creating fine-grained PATs)*

6.  **Store PAT in Secret Manager:**
    * In the Google Cloud Console, navigate to Security -> Secret Manager.
    * Click "+ Create Secret".
    * Enter a **Name** (e.g., `github-dialogflow-pat`).
    * Paste the copied GitHub PAT into the **Secret value** field.
    * Click "Create Secret".
    * *(Optional: Add link to GCP documentation on Secret Manager)*

7.  **Grant Dialogflow Access to Secret:**
    * In Secret Manager, click on the name of the secret you just created (`github-dialogflow-pat`).
    * Go to the "Permissions" tab.
    * Click "+ Grant Access".
    * In "New principals", enter the Dialogflow Service Agent: `service-[YOUR_PROJECT_NUMBER]@gcp-sa-dialogflow.iam.gserviceaccount.com` (Replace `[YOUR_PROJECT_NUMBER]` with your actual Google Cloud project number).
    * In "Assign roles", select `Secret Manager Secret Accessor`.
    * Click "Save".
    * *(Optional: Add link to GCP documentation on granting Secret Manager permissions)*

8.  **Grant Dialogflow Access to GCS Bucket:**
    * Navigate to Cloud Storage -> Buckets -> click your story data bucket name.
    * Go to the "Permissions" tab.
    * Click "+ Grant Access".
    * In "New principals", enter the Dialogflow Service Agent: `service-[YOUR_PROJECT_NUMBER]@gcp-sa-dialogflow.iam.gserviceaccount.com`.
    * In "Assign roles", select `Storage Object Viewer`.
    * Click "Save".
    * *(Optional: Add link to GCP documentation on granting GCS permissions)*

## Agent Configuration Steps

These steps are performed within the Dialogflow CX / Conversational Agents console.

1.  **Create Agent (if needed):**
    * Create a new agent, selecting **"Build your own"**.
    * Specify Agent Name, **Location/Region** (e.g., `us-central1` - must match GCS bucket), Time zone, Default language.
    * *(Optional: Add link to Dialogflow CX documentation on creating agents)*

2.  **Create Data Store Resource:**
    * Navigate to the "Data Stores" section.
    * Click "Create new data store".
    * Choose **"Cloud Storage"** as the source.
    * **Browse** and select the GCS bucket/folder containing your story files.
    * Choose **"Unstructured documents (PDF, HTML, TXT and more)"** as the data type.
    * Set **Synchronization frequency** to **"One time"** (this was found to work around a potential issue where periodically synced stores didn't immediately appear for tool linking).
    * Click "Continue".
    * **Configure data connector:**
        * **Location:** Select the location corresponding to your agent/bucket region (e.g., `us (multi-region)` for agent region `us-central1`). **This cannot be changed later.**
        * **Data connector name:** Give it a unique name (e.g., `StoryContentGCS-Connector`).
        * Click "Create".
    * Wait for indexing to complete ("Data Ingestion Activity" tab should show "Succeeded").
    * *(Optional: Add link to Dialogflow CX documentation on creating data stores)*

3.  **Create Data Store Tool:**
    * Navigate to the "Tools" section.
    * Click "+ Create".
    * **Tool Name:** e.g., `WorkspaceStoryContextTool-OneTime` (use a consistent name).
    * **Tool Type:** Select `Data store`.
    * **Data stores:** Click "Add data stores" and select the data store created above (`StoryContentGCS` or similar name). It should appear now that ingestion is complete and "One time" sync was used.
    * **Description:** Add a helpful description (e.g., "Queries GCS story data").
    * Click "Save".
    * *(Optional: Add link to Dialogflow CX documentation on creating tools)*

4.  **Configure GitHub Integration (in Agent Settings):**
    * Go to Agent Settings (⚙️ icon).
    * Find the "Git integration" section.
    * Click "+ Add Git integration".
    * **Display name:** e.g., `GitHub Sync - Storyteller Agent`.
    * **Git repository:** Enter the HTTPS URL of your dedicated GitHub repo.
    * **Branch:** Enter the branch name (e.g., `main`).
    * **Access token secret:** Enter the full resource name of the Secret Manager secret version holding your PAT (e.g., `projects/[PROJECT_ID]/secrets/[SECRET_NAME]/versions/latest`).
    * Click "Connect" and then "Save" agent settings.
    * *(Optional: Add link to Dialogflow CX documentation on Git integration)*

5.  **Import/Configure Playbook:**
    * If starting fresh, you can import the playbook from this repository (using the "Import" button in the Playbooks section and choosing "Rename and import as new resources" if conflicts exist).
    * Ensure you are working with the correct playbook (e.g., `Storyteller Playbook - GCS Test`).
    * Go to the Playbook editor.

6.  **Configure Playbook Instructions:**
    * Paste the final instructions (provided in the previous response) into the "Instructions" field. Ensure all references use the GCS tool and `$session.params.datastore_snippets` (or the tool output path directly).

7.  **Configure Playbook Example:**
    * Edit the main example demonstrating the successful path.
    * Ensure the conversational turns for parameter gathering are present.
    * After the user confirms ("yes"), ensure the sequence is:
        * **Tool Use:** Calls `WorkspaceStoryContextTool-OneTime`.
            * Input `requestBody` JSON set correctly (with query using `$session.params.theme` and `$session.params.moral`). Make sure the `fallback` key is *removed* from the input.
            * Example Input JSON:
                ```json
                {
                  "query": "Story about $session.params.theme with a moral of $session.params.moral",
                  "filter": "",
                  "userMetadata": {}
                }
                ```
        * **Agent response:** Contains the generative prompt.
            * This prompt must reference the user parameters (`$session.params.*`) and the *direct output path* from the tool (since output mapping in the tool step seemed problematic): `$tool.WorkspaceStoryContextTool-OneTime.snippets`.
            * Example Prompt Text (Place inside the Agent Response text field):
                ```text
                Okay, traveler! Using the inspiration found in our archives ($tool.WorkspaceStoryContextTool-OneTime.snippets), here is a new short story (300-350 words) about $session.params.protagonist in a $session.params.theme setting. The story must teach the moral '$session.params.moral'. Remember to base the story ONLY on the retrieved context from the archives, adapting creatively but not adding external information or copying verbatim. Make sure it has a beginning, middle, and end.

                Would you like me to craft another tale, traveler?
                ```
    * Configure the "End example with output information" (Summary: "Agent successfully generated...", State: "OK").
    * Save the Example.
    * *(Optional: Add link to Dialogflow CX documentation on Playbook Examples)*

8.  **Configure Agent Generative Settings:**
    * Go to Agent Settings -> Generative AI -> Playbook.
    * Select the desired **Model** (e.g., `gemini-1.5-flash-001`).
    * Set **Temperature** (e.g., `0.5` or `0.6`).
    * Set **Token Limit** (e.g., `600`).
    * Save Agent Settings.

9.  **Initial Git Push:**
    * Go to Agent Settings -> Git integration.
    * Find your configured connection.
    * Click **"Push"**. Provide a commit message (e.g., "Initial agent setup for GCS RAG").

## How to Use/Test

1.  Open the Dialogflow CX / Conversational Agents console.
2.  Select your agent (`AI-Storyteller`).
3.  Open the **Simulator**.
4.  In the Simulator settings (might be a dropdown near the top):
    * Select **Environment:** `Draft` (usually).
    * Select **Start Resource:** Choose `Playbook`.
    * Select the specific playbook: `Storyteller Playbook - GCS Test` (or the name of your imported/active GCS playbook).
5.  Interact with the agent, providing a protagonist, theme, and moral when prompted.
6.  Observe the generated story and use the debugger (Conversation History) if needed.

## Known Issues / Troubleshooting Context

* **Webhook Parameter Preset Bug:** The initial ChromaDB/Webhook approach using parameter presets failed due to a suspected UI bug automatically adding quotes around `$webhookResponse...` paths, preventing successful data extraction. This necessitated the pivot to the GCS Data Store method.
* **GCS Data Store Visibility:** Data Stores created with "Periodic" sync initially failed to appear in the list when creating the Data Store Tool. Using **"One time"** sync resolved this, suggesting a potential delay or state issue with periodic syncing for immediate tool linking.
* **GCS Permissions:** Ensure the Dialogflow Service Agent has `Storage Object Viewer` on the GCS bucket for ingestion to work.
* **Data Store Indexing:** Ingestion and indexing from GCS can take time. Check the "DATA INGESTION ACTIVITY" tab for status ("Running", "Succeeded", "Failed") and errors. The data store won't be usable until indexing succeeds. Temporary resource exhaustion messages may appear during indexing, requiring patience for automated retries.

## Future Work

* Add more Playbook Examples to cover variations (different inputs) and edge cases (e.g., data store tool returning no snippets, user changing parameters).
* Tune the generative prompt and agent Temperature/Token settings for optimal story quality and constraint adherence.
* Refine the conversational style (e.g., enforce one question per turn if desired).
* Monitor Google Cloud updates for fixes related to the parameter preset quoting issue or data store visibility/syncing.
