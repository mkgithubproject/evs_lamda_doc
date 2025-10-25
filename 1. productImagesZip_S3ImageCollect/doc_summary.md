


---

## ✅ **Objective Summary (In Simple Terms)**

This Lambda:

1. Reads product images from **S3 "collected" folder**:
   `product-images/collected/{cainz_product_code}/{image_name}`

2. Cross checks these images with database entries in `ec_product_image` table
   Only images with `status = 0` (unread) are considered.

3. Applies timing and size conditions to decide whether to **create ZIP or wait**.

4. If allowed:

   * Create ZIP of only **complete products**
     (Never include partial images of any cainz product code)
   * Save ZIP under:
     `product-images/validated-zip/send/IFI_IMAGE_DATA_MARKETPLACE_YYYYMMDDhhmmss.zip`
   * Update product images `status = 1` (read)

---

## ✅ **CONFIG Conditions (Important Logic)**

| Condition                                         | Action                              |
| ------------------------------------------------- | ----------------------------------- |
| First unread image uploaded ≥ **10 mins** ago     | ✅ Create ZIP                        |
| Total unread image size ≥ **100 MB**              | ✅ Create ZIP                        |
| First unread image ≥ 10 mins **AND** size < 100MB | ✅ Create ZIP                        |
| First unread image < 10 mins **AND** size < 100MB | ❌ Skip ZIP                          |
| Total images > **1 GB**                           | ✅ Create ZIP but limit files to 1GB |

---

## ✅ **Maintenance Mode Behavior**

| Mode                                                            | Action                                                 |
| --------------------------------------------------------------- | ------------------------------------------------------ |
| MAINTENANCE_MODE = true & ENABLE_LAMBDA_IN_MAINTENANCE_MODE = 0 | ❌ Do nothing (do not ZIP, do not delete unread images) |
| Maintenance ends                                                | ✅ Resume ZIP creation and update status to 1           |

---

## ✅ **Real World Example to Understand Logic**

Assume DB contains:

| Image | Cainz Code | Status |
| ----- | ---------- | ------ |
| A.jpg | 111        | 0      |
| B.jpg | 111        | 0      |
| C.jpg | 222        | 0      |
| D.jpg | 222        | 0      |
| E.jpg | 333        | 0      |

Assume S3 contains:

```
product-images/collected/111/A.jpg   (50 KB)
product-images/collected/111/B.jpg   (50 KB)
product-images/collected/222/C.jpg   (50 KB)
product-images/collected/222/D.jpg   (50 KB)
```

Image `E.jpg` is **missing** from S3.

### ✅ What happens?

✔ unread images = DB entries where status = 0
✔ S3 intersection = A,B,C,D only
❌ Cainz code 333 is incomplete → **skip 333 entirely**

Final ZIP includes only **111 & 222 folders** fully.

ZIP structure:

```
111/product/A.jpg
111/product/B.jpg
222/product/C.jpg
222/product/D.jpg
```

DB update after zipping:

| Image   | New Status                                            |
| ------- | ----------------------------------------------------- |
| A,B,C,D | ✅ 1                                                   |
| E       | ❌ 0 (still unread, will process when E arrives in S3) |

---

## ✅ Timing Logic Example

Assume:

* config.minExecutationTimeInterval = 5 min
* First unread image = uploaded **4 mins ago**
* Total unread size = 80MB (< 100MB)

Decision:
→ **Do not generate ZIP**
→ **Wait until either**
✓ Size ≥ 100MB
or
✓ Oldest image ≥ 10min old

---

## ✅ Size Threshold Example

Assume:

* Total unread images = 1.5 GB (> 1GB)

Decision:
→ Only ZIP the first **1GB worth of image files**
→ Remaining still `status = 0`, processed next run

---

## ✅ Key rule: Never include partial product data

If Cainz product `222` requires 10 images
But only 9 exist in S3
→ Do not include any of the 9
→ Wait for the missing one
→ Protects downstream systems from incomplete datasets

