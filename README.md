# k8e_manifest


# 🛠 GitHub Actions Workflow (Cross-Repo Manifest Update)

✅ **Prerequisites**:
1. Personal Access Token (PAT) with repo scope stored in your main repo as a secret:
 - Secret name: `TARGET_REPO_TOKEN`

2. Variables to set: (Optional)
 - Your target repo name `(e.g., your-username/k8s-manifests)`
 - Path to the manifest file `(e.g., deployments/my-app.yaml)`

```bash
name: Build, Push Docker Image, and Update External K8s Manifest

on:
  push:
    branches:
      - main

jobs:
  build-and-update-manifest:
    runs-on: ubuntu-latest

    env:
      IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/your-image-name
      TARGET_REPO: your-username/k8s-manifests
      MANIFEST_PATH: deployments/my-app.yaml

    steps:
    - name: Checkout source repo
      uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Log in to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}

    - name: Generate Docker image tag
      id: date
      run: echo "TAG=$(date -u +'%Y%m%d-%H%M%S')" >> $GITHUB_ENV

    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: |
          ${{ env.IMAGE_NAME }}:${{ env.TAG }}
          ${{ env.IMAGE_NAME }}:latest

    - name: Clone target repo (K8s manifests)
      run: |
        git clone https://x-access-token:${{ secrets.TARGET_REPO_TOKEN }}@github.com/${{ env.TARGET_REPO }} target-manifests

    - name: Update image tag in manifest
      run: |
        sed -i "s|${{ env.IMAGE_NAME }}:.*|${{ env.IMAGE_NAME }}:${{ env.TAG }}|g" target-manifests/${{ env.MANIFEST_PATH }}

    - name: Commit and push updated manifest
      run: |
        cd target-manifests
        git config user.name "github-actions"
        git config user.email "github-actions@github.com"
        git add ${{ env.MANIFEST_PATH }}
        git commit -m "Update image to ${{ env.IMAGE_NAME }}:${{ env.TAG }}"
        git push
```


# Step-by-Step: Create TARGET_REPO_TOKEN

1. Create a Personal Access Token
 a) Go to GitHub `Developer Settings` → `Personal Access Tokens`.
 b) Click `"Fine-grained tokens"` or `"Tokens (classic)"` → `Generate new token`.
 c) Give it a name like `GitHub Actions Cross-Repo Push`.
 d) Set an expiration `(recommended: 90 days or more)`.
 e) Under `Repository Access`, choose:
    - Only select `repositories` → choose your `manifest repo` `(e.g., your-username/k8s-manifests)`

 f) Under Permissions, enable: 
    - `Contents → Read and Write`

2. Copy the Token
  You’ll see the token once — copy it securely.


3. Store as GitHub Secret
 - In the main repository (where your workflow runs):
 - Go to `Settings` → `Secrets` and `variables` → `Actions`.
 - Click `New repository secret`.
 - Name it: `TARGET_REPO_TOKEN`
 - Paste your token value

That's it! Now your GitHub Actions workflow will be able to:
 - Clone the manifest repo
 - Modify the Kubernetes YAML file
 - Push changes back using the token's credentials

 ✅ # Overview
**Docker Image Repo**:  is a repo where the GitHub Actions workflow runs.
**K8s Manifests Repo**: is another repo in the same GitHub account.

You’ll use a Personal Access Token (PAT) stored as `TARGET_REPO_TOKEN` to allow your workflow to push to the `manifests repo`

🔐 **PAT Requirements**
When creating the token, ensure:
 - It belongs to your GitHub account
 - It has access to both repos
Permissions:
 - `contents: read/write` (for the K8s manifest repo).
If you create a `Fine-grained token`, just select both repos during token setup.

📌 **Example Setup Recap**
If your account is `awesome-dev`, and:
 - Docker image repo = `awesome-dev/my-app`
 - Manifest repo = `awesome-dev/k8s-manifests`
 - Manifest file = `apps/my-app/deployment.yaml`

You’d:
 - Create a token under `awesome-dev` with access to both repos.
 - Add that token as a secret named `TARGET_REPO_TOKEN` in `my-app`.