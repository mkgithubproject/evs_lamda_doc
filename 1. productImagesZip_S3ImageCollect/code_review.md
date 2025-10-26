The code you shared is large and complex. A literal **line-by-line** rewrite with explanation inserted into every single line would result in a very long and unreadable answer here. The most professional way is to break it into **logical blocks** and then explain each block **line by line inside that block**, with examples where needed. This allows full clarity without overwhelming fragmentation.

Below is **Part 1**. Once you review and confirm the format is correct, the remaining sections can be delivered in the same style until the full file is covered.

---

# ✅ Code Explanation: Section-by-Section, Line-By-Line

**(Part 1: Setup, DB connection, Repository Init, Maintenance Mode Check)**

---

## ✦ Imports and Initial Configuration

```js
// *********************************************
// This function is responsible to gather images
// from "collected" folder on S3 bucket and create
// a zip file to "send" folder inside the 
// "validated-zip" folder.
// *********************************************
```

Comment describing functionality:
Collects product images → creates zip → uploads to validated-zip/send folder.

```js
var AWS = require("aws-sdk");
```

Import AWS SDK for S3 and Secrets Manager.

```js
var typeorm = require("typeorm");
const { In } = require("typeorm");
```

Import TypeORM to interact with PostgreSQL and `In` operator for filtering in DB queries.

```js
const archiver = require("archiver");
```

Used to create ZIP files in memory.

```js
const { SecretsManagerClient, GetSecretValueCommand } =
require("@aws-sdk/client-secrets-manager");
```

AWS Secrets Manager for secure DB connection details.

```js
const config = require("./config.js");
```

Local configuration file: AWS S3 paths, thresholds, bucket name, etc.

```js
const stream = require("stream");
```

Used to pipe ZIP data.

```js
const LambdaCron = require("./entity/LambdaCron.js");
const ProductImage = require("./entity/ProductImage.js");
const { developerLog, createDeveloperLogStream } = require("./developerLog.js");
const SystemConfig = require("./entity/SystemConfig.js");
const Product = require("./entity/Product.js");
```

Entity imports for interacting with PostgreSQL tables.

Example meaning:
`ProductImage` table helps determine unread images.

---

## ✦ AWS SDK Setup

```js
AWS.config.update({
  accessKeyId: config.awsKeyId,
  secretAccessKey: config.secretAccessKey,
  region: config.awsRegion
});
```

Configures AWS SDK with credentials and region.

Example:
Region `ap-northeast-1`, Bucket: `marketplace-images`

---

## ✦ Local Default DB Config

```js
let DBCondfig = {
  type: "postgres",
  host: config.host,
  port: config.port,
  username: config.userName,
  password: config.password,
  database: "postgres",
  synchronize: false,
  entities: [LambdaCron, ProductImage, SystemConfig, Product],
};
```

Temporary local DB config that **will later be overwritten** using Secrets Manager.

---

```js
let dataSource = new typeorm.DataSource(DBCondfig);
```

Initializing TypeORM datasource reference.

---

### ✦ Function: DBConnection()

Fetches actual DB credentials from Secrets Manager and connects.

```js
const DBConnection = async () => {
  await developerLog("secret db start");
```

Log that DB connection process started.

```js
  const secret_name = config.secretName;
```

Name of secret to fetch from AWS.

Example:
`prod/rds/postgres/marketplace`

```js
  const client = new SecretsManagerClient({
    region: config.awsRegion,
  });
```

Create Secrets Manager client.

```js
  let secretResponse = await client.send(
    new GetSecretValueCommand({
      SecretId: secret_name,
      VersionStage: "AWSCURRENT",
    })
  );
```

Fetch secret JSON from AWS Secrets Manager.
VersionStage `AWSCURRENT` ensures latest credentials.

```js
  const secret = JSON.parse(secretResponse.SecretString);
```

Parse JSON secret.

Example JSON Structure:

```json
{
  "username": "admin",
  "password": "pwd123",
  "host": "db.abc123.rds.amazonaws.com",
  "dbname": "marketplace",
  "port": 5432
}
```

```js
  DBCondfig.host = secret.host;
  DBCondfig.username = secret.username;
  DBCondfig.password = secret.password;
  DBCondfig.database = secret.dbname;
  DBCondfig.port = secret.port;
```

Override local config with real RDS DB details.

