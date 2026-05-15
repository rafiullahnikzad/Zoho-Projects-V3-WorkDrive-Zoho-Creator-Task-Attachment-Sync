# Zoho Projects V3 → WorkDrive → Zoho Creator: Task Attachment Sync

A Zoho Deluge snippet that fetches all file attachments from a Zoho Projects task (V3 API), downloads each file binary from WorkDrive, and uploads them to a file upload field on a Zoho Creator record — using the correct Creator v2.1 upload URL format.

> 💡 **2 days of debugging distilled into one working pattern.** The V3 API (`/api/v3/`) is the **only** Projects endpoint that returns WorkDrive-backed file attachments with their `third_party_file_id`. V1/V2 do not.

---

## 🎉 Confirmed Working

```
✅ Love JS.jpg                          → File Uploaded Successfully
✅ Screenshot (4603)...png              → File Uploaded Successfully
✅ Get_Started_With_Smallpdf...pdf      → File Uploaded Successfully
```

All 3 files transferred from a Zoho Projects task directly into a Zoho Creator record file upload field in a single Deluge function run.

---

## Table of Contents

- [Overview](#overview)
- [The Three-Step Pipeline](#the-three-step-pipeline)
- [Prerequisites](#prerequisites)
- [Connections & Required Scopes](#connections--required-scopes)
- [API Endpoints Used](#api-endpoints-used)
- [Code](#code)
- [How It Works](#how-it-works)
- [Key Discoveries](#key-discoveries)
- [Configuration Reference](#configuration-reference)
- [Extending the Pattern](#extending-the-pattern)
- [Common Errors](#common-errors)
- [Related](#related)

---

## Overview

This snippet solves a three-system file transfer problem entirely in Deluge — no middleware, no external tools:

```
Zoho Projects Task
       │
       │  V3 API → gets attachment list + WorkDrive file IDs
       ▼
Zoho WorkDrive
       │
       │  Download API → gets raw file binary
       ▼
Zoho Creator Record
       │
       │  v2.1 Upload API → stores file in upload field
       ▼
  ✅ File appears in Creator record
```

Three connections, three API calls per file, zero manual steps.

---

## The Three-Step Pipeline

| Step | System | What Happens |
|---|---|---|
| 1 | Zoho Projects V3 API | Fetch the attachment list for the task; extract `third_party_file_id` (WorkDrive ID) per file |
| 2 | Zoho WorkDrive API | Download the actual file binary using the WorkDrive file ID |
| 3 | Zoho Creator v2.1 API | Upload the binary to a file upload field on a specific Creator record |

---

## Prerequisites

| Requirement | Details |
|---|---|
| Zoho Projects portal | Must have V3 API access |
| Portal ID | Numeric ID of your Projects portal |
| Project ID | Numeric ID of the target project |
| Task ID | Numeric ID of the task with attachments |
| Creator record ID | ID of the Creator record to upload files into |
| Creator account | Your Creator account/portal name |
| Creator app | App link name |
| Creator report link name | The **report** link name — NOT the form name (critical difference) |
| Creator field name | The API name of the file upload field on the form |

---

## Connections & Required Scopes

Three separate OAuth connections are required. Create each under **Settings → Connections → Zoho OAuth** in your Creator app.

---

### Connection 1: `zohoprojects`

Used to fetch the task attachment list from Zoho Projects V3.

| Property | Value |
|---|---|
| Connection name | `zohoprojects` |
| Service | Zoho Projects |

**Required scopes:**

| Scope | Purpose |
|---|---|
| `ZohoProjects.tasks.ALL` | Read task data and attachment metadata |
| `ZohoPC.files.ALL` | Access Projects-linked file references |

> **Why V3?** Only the V3 API (`/api/v3/`) returns the `third_party_file_id` field, which is the WorkDrive file ID needed for the download step. V1 and V2 attachment endpoints do not include this field.

---

### Connection 2: `zoho_workdrive_conn`

Used to download the actual file binary from WorkDrive.

| Property | Value |
|---|---|
| Connection name | `zoho_workdrive_conn` |
| Service | Zoho WorkDrive |

**Required scopes:**

| Scope | Purpose |
|---|---|
| `WorkDrive.files.READ` | Download file content from WorkDrive by file ID |

> The download URL format is: `https://workdrive.zoho.com/api/v1/download/{third_party_file_id}`

---

### Connection 3: `creator`

Used to upload the file binary into the Creator record.

| Property | Value |
|---|---|
| Connection name | `creator` |
| Service | Zoho Creator |

**Required scopes:**

| Scope | Purpose |
|---|---|
| `ZohoCreator.report.UPDATE` | Write/upload files to an existing record via the report endpoint |

> **Why `report.UPDATE` and not `form.CREATE`?** The v2.1 file upload endpoint uses the **report** URL path to target an existing record. Form-based endpoints are for new record creation, not file uploads to existing records.

---

### Scope Summary Table

| Connection Name | Scopes |
|---|---|
| `zohoprojects` | `ZohoProjects.tasks.ALL`, `ZohoPC.files.ALL` |
| `zoho_workdrive_conn` | `WorkDrive.files.READ` |
| `creator` | `ZohoCreator.report.UPDATE` |

---

## API Endpoints Used

| Step | Method | Endpoint |
|---|---|---|
| Get task attachments | `GET` | `https://projectsapi.zoho.com/api/v3/portal/{portal_id}/projects/{project_id}/tasks/{task_id}/attachments` |
| Download file from WorkDrive | `GET` | `https://workdrive.zoho.com/api/v1/download/{third_party_file_id}` |
| Upload file to Creator record | `POST` | `https://www.zohoapis.com/creator/v2.1/data/{account}/{app}/report/{report}/{record_id}/{field}/upload` |

---

## Code

```deluge
// ============================================================
// FETCH TASK ATTACHMENTS (V3) → UPLOAD TO ZOHO CREATOR RECORD
// Connections required:
//   zohoprojects       → ZohoProjects.tasks.ALL, ZohoPC.files.ALL
//   zoho_workdrive_conn → WorkDrive.files.READ
//   creator            → ZohoCreator.report.UPDATE
// ============================================================

portal_id         = "YOUR_PORTAL_ID";
project_id        = "YOUR_PROJECT_ID";
task_id           = "YOUR_TASK_ID";
creator_record_id = "YOUR_CREATOR_RECORD_ID";
creator_account   = "YOUR_CREATOR_ACCOUNT";
creator_app       = "your-app-link-name";
creator_report    = "Your_Report_Link_Name";   // Report link name — NOT the form name
creator_field     = "Your_File_Upload_Field";

// -------------------------------------------------------
// STEP 1: Get attachments from Zoho Projects V3 API
// NOTE: Only V3 returns third_party_file_id (WorkDrive ID)
// -------------------------------------------------------
api_url = "https://projectsapi.zoho.com/api/v3/portal/"
        + portal_id + "/projects/"
        + project_id + "/tasks/"
        + task_id + "/attachments";

response = invokeurl
[
    url: api_url
    type: GET
    connection: "zohoprojects"
];

// V3 returns key "attachment" (singular), not "attachments"
attachments = response.get("attachment");

if(attachments == null || attachments.size() == 0)
{
    info "No attachments found.";
    return "";
}

info "✅ Total attachments: " + attachments.size();

upload_count = 0;

for each att in attachments
{
    att_name          = att.get("name");
    att_type          = att.get("type");
    workdrive_file_id = att.get("third_party_file_id");

    info "--- Processing: " + att_name;

    // -------------------------------------------------------
    // STEP 2: Download file binary from WorkDrive
    // Requires: WorkDrive.files.READ scope
    // -------------------------------------------------------
    wd_download_url = "https://workdrive.zoho.com/api/v1/download/"
                    + workdrive_file_id;

    file_data = invokeurl
    [
        url: wd_download_url
        type: GET
        connection: "zoho_workdrive_conn"
    ];

    info "Downloaded OK: " + att_name;

    // -------------------------------------------------------
    // STEP 3: Upload to Creator record using v2.1 upload URL
    // Format: /creator/v2.1/data/{account}/{app}/report/{report}/{recordID}/{field}/upload
    // Requires: ZohoCreator.report.UPDATE scope
    // IMPORTANT: Use `files:` not `parameters:` for this endpoint
    // IMPORTANT: Use report link name, not form name
    // -------------------------------------------------------
    creator_upload_url = "https://www.zohoapis.com/creator/v2.1/data/"
                       + creator_account + "/"
                       + creator_app + "/report/"
                       + creator_report + "/"
                       + creator_record_id + "/"
                       + creator_field + "/upload";

    file_part = map();
    file_part.put("stringPart", "false");
    file_part.put("paramName", "file");
    file_part.put("content", file_data);
    file_part.put("fileName", att_name);
    file_part.put("contentType", att_type);

    files_list = list();
    files_list.add(file_part);

    upload_resp = invokeurl
    [
        url: creator_upload_url
        type: POST
        files: files_list
        connection: "creator"
    ];

    info "Upload response for " + att_name + ": " + upload_resp;
    upload_count = upload_count + 1;
}

info "✅ Done! " + upload_count + " file(s) uploaded to Creator record: " + creator_record_id;
return "";
```

---

## How It Works

### Step 1 — Fetch Attachments (Projects V3 API)

The V3 endpoint returns a list under the key `"attachment"` (singular). Each object includes:

- `name` — original filename
- `type` — MIME type / file extension category
- `third_party_file_id` — the WorkDrive file ID, used in Step 2

```deluge
attachments = response.get("attachment");  // "attachment" not "attachments"
workdrive_file_id = att.get("third_party_file_id");
```

### Step 2 — Download from WorkDrive

The WorkDrive download API accepts the file ID directly:

```
GET https://workdrive.zoho.com/api/v1/download/{third_party_file_id}
```

The response is the raw file binary, ready to be passed to the Creator upload.

### Step 3 — Upload to Creator Record

The Creator v2.1 file upload URL targets an **existing record** via its report endpoint:

```
POST https://www.zohoapis.com/creator/v2.1/data/{account}/{app}/report/{report}/{recordID}/{field}/upload
```

The file is sent using `files:` (not `parameters:`), structured as a map with five required keys:

| Key | Value |
|---|---|
| `stringPart` | `"false"` — marks this as a binary file part |
| `paramName` | `"file"` — the multipart field name expected by Creator |
| `content` | the downloaded file binary |
| `fileName` | the original filename including extension |
| `contentType` | the MIME type of the file |

---

## Key Discoveries

These are the non-obvious findings that make this script work — each one cost significant debugging time:

| Discovery | Detail |
|---|---|
| **V3 API is required** | Only `/api/v3/` returns `third_party_file_id`. V1 and V2 attachment endpoints omit this field entirely. |
| **Response key is singular** | Projects V3 returns `"attachment"` not `"attachments"`. Using the wrong key returns `null`. |
| **Use report URL, not form URL** | The Creator v2.1 file upload endpoint requires the **report** link name in the URL path. The form link name returns a 404. |
| **Use `files:` not `parameters:`** | For Creator file uploads, `files:` is correct. Using `parameters:` (which works for Desk attachments) fails for Creator. |
| **`stringPart: "false"` is required** | Omitting this causes Creator to treat the binary as a plain string and reject the upload. |
| **Three separate connections** | Each system uses its own OAuth connection with its own scope set. A single shared connection cannot cover all three APIs. |
| **`ZohoCreator.report.UPDATE` not `CREATE`** | Uploading a file to an existing record requires the UPDATE scope on the report, not the CREATE scope on the form. |

---

## Configuration Reference

| Placeholder | Replace With |
|---|---|
| `YOUR_PORTAL_ID` | Numeric portal ID from Zoho Projects |
| `YOUR_PROJECT_ID` | Numeric project ID |
| `YOUR_TASK_ID` | Numeric task ID |
| `YOUR_CREATOR_RECORD_ID` | Numeric ID of the target Creator record |
| `YOUR_CREATOR_ACCOUNT` | Your Creator account/portal name (e.g. `mycompany`) |
| `your-app-link-name` | Creator app link name (from app URL) |
| `Your_Report_Link_Name` | API link name of the report (not the form name) |
| `Your_File_Upload_Field` | API field name of the file upload field |
| `zohoprojects` | Your Projects OAuth connection name |
| `zoho_workdrive_conn` | Your WorkDrive OAuth connection name |
| `creator` | Your Creator OAuth connection name |

### Finding the Report Link Name

The report link name is not the same as the display name shown in the UI.

1. Open your Creator app
2. Go to **Reports**
3. Click the report → **Properties**
4. The **Link Name** field is what goes in the URL (e.g. `Projects_for_Admins`)

Alternatively, open the report in the Creator UI and read it from the browser URL:
```
https://creator.zoho.com/.../#Report:Your_Report_Link_Name
```

---

## Extending the Pattern

### Process Only Specific File Types

```deluge
for each att in attachments
{
    att_name = att.get("name");
    // Only process PDFs
    if(att_name.contains(".pdf") || att_name.endsWith(".pdf"))
    {
        // download and upload logic here
    }
}
```

### Add Error Handling per File

Wrap each download/upload in `try/catch` so one bad file doesn't abort the entire loop:

```deluge
for each att in attachments
{
    try
    {
        // download + upload steps here
        upload_count = upload_count + 1;
    }
    catch (e)
    {
        info "❌ Failed for " + att.get("name") + ": " + e;
    }
}
```

### Store Upload Results Back in Creator

After uploading, update a text field on the same record to log the sync timestamp:

```deluge
update_map = map();
update_map.put("Last_Sync", zoho.currenttime.toString("dd-MMM-yyyy HH:mm"));
update_map.put("Files_Synced", upload_count.toString());

zoho.creator.updateRow(
    "YOUR_CREATOR_ACCOUNT",
    "your-app-link-name",
    "Your_Report_Link_Name",
    "ID == " + creator_record_id,
    update_map
);
```

### Run as a Scheduled Sync

Wrap the script in a `void` function and call it from a Zoho Creator scheduler, passing the task ID and record ID as parameters — enabling automatic periodic syncing of task attachments into Creator records.

---

## Common Errors

| Error | Likely Cause | Fix |
|---|---|---|
| `attachment` key is `null` | Task has no attachments, or using wrong API version | Confirm task has files; ensure URL uses `/api/v3/` not `/api/v1/` |
| `third_party_file_id` is `null` or empty | Using V1 or V2 Projects API | Switch to `/api/v3/` endpoint — only V3 returns this field |
| WorkDrive download returns `401` | Missing `WorkDrive.files.READ` scope | Edit `zoho_workdrive_conn` and add the scope |
| Creator upload returns `404` | Using form link name instead of report link name | Check the report's **Link Name** in Creator → Reports → Properties |
| Creator upload returns `400` | Wrong field name, or `stringPart` missing | Verify the field API name; ensure `stringPart` is set to `"false"` |
| Creator upload returns `403` | Missing `ZohoCreator.report.UPDATE` scope | Edit the `creator` connection and add the scope |
| Files upload but field appears empty | Using `parameters:` instead of `files:` | Switch to `files: files_list` in the Creator `invokeurl` block |

---

## Related

- [Get Task Attachments — Zoho Projects V3 API](./README_task_attachments.md) — the fetch-only version of Step 1
- [Zoho Creator → Zoho Desk: File Upload via Deluge](./README_file_upload.md) — same download pattern, different upload destination
- [Full Onboarding Wizard Ticket Creator](./README.md) — end-to-end ticket creation with multi-source attachment handling
- [Zoho Projects V3 Attachments API](https://www.zoho.com/projects/help/rest-api/task-attachments-api.html)
- [Zoho WorkDrive Download API](https://workdrive.zoho.com/apidocs/v1/)
- [Zoho Creator v2.1 File Upload API](https://www.zoho.com/creator/help/api/v2.1/)

---

*Built with Zoho Deluge | Zoho Projects V3 API | Zoho WorkDrive API | Zoho Creator v2.1 API*
