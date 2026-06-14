# Gitflow Workflow

## Branch Structure

```
master
└── dev
    └── feature/your-feature
```

---

## Workflow

### 1. Start a Feature

Branch off `dev`:

```bash
git checkout dev
git pull
git checkout -b feature/your-feature
```

---

### 2. Open a PR → dev

Triggers the Jenkins multibranch pipeline on the feature branch:

- Runs `npm run build:dev` and packages the artifact:
  ```
  my-app-1.2.3-a3f91bc-dev.zip
  ```
- Deploys to Lambda (dev environment)
- Uploads artifact to S3 (dev bucket)
- PR is blocked from merging until build passes

---

### 3. Manual Validation

Test the deployed dev Lambda. Once satisfied, approve and merge the PR into `dev`.

---

### 4. Merge to `dev`

Jenkins detects the merge and runs the `dev` branch pipeline:

- Runs `npm run build:qa` and packages a fresh artifact:
  ```
  my-app-1.2.3-a3f91bc-qa.zip
  ```
- Uploads artifact to S3 (QA bucket)
-  manually update the Lambda function from S3, selecting the artifact by version and SHA
- No automated Lambda update

---

### 5. Merge `dev` → `master`

Jenkins detects the merge and runs the `master` branch pipeline:

- Runs `npm run build:prod` and packages a fresh artifact:
  ```
  my-app-1.2.3-a3f91bc-prod.zip
  ```
- Uploads artifact to S3 (prod bucket)
- Prod Lambda is manually updated from S3

---

## Environment Matrix

| Event | Build Command | Artifact | Lambda Deploy | S3 Upload |
|---|---|---|---|---|
| PR opened (feature branch) | `npm run build:dev` | `app-1.2.3-a3f91bc-dev.zip` | ✅ automated (dev) | ✅ dev |
| Merge to `dev` | `npm run build:qa` | `app-1.2.3-a3f91bc-qa.zip` | ❌ manual handoff | ✅ QA |
| Merge to `master` | `npm run build:prod` | `app-1.2.3-a3f91bc-prod.zip` | ❌ manual handoff | ✅ prod |

---

## Notes

- Each environment gets a dedicated build with its own environment variables set by npm run build:dev|qa|prod
  - npm run build will set the version number to local-timestamp when run locally
- QA is a pipeline stage, not a branch — there is no `qa` branch in this workflow
- The artifact version and SHA provide traceability between what passed the PR gate and what QA or prod is testing
- The previous workflow used branch names as environment signals and had no build gates; environment promotion is now driven by the merge chain