```js
  const AppDataSource = new typeorm.DataSource(DBCondfig);
  const connection = await AppDataSource.initialize();
  return connection;
};
```

Create real DataSource → connect to DB → return connection.

---

## ✦ Default Lambda Response Template

```js
let response = {
  statusCode: 200,
  body: JSON.stringify('File Upload Successfully!'),
};
```

Response returned when Lambda succeeds.

---

## ✦ Main Lambda Handler (Execution Entry Point)

```js
exports.handler = async (event) => {
  await createDeveloperLogStream()
```

Prepare logging output for developer debugging.

```js
  dataSource = await DBConnection();
```

Establish PostgreSQL DB connection.

```js
  try {
```

Start try block for execution.

---

## ✦ Initialize S3 Client

```js
    const s3 = new AWS.S3();
```

S3 client will be used to list and download images.

---

## ✦ Date and Repository Initialization

```js
    const currentDate = new Date();
```

Current timestamp used to compare image timestamps.

```js
    const imageName = [];
    const unreadImage = [];
```

Arrays to store unread file names.

```js
    const lambdaCronRepository = dataSource.getRepository(LambdaCron);
    const productImageRepository = dataSource.getRepository(ProductImage);
    const systemConfigRepository = dataSource.getRepository(SystemConfig);
    const productRepository = dataSource.getRepository(Product);
```

Repository objects to perform CRUD in PostgreSQL.

---

## ✦ Maintenance Mode Check

This prevents pipeline from running when system is paused.

```js
    const configNames = [
      "MAINTENANCE_MODE",
      "ENABLE_LAMBDA_IN_MAINTENANCE_MODE",
    ];
```

Two flags:

| Setting                           | Purpose                           |
| --------------------------------- | --------------------------------- |
| MAINTENANCE_MODE                  | Whole system locked               |
| ENABLE_LAMBDA_IN_MAINTENANCE_MODE | Only allow lambda if set to **1** |

```js
    const configs = await systemConfigRepository.find({
      where: { name: In(configNames) }
    });
```

Query both settings from DB.

```js
    const configMap = configs.reduce((acc, config) => {
      acc[config.name] = config;
      return acc;
    }, {});
```

Create key-value map for easier reading.

Example result:

```js
configMap = {
  "MAINTENANCE_MODE": {value: "true"},
  "ENABLE_LAMBDA_IN_MAINTENANCE_MODE": {value: "0"}
}
```

```js
    const maintenanceMode = configMap["MAINTENANCE_MODE"];
    const enableLambdaInMaintenanceMode = configMap["ENABLE_LAMBDA_IN_MAINTENANCE_MODE"];
```

```js
    if (maintenanceMode?.value == "true" && enableLambdaInMaintenanceMode.value === "0") {
      console.log("System is in maintenance mode");
      return {
        statusCode: 503,
        body: JSON.stringify("System is in maintenance mode")
      }
    }
```

❌ Condition means STOP EXECUTION
Maintenance active + Lambda not allowed → return 503 error

---

✅ **End of Part 1**

---

# To Continue

Your file still contains major sections:

| Part | Topic                                              |
| ---- | -------------------------------------------------- |
| 2    | Fetch unread images & find valid full product sets |
| 3    | Create ZIP archive and add images correctly        |
| 4    | Upload ZIP to send folder in S3                    |
| 5    | Update DB product image count                      |
| 6    | Mark ProductImage rows as status=1                 |
| 7    | Error handling, cleanup                            |

---
# ✅ Continued — Full, detailed explanation (Option A)

Below is **Part 2 → Part 7**, maintained in the same block-wise, line-by-line style with examples and clarifications. Each code block is explained in context and illustrated with concrete example values where helpful.

---

## Part 2 — Fetch unread images & determine valid, complete product sets

### Code + explanation

```js
const lambdaCronData = await lambdaCronRepository.find({
  order: {
    id: "desc"
  }
});
```

Fetch all rows from the `LambdaCron` table ordered by `id` descending.
The code assumes the most recent row is at index `0`. This table stores the `last_execution_time`.
**Example:** `lambdaCronData[0].last_execution_time = 2025-10-25T08:00:00Z`.

```js
const productImage = await productImageRepository.find({
  where: {
    status: 0
  }
});
```

Fetch all `ProductImage` records that are unread (`status = 0`). These represent images the system has not yet processed.
**Example result:**

