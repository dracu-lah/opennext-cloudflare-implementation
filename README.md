# 🚀 Vercel to Cloudflare Migration Guide

This guide details the transition of the `my-next-app` project from Vercel to **Cloudflare Workers**, utilizing **OpenNext** for full Next.js feature compatibility.

---

## 📋 Table of Contents

- [1. Infrastructure Setup](#1-infrastructure-setup)
  - [Domain](#domain)
  - [Storage & Caching (R2 & D1)](#storage--caching-r2--d1)
- [2. Project Configuration](#2-project-configuration)
  - [Dependencies](#dependencies)
  - [Git Configuration](#git-configuration)
  - [Wrangler Configuration](#wrangler-configuration-wranglerjsonc)
  - [OpenNext Configuration](#opennext-configuration-opennextconfigts)
- [3. Image Optimization (Dashboard)](#3-image-optimization-dashboard)
  - [Step 1: Page Rule](#step-1-page-rule-cache-images)
  - [Step 2: Transform Rule](#step-2-transform-rule-webp-fallback)
- [4. Deployment Pipeline](#4-deployment-pipeline)
- [📚 References](#references)

---

## 🛠️ 1. Infrastructure Setup

Before modifying the codebase, ensure the following Cloudflare resources are provisioned.

### Domain
*   **Action:** Transfer or point your domain to Cloudflare.
*   **Target:** `example.com`

### Storage & Caching
OpenNext requires specific bindings for incremental caching and revalidation.

#### **A. R2 Bucket (Incremental Cache)**
Used for storing ISR/SSG pages and optimized images.
```bash
pnpx wrangler r2 bucket create my-next-app-next-cache
```

#### **B. D1 Database (Tag Cache)**
Used for on-demand revalidation (`revalidatePath` / `revalidateTag`).
```bash
pnpx wrangler d1 create my-next-app_next_d1_next_cache
```
> [!IMPORTANT]
> Note the `database_id` from the command output; you will need it for the `wrangler.jsonc` file.

---

## ⚙️ 2. Project Configuration

### Dependencies
Install the OpenNext Cloudflare adapter and the Wrangler CLI.

```bash
# Install OpenNext adapter
pnpm install @opennextjs/cloudflare@latest

# Install Wrangler for deployment
pnpm install --save-dev wrangler@latest
```

### Git Configuration
Update `.gitignore` to prevent deployment artifacts from being committed.

```gitignore
# cloudflare / open-next
.open-next/
.wrangler/
```

### Wrangler Configuration (`wrangler.jsonc`)
Create a `wrangler.jsonc` file in your root directory to define the environment and bindings.

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "my-next-app",
  "main": ".open-next/worker.js",
  "compatibility_date": "2024-12-30",
  "compatibility_flags": ["nodejs_compat", "global_fetch_strictly_public"],
  "assets": {
    "directory": ".open-next/assets",
    "binding": "ASSETS",
  },
  "r2_buckets": [
    {
      "binding": "NEXT_INC_CACHE_R2_BUCKET",
      "bucket_name": "my-next-app-next-cache",
    },
  ],
  "services": [
    {
      "binding": "WORKER_SELF_REFERENCE",
      "service": "my-next-app",
    },
  ],
  "durable_objects": {
    "bindings": [
      {
        "name": "NEXT_CACHE_DO_QUEUE",
        "class_name": "DOQueueHandler",
      },
    ],
  },
  "migrations": [
    {
      "tag": "v1",
      "new_sqlite_classes": ["DOQueueHandler"],
    },
  ],
  "d1_databases": [
    {
      "binding": "NEXT_TAG_CACHE_D1",
      "database_name": "my-next-app_next_d1_next_cache",
      "database_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    },
  ],
  "images": {
    "binding": "IMAGES",
  },
}
```

### OpenNext Configuration (`open-next.config.ts`)
Configure OpenNext to utilize the R2 and D1 bindings.

```typescript
import { defineCloudflareConfig } from "@opennextjs/cloudflare";
import r2IncrementalCache from "@opennextjs/cloudflare/overrides/incremental-cache/r2-incremental-cache";
import d1NextTagCache from "@opennextjs/cloudflare/overrides/tag-cache/d1-next-tag-cache";
import doQueue from "@opennextjs/cloudflare/overrides/queue/do-queue";
 
export default defineCloudflareConfig({
  incrementalCache: r2IncrementalCache,
  queue: doQueue,
  // Required for On-demand revalidation (revalidatePath/revalidateTag)
  tagCache: d1NextTagCache,
  // Disable if using Partial Prerendering (PPR)
  enableCacheInterception: true,
});
```

### Package Scripts (`package.json`)
Add the following scripts to your `package.json` to facilitate deployment and preview.

```json
{
  "scripts": {
    "deploy": "opennextjs-cloudflare build && opennextjs-cloudflare deploy",
    "upload": "opennextjs-cloudflare upload",
    "secrets:upload": "wrangler secret bulk .env",
    "preview": "opennextjs-cloudflare build && opennextjs-cloudflare preview"
  }
}
```

---

## 🖼️ 3. Image Optimization (Dashboard)

To support `next/image` effectively on Cloudflare, configure these rules in your Dashboard.

### Step 1: Page Rule (Cache Images)
Caches optimized images at the edge to reduce Worker invocations.

*   **URL:** `example.com/_next/image?*`
*   **Setting:** Cache Level → **Cache Everything**
*   **Edge Cache TTL:** 7 Days (or preferred duration)

### Step 2: Transform Rule (WebP Fallback)
Ensures compatibility for browsers that do not support WebP.

1.  Navigate to **Rules > Transform Rules > URL Rewrite Rules**.
2.  Click **Create Rule**.
3.  **Expression:**
    ```
    (http.request.uri.path eq "/_next/image" and not any(http.request.headers["accept"][*] contains "image/webp"))
    ```
4.  **Rewrite Query to:**
    ```
    concat(http.request.uri.query, "&webp=false")
    ```

---

## 🚀 4. Deployment Pipeline

The deployment is automated via the Cloudflare Pages/Workers GitHub integration.

### Setup Steps
1.  **Project Connection:** In the Cloudflare Dashboard, select "Create application" and connect your GitHub repository.
2.  **Environment Variables:** Add all required environment variables in the Dashboard settings during the setup.
3.  **Build Settings:**
    *   **Build Command:** `pnpm run deploy`
    *   **Deployment Script:** Ensure your `package.json` contains a `deploy` script that triggers `opennextjs-cloudflare`.

---

## 💰 Cost & Performance Considerations

While Cloudflare offers a generous free tier, specific Next.js features can impact usage limits.

### 1. Image Optimization & Worker CPU
The default Next.js image optimization (`/_next/image`) runs inside the Worker using a WebAssembly version of `sharp`.
*   **Risk:** Transforming images is CPU-intensive. If images are not effectively cached (see **Section 3**), every image load will invoke the Worker and consume significant CPU time.
*   **Mitigation:** Ensure the "Cache Everything" page rule is active and effectively hitting the cache to minimize Worker invocations.

### 2. Incremental Static Regeneration (ISR)
*   **Impact:** Frequent use of `revalidatePath` or `revalidateTag` triggers read/write operations on **D1** (for tags) and **R2** (for pages/data).
*   **Cost:** While inexpensive, high-frequency revalidation on high-traffic sites can accumulate costs for database rows read/written and storage operations.

---

## 📚 References

*   [OpenNext Cloudflare Documentation](https://opennext.js.org/cloudflare)
*   [Next.js Image Optimization with Cloudflare](https://st4ng.medium.com/how-to-use-next-js-image-optimization-with-cloudflare-569da7b3ddc6)
*   [Video Tutorial Reference](https://www.youtube.com/watch?v=Ja_ycZsYH0o)

