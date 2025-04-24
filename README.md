# 🛠 GitHub Actions Workflow (Cross-Repo Manifest Update)
The purpose of this project is to build and push Docker image to Dockerhub using github actions and then, update Kubernetes manifest files which are in a separate repository within the same GitHub account.

✅ **Prerequisites**:
1. Personal Access Token (PAT) with repo scope stored in your main repo as a secret:
 - Secret name: `TARGET_REPO_TOKEN`

2. Variables to set: (Optional)
 - Your target repo name `(e.g., your-username/k8s-manifests)`
 - Path to the manifest file `(e.g., deployments/my-app.yaml)`


```bash
name: Docker Image Build and Push To Docker Hub

on:
  push:
    branches:
      - main
    paths:
      - '**/*' # This will trigger whenever there are changes in any file in the repository
jobs:
  build:
    runs-on: ubuntu-latest

    env:
      TARGET_REPO: your_github_username/your_manifests_repo_name
      TARGET_REPO_TOKEN: ${{ secrets.TARGET_REPO_TOKEN }}
      MANIFEST_PATH: your_manifests_path/deploy.yml # (deploy.yml is the manifest_file_name)

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Docker
        uses: docker/setup-buildx-action@v2

      - name: Login To Dockerhub With Dockerhub Credentials
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Tag Docker Image With Current Date and Timestamp
        id: date_time
        run: echo "timestamp=$(date +'%Y-%m-%d-%H-%M')" >> $GITHUB_OUTPUT
      
      - name: Build Docker Image
        run: |
          docker build -t your_docker_username/your_image_name:${{ steps.date_time.outputs.timestamp }} . 
      
      - name: Push Docker Image To Dockerhub Repository
        run: |
          docker push your_docker_username/your_image_name:${{ steps.date_time.outputs.timestamp }}

#----------------------------------------------------
# UPDATE THE K8S MANIFEST FILES IN ANOTHER REPOSITORY
#-----------------------------------------------------

# Checkout target repository
      - name: Checkout target repository
        uses: actions/checkout@v2
        with:
          repository: ${{ env.TARGET_REPO }}
          token: ${{ secrets.TARGET_REPO_TOKEN }}

# Clone the target repository (K8s manifests)
      - name: Clone Manifest Repository (K8s manifests)
        run: |
          git clone https://x-access-token:${{ secrets.TARGET_REPO_TOKEN }}@github.com/${{ env.TARGET_REPO }}
          cd your_k8s_manifest_repo

# View the old k8s manifest (This step is optional)
      - name: Show Original Kubernetes Manifest
        run: |
          cat your_manifests_path/deploy.yml

# Update the image tag in the Kubernetes manifest file (Required) 
      - name: Update image tag in manifest
        run: |
          find ./manifests -type f -name "deploy.yml" -exec sed -i "s|${{ secrets.DOCKERHUB_USERNAME }}/your_image_name:.*|${{ secrets.DOCKERHUB_USERNAME }}/your_image_name:${{ steps.date_time.outputs.timestamp }}|g" {} +
        

 # View the new k8s manifest after update (This step is optional)
      - name: Show Updated Kubernetes Manifest
        run: |
         cat your_manifests_path/deploy.yml


# Commit and push updated manifests to target repository.
      - name: Commit and push updated manifest
          git config user.name "github-actions"
          git config user.email "github-actions@github.com"
          git add ${{ env.MANIFEST_PATH }}
          git commit -m "Update image tag to ${{ steps.date_time.outputs.timestamp }}" || echo "No changes to commit"
          git push
# OKAY, THE WORKFLOW IS COMPLETE.
```


# Step-by-Step: Create TARGET_REPO_TOKEN

1. Create a Personal Access Token
 - Go to GitHub `Developer Settings` → `Personal Access Tokens`.
 - Click `"Fine-grained tokens"` or `"Tokens (classic)"` → `Generate new token`.
 - Give it a name like `GitHub Actions Cross-Repo Push`.
 - Set an expiration `(recommended: 90 days or more)`.
 - Under `Repository Access`, choose:
    - Only select `repositories` → choose your `manifest repo` `(e.g., your-username/k8s-manifests)`
 - Under Permissions, enable: 
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
 - Clone the manifest repo.
 - Modify the Kubernetes YAML file.
 - Push changes back using the token's credentials.

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