```
[
  { cainz_code: "111", image_name: "111_1.jpg", status:0 },
  { cainz_code: "111", image_name: "111_2.jpg", status:0 },
  { cainz_code: "222", image_name: "222_1.jpg", status:0 }
]
```

```js
productImage.forEach((image) => {
  const imageLocation = config.mpSideDir + image.cainz_code + "/" + image.image_name;
  unreadImage.push(imageLocation);
})
```

Build `unreadImage` as a list of full S3 keys expected to exist in `product-images/collected/` (where `config.mpSideDir` holds that prefix).
**If** `config.mpSideDir = 'product-images/collected/'` then for the example above `unreadImage` becomes:

```
[
 'product-images/collected/111/111_1.jpg',
 'product-images/collected/111/111_2.jpg',
 'product-images/collected/222/222_1.jpg'
]
```

```js
let executationTime = "";
let lastExecutationTime = "";
let lambdaCronId;
let size = 0;
let imageCountResult = []
```

Initialize variables:

* `executationTime` — this run's timestamp (will set later)
* `lastExecutationTime` — timestamp of previous successful run from DB
* `lambdaCronId` — id of latest LambdaCron row if present
* `size` — accumulator used to stop adding images beyond configured `sizeThreshold`
* `imageCountResult` — collected counts to later update product image counters

```js
if (lambdaCronData.length == 0) {
  lastExecutationTime = currentDate;
  await lambdaCronRepository.save({ last_execution_time: currentDate });
} else {
  lastExecutationTime = lambdaCronData[0].last_execution_time;
  lambdaCronId = lambdaCronData[0].id;
}
```

If no `LambdaCron` rows exist, create one with the current time and use that as `lastExecutationTime`. Otherwise use the most recent stored `last_execution_time`. This ensures only new images (after last run) will be considered when applying `LastModified > lastExecutationTime` filter later.

**Example:** If `lastExecutationTime = 2025-10-25T08:00:00Z`, images uploaded after that timestamp are eligible (subject to other rules).

---

### Timing and size logic: selecting S3 objects

This large chunk controls when to proceed and which S3 keys to include.

```js
const readValidFileParams = {
  Bucket: config.bucketName,
  Prefix: config.mpSideDir
};
```

Parameters to list objects under the `product-images/collected/` prefix.

```js
let currentDateUTC = new Date();
let timeThreshold = new Date(new Date().getTime() - 10 * 60 * 1000); // 10 minutes ago
```

* `currentDateUTC` — now
* `timeThreshold` — 10 minutes before now; used to check whether the earliest unread image is older than 10 minutes.

```js
let imageKeys;
let countResult = [];
if (config.minExecutationTimeInterval < config.maxExecutationTimeInterval) {
  imageKeys = await s3.listObjectsV2(readValidFileParams).promise()
    .then((data) => {
      executationTime = new Date();
      const unreadFilter = data.Contents.filter((item) => {
        return unreadImage.includes(item.Key);
      })
```

List S3 objects under the prefix and keep only those whose keys are present in `unreadImage` (intersection). `unreadFilter` is S3 metadata (Key, Size, LastModified) for matching objects.

**Example:** S3 returned `Contents` with keys:

```
product-images/collected/111/111_1.jpg  (Size 50KB, LastModified 2025-10-25T07:55:00Z)
product-images/collected/111/111_2.jpg  (Size 50KB, LastModified 2025-10-25T07:56:00Z)
product-images/collected/222/222_1.jpg  (Size 1MB, LastModified 2025-10-25T08:05:00Z)
product-images/collected/333/333_1.jpg  (not in unreadImage)
```

`unreadFilter` will include the first three only.

```js
const imagesKeyArr = [];
const cainzProductCodes = new Set();
unreadFilter.map((item, index) => {
  let itemArr = item.Key.split("/");
  imagesKeyArr.push(itemArr[2]);
  cainzProductCodes.add(itemArr[2]);
});
const cainzProductCodesArr = Array.from(cainzProductCodes);
cainzProductCodesArr.map((item) => {
  let regex = new RegExp(item, "g");
  const length = imagesKeyArr.toString().match(regex).length;
  countResult.push({
    cainzCode: item,
    count: length,
  });
});
```

This block constructs a per-product count of how many unread images exist in S3 for each `cainzCode`. `imagesKeyArr` holds code for each file (position `2` after splitting by `/`), producing counts such as:

