Here is a **detailed explanation with workflow and examples** for the `evdev-cma-reportDelete` Lambda function.

---

## **1. Objective Overview**

The `evdev-cma-reportDelete` Lambda function is responsible for **cleaning up old files and database entries** related to bulk upload operations and product downloads. It is triggered **hourly** via EventBridge cron.

**Key responsibilities:**

| Operation             | Files / Data                     | Retention Rule                          |
| --------------------- | -------------------------------- | --------------------------------------- |
| Bulk product upload   | XLSX, CSV, report files          | Delete > 30 days                        |
| Bulk inventory upload | XLSX, CSV ('before' folder)      | Delete > 30 days                        |
| Bulk image upload     | Raw ZIP, validated ZIP, send ZIP | Delete > 48 hours; DB entries > 30 days |
| Product list download | Downloaded CSV/XLSX              | Delete > 1 hour                         |

---

## **2. Locations in S3**

| Folder                              | Purpose                      |
| ----------------------------------- | ---------------------------- |
| `product-images/raw-zip`            | Raw bulk image uploads       |
| `product-images/validated-zip`      | Validated bulk image uploads |
| `product-images/validated-zip/send` | Final ZIPs for Cainz pickup  |
| `product/xlsx`                      | Product bulk upload XLSX     |
| `product/csv`                       | Product bulk upload CSV      |
| `product/report`                    | Product report files         |
| `inventory/xlsx`                    | Inventory bulk upload XLSX   |
| `inventory/csv`                     | Inventory bulk upload CSV    |
| `inventory/before`                  | Pre-processed inventory CSVs |
| `product/downloaded-list/`          | Downloaded product files     |

---

## **3. Step-by-Step Workflow**

```plaintext
 ┌─────────────────────────────┐
 │ EventBridge Cron Trigger     │
 │ (every 1 hour)              │
 └───────────┬─────────────────┘
             │
             ▼
 ┌─────────────────────────────┐
 │ Initialize AWS SDK & DB     │
 │ connection (TypeORM)        │
 └───────────┬─────────────────┘
             │
             ▼
 ┌─────────────────────────────┐
 │ Check Maintenance Mode       │
 │ - If ON and lambda disabled │
 │   → Exit with 503           │
 └───────────┬─────────────────┘
             │
             ▼
 ┌─────────────────────────────┐
 │ Calculate Deletion Timestamps│
 │ - 30 days ago               │
 │ - 48 hours ago              │
 │ - 1 hour ago                │
 └───────────┬─────────────────┘
             │
             ▼
 ┌─────────────────────────────┐
 │ Fetch old bulk upload       │
 │ transactions from DB       │
 │ (product, inventory, images)│
 └───────────┬─────────────────┘
             │
             ▼
 ┌─────────────────────────────┐
 │ Delete bulk image files     │
 │ - raw-zip, validated-zip   │
 │ - older than 48 hours      │
 │ - send folder for Cainz     │
 └───────────┬─────────────────┘
             │
             ▼
 ┌─────────────────────────────┐
 │ Delete bulk product files   │
 │ - XLSX, CSV, report files  │
 │ - older than 30 days       │
 └───────────┬─────────────────┘
             │
             ▼
 ┌─────────────────────────────┐
 │ Delete bulk inventory files │
 │ - XLSX, CSV, before folder │
 │ - older than 30 days       │
 └───────────┬─────────────────┘
             │
             ▼
 ┌─────────────────────────────┐
 │ Delete downloaded product   │
 │ files from S3 > 1 hour old │
 └───────────┬─────────────────┘
             │
             ▼
 ┌─────────────────────────────┐
 │ Remove DB entries           │
 │ - BulkUploadTransaction     │
 │ - BulkUploadTransactionImages│
 └───────────┬─────────────────┘
             │
             ▼
 ┌─────────────────────────────┐
 │ Return success response     │
 │ statusCode 200              │
 └─────────────────────────────┘
```

---

## **4. Example Scenarios**

### **A. Delete old bulk product upload**

* **DB:** Entry in `BulkUploadTransaction` with batch_id `12345` uploaded 31 days ago.
* **S3 Paths to delete:**

  * `product/xlsx/12345/`
  * `product/csv/12345/`
  * `product/report/12345/`
* **Action:** Lambda deletes files and removes DB entry.

### **B. Delete old bulk inventory upload**

* **DB:** Entry in `BulkUploadTransaction` with batch_id `54321` uploaded 32 days ago.
* **S3 Paths to delete:**

  * `inventory/xlsx/54321/`
  * `inventory/before/` (any CSV older than 30 days)
* **Action:** Lambda deletes files and removes DB entry.

### **C. Delete old bulk image upload**

* **DB:** Bulk image transaction batch `img001` uploaded 50 hours ago.
* **S3 Paths to delete:**

  * `product-images/raw-zip/img001/`
  * `product-images/validated-zip/img001/`
  * `product-images/validated-zip/send/` older than 48 hours
* **Action:** Lambda deletes files and removes DB entry if older than 30 days.

### **D. Delete old downloaded product lists**

* **S3 Path:** `product/downloaded-list/`
* **Rule:** Any file older than 1 hour is deleted.

---

## **5. Notes on Implementation**

1. **Utility function `deleteS3Files(s3, params, olderThan?)`** handles deletion of files in S3 with optional date filtering.
2. **Promise.all** ensures parallel deletion for efficiency.
3. **Maintenance mode check** prevents deletion during system downtime.
4. **Chunked removal** of DB entries ensures TypeORM performance (`chunk: 100`).

---

This workflow ensures that **S3 storage and database remain clean**, preventing accumulation of temporary bulk upload and download files.

---

If needed, I can create a **visual workflow diagram** for this Lambda similar to the one for `evdev-cma-downloadProduct`. It will clearly show all deletion flows with time thresholds.

Do you want me to create that diagram?
