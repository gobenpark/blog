---
title: "Gitops"
date: 2024-06-03T13:56:46+09:00
image: ''
draft: true
---


## gitops + github action + artifact registry 

> 해당 글은 gitops와 github action 을 통해 gcp 의 artifact registry에 Push를 위한 방법을 기록하기 위한 글입니다.


추후 기록을 위해서 tofu 코드를 통해 모든 것을 작성

1. google workload identity pool을 생성 
```terraform
resource "google_iam_workload_identity_pool" "wi_pool" {
  provider      = google
  disabled = false
  display_name = "GitHub Actions Pool"
  project = var.project_id
  workload_identity_pool_id = "github"
  lifecycle {
    ignore_changes = [
      project,  // project 변경을 무시하도록 설정
    ]
  }
}
```

2. oidc인증을 위한 github provider 생성을 진행한다
```terraform
resource "google_iam_workload_identity_pool_provider" "oidc_provider" {
  provider                    = google
  project = var.project_id
  workload_identity_pool_id   = "github"
  workload_identity_pool_provider_id = "repo"
  display_name                = "Github repo Provider"
  attribute_mapping = {
    "google.subject"        = "assertion.sub"
    "attribute.actor"       = "assertion.actor"
    "attribute.repository"  = "assertion.repository"
    "attribute.repository_owner" = "assertion.repository_owner"
  }
  attribute_condition         = "assertion.repository_owner == 'gobenpark'"
  oidc {
    issuer_uri               = "https://token.actions.githubusercontent.com"
  }
  lifecycle {
    ignore_changes = [
      project,  // project 변경을 무시하도록 설정
    ]
  }
}
```

3. 해당 pool을 이용하기 위한 service account 생성
```terraform
resource "google_service_account" "packager_gsa" {
  account_id   = "github"
  display_name = "github"
  project      = var.project_id
}
```

4. workload identity user권한을 바인딩한다.
```terraform
resource "google_service_account_iam_binding" "workload_identity_user" {
  service_account_id = "${google_service_account.packager_gsa.name}"
  role               = "roles/iam.workloadIdentityUser"
  members = [
    "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.wi_pool.id}/attribute.repository/gobenpark/charts",
    "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.wi_pool.id}/attribute.repository/gobenpark/ai"
  ]
}
```

5. artifact registry에 푸시가 가능하도록 권한을 주입한다.
```terraform

resource "google_project_iam_member" "artifact_registry_binding" {
  project = var.project_id
  role    = "roles/artifactregistry.writer"

  member = "serviceAccount:${google_service_account.packager_gsa.email}"
}
``` 

6. github action yaml설정
```yaml
name: Docker Image CI

permissions:
  contents: read
  id-token: write
on:
  push:
    branches: [ "main" ]
jobs:
  build:
    runs-on: self-hosted
    outputs:
      TAG: ${{ github.sha }}
      REPOSITORY: us-west1-docker.pkg.dev/${{ secrets.PROJECT_ID }}/${{ secrets.ARTIFACT_REGISTRY_REPOSITORY }}/${{ github.repository }}
    steps:
      - uses: actions/checkout@v4
        name: Checkout
      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          # list of Docker images to use as base name for tags
          images: |
            us-west1-docker.pkg.dev/${{ secrets.PROJECT_ID }}/${{ secrets.ARTIFACT_REGISTRY_REPOSITORY }}/${{ github.repository }}
          tags: |
            latest
            ${{ github.sha }}
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: '${{ secrets.WI_POOL_PROVIDER_ID }}'
          service_account: '${{ secrets.PACKAGER_GSA_ID }}'
          token_format: 'access_token'
      - uses: google-github-actions/setup-gcloud@v2
        with:
          version: latest
      - name: login to artifact registry
        run: |
          gcloud auth configure-docker ${{ secrets.ARTIFACT_REGISTRY_HOST_NAME }} --quiet
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./deployments/Dockerfile
          push: true
          labels: ${{ steps.meta.outputs.labels}}
          tags: ${{ steps.meta.outputs.tags }}
  manifest:
    runs-on: self-hosted
    needs: build
    steps:
      - uses: actions/checkout@v4
        with:
          repository: gobenpark/manifest
          token: ${{ secrets.PAT }}
          ref: main
      - name: Change manifest ja
        uses: mikefarah/yq@master
        with:
          cmd: yq e --inplace '.image.tag = "${{ needs.build.outputs.TAG }}" | .image.repository = "${{ needs.build.outputs.REPOSITORY }}"' workloads/env/express/external/ai_values.yaml
      - name: git push
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git config credential.helper store
          git add workloads/env/express/external/ai_values.yaml
          git commit -m "🎉 Update: Image [${{ needs.build.outputs.TAG }}]"
          git push

```

7. github repository secret 설정 
>  각 요소의 secret을 넣어준다.
ll
> 