```
countResult = [
  { cainzCode: "111", count: 2 },
  { cainzCode: "222", count: 1 }
]
```

```js
let lastModificationDate = currentDateUTC;
let totalSize = 0;
let imageKeys = [];
unreadFilter.forEach((data) => {
  if (lastModificationDate > data.LastModified) {
    lastModificationDate = data.LastModified;
  }
  totalSize = totalSize + data.Size;
})
```

Compute:

* `lastModificationDate` — the earliest LastModified among unread S3 objects (oldest image time).
* `totalSize` — sum of sizes of all matched unread objects.

**Example:** If images have LastModified times 07:55, 07:56, 08:05 then `lastModificationDate = 07:55`. If sizes are 50KB, 50KB, 1MB then `totalSize ≈ 1.1MB`.

```js
if (totalSize < config.minSizeThreshold && lastModificationDate <= timeThreshold || totalSize > config.minSizeThreshold) {
  const filterData = unreadFilter.filter((item) => {
    if (size < config.sizeThreshold) {
      size = size + item.Size;
    }
    return item.LastModified > lastExecutationTime && item.Key != config.mpSideDir && size < config.sizeThreshold;
  });
  imageKeys = filterData.map(obj => obj.Key);
}
```

This central condition governs whether images are selected:

* If `totalSize` is less than `config.minSizeThreshold` **and** the earliest unread image (`lastModificationDate`) is older than `timeThreshold` (10 minutes ago), then select images.
* If `totalSize` is greater than `config.minSizeThreshold`, then select images immediately.
* While selecting, accumulate `size` and stop adding more S3 items once `size` reaches `config.sizeThreshold`. This ensures the selected set does not exceed the maximum zip size.

**Example with values:**

* `config.minSizeThreshold = 100 * 1024 * 1024` (100 MB)
* `config.sizeThreshold = 1 * 1024 * 1024 * 1024` (1 GB)

For `totalSize = 1.1MB (<100MB)` and `lastModificationDate = 07:55`, if current time is 08:05 then `lastModificationDate <= timeThreshold` (older than 10 minutes), so selection proceeds. Articles uploaded at 08:00 are 5 minutes old and would not allow selection unless `totalSize > minSizeThreshold`.

`filterData` further restricts to only images with `LastModified > lastExecutationTime` (new since last processing) and avoids selecting folder prefix itself.

`size` accumulation occurs in the `filter` callback. The code relies on `unreadFilter` order; the order affects which images get included when approaching the `sizeThreshold`. This is a potential source of non-determinism if S3 returns Contents in non-stable order.

---

### Ensuring full product completeness (no partial products)

```js
const finalImageKeys = [];
countResult.forEach((data) => {
  let regex = new RegExp("/" + data.cainzCode + "/", "g");
  const filterData = imageKeys.filter((data) => {
    return data.toString().match(regex);
  });
  if (data.count == filterData.length) {
    finalImageKeys.push(...filterData);

    imageCountResult.push({
      cainzCode: data.cainzCode,
      count: filterData.length
    });
  }
});
```

This block enforces the rule **never include a partial product**:

1. `countResult` contains how many unread images exist for a product in S3 (call this `S3Count`).
2. `imageKeys` is the set selected under size/time rules and filtered by `LastModified > lastExecutationTime`. `filterData` are the selected keys for this product (call its length `SelectedCount`).
3. If `S3Count === SelectedCount`, the product is complete **relative to the matched unread set** and all selected images for that product are added to `finalImageKeys`. Otherwise the product is skipped entirely.

**Concrete example:**

* `S3Count` for product `222` = 10 (S3 has 10 unread images present)
* `SelectedCount` = 9 (size limit cut off one image or lastExecutationTime filtered one)
  Because `10 !== 9`, the product `222` is skipped to avoid partial inclusion.

`imageCountResult` stores product counts for later updating the `Product` table's `image_count` field.

---

### End of selection logic

```js
return finalImageKeys;
```

The promise returns the fully validated list of S3 keys to include in the ZIP. If empty, the calling code resolves `null` and the Lambda ends with `"No new file found..."`.

---

## Part 3 — Create ZIP archive and append images

### Code + explanation

```js
const archive = archiver('zip', { zlib: { level: 9 } });
const outputStream = new stream.PassThrough();

archive.on('error', (error) => reject(error));
outputStream.on('error', (error) => reject(error));
archive.pipe(outputStream);
```

