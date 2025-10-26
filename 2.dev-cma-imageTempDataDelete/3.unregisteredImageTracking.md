Perfect ✅
You have shared the full code for:

### ✅ Lambda 2: **evdev-cma-unregisteredImageTracking**

Now I will explain this Lambda like a real-time story with a clear example — very easy to understand.
No confusion ✅

---

# ✅ What This Lambda Does — In Simple Words

### 🎯 Goal:

Check **all products whose images were missing earlier**, and try to find those images on Imgix.

📌 It keeps checking:

* Every 3 hours (triggered by EventBridge)
* For **up to 24 hours**
* Max processing = **40,000** records per run

---

## ✅ Step-by-Step Flow

```
Get all products from ec_unregistered_images table where status = 0
│
├─ If image is found on Imgix ✅
│     ├─ Remove record from unregistered table ❌
│     └─ Update product table → image_flg_ec = 1 ✅
│
└─ If image still not found ⛔
      ├─ Keep retrying until record becomes 24 hours old
      └─ After 24 hours:
            ├─ Mark record as status = 1 (permanent error)
            └─ Insert activity log (for admin visibility)
```

---

## ✅ With Example (Super Clear)

Let’s assume the table contains these:

| ID | Cainz Code | Created At   | Status | Comment           |
| -- | ---------- | ------------ | ------ | ----------------- |
| 1  | 111        | 3 hours old  | 0      | retry image check |
| 2  | 222        | 10 hours old | 0      | retry image check |
| 3  | 333        | 26 hours old | 0      | already too old   |

---

### 🕵️ Imgix Checks:

| Code    | Imgix Response                  | Action Lambda Takes                                |
| ------- | ------------------------------- | -------------------------------------------------- |
| **111** | ✅ Found image                   | ✅ Delete row from table + Update product flag to 1 |
| **222** | ❌ Missing                       | ⏳ Keep record in table (retry later)               |
| **333** | ❌ Missing & Expired 24hr window | 🔴 Mark error + Add to Activity Log                |

✅ After execution:

| Code  | What happens                                    |
| ----- | ----------------------------------------------- |
| 111 ✅ | Image found successfully → remove from tracking |
| 222 ⏳ | Still tracking — retry in next run              |
| 333 ❌ | Permanently failed & logged as error            |

---

## ✅ How It Prevents Overload

| Feature                           | Limit                                    |
| --------------------------------- | ---------------------------------------- |
| Max items processed per execution | **40,000**                               |
| Requests per batch                | divided into **1000** concurrency groups |

So performance remains stable ✅

---

## ✅ Why Status Values Matter

| Status | Meaning                                        |
| ------ | ---------------------------------------------- |
| 0      | Pending (waiting for image to appear on Imgix) |
| 1      | Failed after 24 hour wait ✅ Logged             |

✅ Only status = 0 are processed
✅ Status = 1 are **not retried**

---

## ✅ Database Updates Summary

| When an image is found ✅             |
| ------------------------------------ |
| Remove from `ec_unregistered_images` |
| Set `ec_product.image_flg_ec = 1`    |

| When image missing after 24 hours ❌  |
| ------------------------------------ |
| Update `status = 1`                  |
| Insert error entry in `activity_log` |

---

## ✅ Relationship Between Both Lambdas

| Lambda Name                                 | What It Does                                               |
| ------------------------------------------- | ---------------------------------------------------------- |
| **1️⃣ evdev-cma-imageTempDataDelete**       | Finds missing image products and ADD to unregistered table |
| **2️⃣ evdev-cma-unregisteredImageTracking** | Keeps checking on Imgix and REMOVE/LOG results             |

✅ A smooth two-step verification + recovery pipeline

---

## ✅ Final Visual Flowchart

```
Start
 │
 ▼
Fetch up to 40k unregistered images (status=0)
 │
 ▼
Check each image on Imgix
 │
 ├─ ✅ Image Found
 │       ├─ Delete row from unregistered table
 │       └─ Update product.image_flg_ec = 1
 │
 └─ ⛔ Image Not Found
         │
         ├─ If < 24 hrs → retry next run
         │
         └─ If > 24 hrs
              ├─ update status = 1
              └─ insert activity log
 │
 ▼
Store last execution time in system_config
 │
 ▼
END ✅
```

---

## ✅ Now Tell Me…

Would you like me to:

✅ Rewrite this code cleaner and more optimized?
✅ Add comments for every major action?
✅ Draw a high-quality diagram you can use in documentation?
✅ Explain index and optimization to avoid DB load issues?

Just say what you want next 😄