---

## ✅ What Lambda Cron Table Does

`LambdaCron` stores last successful execution timestamp.
Used to filter only **new images since last run**.

---

## ✅ Zip Naming Convention

Example:
`IFI_IMAGE_DATA_MARKETPLACE_20251025_142530.zip`

Meaning:
YYYY = 2025
MM = 10
DD = 25
hhmmss = 14:25:30 (Tokyo time)

Stored at:

```
product-images/validated-zip/send/
```

---

## ✅ Summary Flow Diagram

```
DB (status=0) → Filter unread images → Match with S3 collected folder
                     ↓
          Apply timing + size + completeness rules
                     ↓
          If allowed → Create ZIP (max 1GB)
                     ↓
       Upload ZIP → Update DB status = 1 → Store product image count
```

If conditions NOT met:
→ Do nothing
→ Keep status = 0
→ Try next scheduled run

---

## ✅ Benefits of this Logic

| Feature               | Why it matters                   |
| --------------------- | -------------------------------- |
| Timing handling       | Avoid too frequent ZIP creation  |
| Size threshold        | Avoid too small ZIP files        |
| 1GB max cap           | Avoid exceeding system limits    |
| Complete product rule | Ensures accuracy and consistency |
| Maintenance mode safe | No corruption during downtime    |

---

Certainly. Here is a very clear and simplified explanation of that rule with a real example.

---

### ✅ Key Rule: **Never include partial product data**

Each product (identified by a **cainz product code**) must have **all required images** before being included in the ZIP.

#### Why is this rule required?

Downstream systems expect **complete data** for every product.
If even **one image** is missing, that product is considered **incomplete**.

---

### ✅ Example Scenario

| Product Code | Required Image Count | Images Found in S3 | Include in ZIP? | Reason                 |
| ------------ | -------------------- | ------------------ | --------------- | ---------------------- |
| **111**      | 8 images             | 8 found            | ✅ Yes           | Complete set           |
| **222**      | 10 images            | 9 found            | ❌ No            | Missing 1 → incomplete |
| **333**      | 6 images             | 6 found            | ✅ Yes           | Complete set           |
| **444**      | 12 images            | 0 found            | ❌ No            | No unread images       |

---

### ✅ What the Lambda does in this case

It filters like this:

```js
finalImageKeys = [];
countResult.forEach((product) => {
  if (product.count === expectedCount) {
    finalImageKeys.push(...allImagesOfProduct);
  }
});
```

So only the complete products go into the ZIP.

---

### 🧠 Why important?

Without this rule:

• Product **222** would upload 9 images today
• The missing 1 might upload tomorrow
→ Downstream system receives **partial product data**, causes errors or invalid listings

---

### 📌 Visual Representation

```
product-images/collected/  
  |--- 111/  ✅ complete → ZIP
  |--- 222/  ❌ missing images → WAIT
  |--- 333/  ✅ complete → ZIP
```

---

### ✅ Summary in one sentence

> Only products with all expected images are added to the ZIP. If any image of a product is missing, that entire product is skipped until it becomes complete.

---

Start
  |
  v
Fetch unread product images from database (status = 0)
  |
  v
Check corresponding images in S3 "collected" folder
  |
  v
Group images by cainz product code
  |
  v
For each product:
  |
  v
Is image count complete for this product?
  |                     |
  | Yes                 | No
  v                     v
Add ALL images          Skip product entirely
of this product         (do not add partial images)
to ZIP queue            wait for missing images next run
  |
  v
After checking all products
  |
  v
Is total ZIP size ≤ 1 GB?
  |                     |
  | Yes                 | No
  v                     v
Proceed with ZIP        Stop adding more products
creation                until size limit allows
  |
  v
Create ZIP file with all complete products
  |
  v
Upload ZIP to "validated-zip/send"
  |
  v
Update product images status from 0 -> 1 in database
  |
  v
End