Create an in-memory zip stream using `archiver`. `PassThrough` is used to hold result in memory and later pass as `Body` to S3 upload. Listeners propagate archive and stream errors to reject the promise.

**Note:** Using in-memory streams is fine for moderate sizes. For large archives approach streaming to a file or multi-part S3 upload.

```js
const getObjectPromises = imageKeys.map(key => {
  const params = {
    Bucket: config.bucketName,
    Key: key,
  };
  return s3.getObject(params).promise();
});

Promise.all(getObjectPromises)
  .then(objects => {
    objects.forEach((object, index) => {
      const key = imageKeys[index];
      const fileName = key.split('/').pop();
      const imageName = fileName.split("_")[0] + "/product/" + fileName;
      const s3Stream = object.Body;
      archive.append(s3Stream, { name: imageName });
    });

    archive.finalize();
    resolve(outputStream);
  });
```

* `getObjectPromises` downloads each S3 object into memory (object.Body).
* After download completes, iterate in the same order as `imageKeys` and append each object stream to the archive.
* Each file is stored in the archive under a path `"{productCode}/product/{fileName}"` where `fileName.split("_")[0]` extracts the prefix (often product code).
* `archive.finalize()` closes the archive. `resolve(outputStream)` returns the zip stream to the main execution flow.

**Example mapping:**
If `key = product-images/collected/111/111_1.jpg`, then:

* `fileName = '111_1.jpg'`
* `imageName = '111/product/111_1.jpg'` inside zip

This structure keeps files grouped by product inside the zip.

**Important considerations:**

* `Promise.all` loads all images into memory concurrently. For many large images this might exhaust memory. A streaming approach (append each `s3.getObject().createReadStream()` as it resolves) would be more memory-efficient.
* Order of addition matters if downstream expects deterministic ordering.

---

## Part 4 — Upload ZIP to S3 send folder and naming

### Code + explanation

```js
const newCurrentDate = new Date(new Date().toLocaleString('ja', { timeZone: 'Asia/Tokyo' }));
const year = newCurrentDate.getFullYear();
const month = (newCurrentDate.getMonth() + 1).toString().padStart(2, '0');
const day = newCurrentDate.getDate().toString().padStart(2, '0');
const seconds = newCurrentDate.getSeconds().toString().padStart(2, '0');
const minutes = newCurrentDate.getMinutes().toString().padStart(2, '0');
const hour = newCurrentDate.getHours().toString().padStart(2, '0');

const validZipFileName = "IFI_IMAGE_DATA_MARKETPLACE_" + year + month + day + hour + minutes + seconds + ".zip";
```

Build a timestamped filename using Tokyo timezone. The zip will be named e.g. `IFI_IMAGE_DATA_MARKETPLACE_20251025HHMMSS.zip`.

```js
await s3.putObject({ Bucket: config.bucketName, Key: config.zipFileDestination }, function (err, data) {
  if (err) {
    console.error('Error creating directory:', err);
  } else {
    console.log('Directory created successfully');
  }
}).promise();
```

This call attempts to ensure the destination prefix exists by writing a zero-byte object at that key (some teams use S3 "folders" as zero-byte keys). `config.zipFileDestination` likely equals `'product-images/validated-zip/send/'`.

```js
const zipParams = {
  Bucket: config.bucketName,
  Key: config.zipFileDestination + validZipFileName,
  Body: zipStream
};
await s3.upload(zipParams).promise();
```

Upload the zip stream to S3. `Body` is the `PassThrough` stream returned earlier. The object will be stored at:

```
product-images/validated-zip/send/IFI_IMAGE_DATA_MARKETPLACE_20251025HHMMSS.zip
```

**Important note:** `s3.upload` internally can handle streaming uploads; ensure Node environment does not buffer excessively.

---

## Part 5 — Update Product table image counts

```js
if (imageCountResult.length > 0) {
  await Promise.all(imageCountResult.map((item) => {
    return productRepository.update({ daihyo_jan_cd: item?.cainzCode }, { image_count: item?.count });
  }));
}
```

`imageCountResult` contains `{ cainzCode, count }` for each included product. This updates the `Product` table where `daihyo_jan_cd` is the product code, setting `image_count` to the number included in the zip.

**Example:** For `imageCountResult = [{cainzCode:'111', count:2}]`, `Product` row with `daihyo_jan_cd = '111'` will receive `image_count = 2`.

---

