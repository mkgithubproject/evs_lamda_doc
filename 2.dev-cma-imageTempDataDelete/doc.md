Got it! Based on the details you shared, I can summarize and organize the information into a **flowchart-style logical flow** for your `evdev-cma-imageTempDataDelete` Lambda function. Here’s a detailed breakdown:

---

## **Lambda Function: evdev-cma-imageTempDataDelete**

**Trigger:** EventBridge (every 15 minutes)
**Memory:** 1024 MB
**Layers:** `dev-cma-layer-bulk`, `dev-cma-layer-bulk2`, `dev-cma-layer-sellerpayment`

**Objective:** Delete unused images from S3 and clean up database entries.

---

### **S3 Folders and Deletion Logic**

| Folder                | S3 Path                                | Timestamp Timezone | Deletion Criteria  |
| --------------------- | -------------------------------------- | ------------------ | ------------------ |
| Collected Images      | `product-images/collected/`            | JP Time            | Older than 3 hours |
| Temporary Images      | `product-images/temp/`                 | UTC                | Older than 3 hours |
| Bulk Collected Images | `product-images/bulk-image-collected/` | UTC                | Older than 3 hours |

**Database Cleanup:**

* Table: `ec_product_images`
* Delete entries **only** if `status = 1` for the images deleted in S3.

---

### **New Function Addition (Post-Main Execution)**

After the main image cleanup:

1. **Retrieve products**

   * Get all product data where `image_flg_ec = 0` for all `cainz_product_code` processed in the main function.

2. **Check ec_unregistered_image table**

   * For each `cainz_product_code` retrieved, check if an entry exists in `ec_unregistered_image`.

3. **Insert if missing**

   * If no entry exists, create a new entry in `ec_unregistered_image` for that `cainz_product_code`.

---

### **Flowchart Representation**

```
EventBridge Trigger (Every 15 min)
            │
            ▼
  Start Lambda Function
            │
            ▼
   Delete S3 Images (older than 3 hours)
   ├─ Collected Images (JP Time)
   ├─ Temporary Images (UTC)
   └─ Bulk Collected Images (UTC)
            │
            ▼
 Delete entries from ec_product_images where status = 1
            │
            ▼
  --- New Function Execution ---
            │
 Retrieve all products where image_flg_ec = 0
            │
 For each cainz_product_code:
 ├─ Check ec_unregistered_image table
 └─ If not exists → Insert new entry
            │
            ▼
      Lambda Execution Ends
```

---