## Part 6 — Mark processed ProductImage rows as read (`status = 1`)

```js
const readedImages = productImage.map((item) => {
  if (imageName.includes(item.image_name)) {
    item.status = 1;
  }
  return item;
})
const updateStatus = await productImageRepository.save(readedImages);
```

* `productImage` earlier was the DB result of all `status=0` rows.
* `imageName` array stores just the filenames added to the zip (pushed earlier).
* For each `ProductImage` row, if its `image_name` was included in the zip, set `status = 1` (read). Save all rows back; TypeORM detects changed entities and performs updates.

**Careful points:**

* This approach sets `status = 1` by checking filenames only. If two different products share the same filename across codes, this could incorrectly mark an image as read. The safer check is to match both `cainz_code` and `image_name` by S3 key equality.
* Race condition: If new images are added between listing S3 and saving DB, those new images will still have `status=0` because `imageName` uses image names from `imageKeys`.

**Example:** If `imageName = ['111_1.jpg','111_2.jpg']`, the `ProductImage` entries for those files receive `status = 1`.

---

## Part 7 — Error handling and cleanup

```js
catch (error) {
  console.log("Error during lambda Execution : " + error);
  const response = {
    statusCode: 500,
    body: JSON.stringify('internal server '),
  };
  return response;
}
finally {
  dataSource.destroy();
}
```

* `catch` logs the error and returns `500`.
* `finally` always destroys the DB `dataSource`, releasing connection pools.

**Important operational notes:**

* If the function fails after zipping but before updating DB and uploading zip, data integrity issues could arise. Implement idempotency or transactional markers if possible. For example, write `LambdaCron` last only when DB status updates succeeded.
* `dataSource.destroy()` in `finally` ensures connections are closed even on failure. Avoid leaving the process alive.

---

## Additional examples and edge-case clarifications

1. **Missing image in S3 for a product**

   * DB has 3 images for product `X`. S3 has only 2. `S3Count = 2`. If the code expects 3 (from somewhere else), product is skipped. In this implementation the code uses the unread S3 counts to compare selected counts, not an external expected count value. That means the definition of "complete" is relative to what `unreadFilter` reported; completeness is judged by whether the selection equals that S3 count. If an *absolute* number of expected images per product should be used, the code needs a lookup (for example `Product.expected_image_count`) — currently that is not present.

2. **Partial selection due to size threshold**

   * Suppose product `111` has 100 images totaling 900MB and product `222` has 2 images totaling 200MB; `sizeThreshold`=1GB. If `111` is processed first and consumes 900MB leaving 100MB free, product `222` has 200MB and cannot be partially included (its 2 images would exceed size). The code will include product `111` only and skip `222`. The order in `unreadFilter` determines which products get included first.

3. **Time-based behavior**

   * If `minExecutationTimeInterval < maxExecutationTimeInterval` branch is used, the code applies the older-time condition; else it takes an alternate branch implementing slightly different filtering. The intent is to use different selection when run frequency is within certain bounds. The code paths are mostly duplicated; consolidation would reduce confusion.

4. **Memory usage**

   * Current implementation downloads all selected images into memory (`s3.getObject(params).promise()` returns entire body). For large image sets approaching GBs, this will cause memory pressure. Consider streaming through `s3.getObject(params).createReadStream()` directly into `archive.append()` without waiting for `Promise.all`, or upload the zip using multipart upload streaming.

5. **Race conditions**

   * Between listing S3 and calling `getObject`, new files may appear or some files may be deleted. The Lambda updates DB after upload; if failures happen between upload and DB update, duplicates or missed status updates can result. Consider atomic markers or a two-phase commit style (for example write a manifest file into S3 then update DB after upload confirmed).

---

## Suggested improvements (brief, actionable)

1. Match DB and S3 keys by both `cainz_code` and `image_name` to avoid filename collision mistakes.
2. Replace `Promise.all(getObjectPromises)` with a streaming approach to reduce memory consumption.
3. Persist `executationTime` (`lambdaCronRepository.update`) **after** successful upload and DB status updates to ensure last execution reflects completed work.
4. Provide deterministic ordering for `unreadFilter` (for example sort by `LastModified` ascending) before applying `size` accumulation, so product selection is predictable.
5. If there is an expected image count per product, read that from `Product` table (e.g., `expected_image_count`) and use it to check completeness instead of using the S3-only `countResult`.

---



