---
title: "external secret 설치후기"
date: 2024-05-23T17:10:06+09:00
image: ''
draft: true
---



회사의 특정 GCP 프로젝트가 삭제되고 staging 환경에서 새로 argocd rollouts,linkerd, external secret operator등을 설치를 하려고한다.

개발환경에서 환경구축을 할 때 손수 매뉴얼하게 진행을 했었는데 이번에는 openTofu를 사용해서 진행을 해보았다


helm을 이용해서 external-secret operator를 설치

```
helm install external-secrets \
   external-secrets/external-secrets \
    -n external-secrets \
    --create-namespace
```

이때 특정 ns(external-secrets)에 pod가 여러개가 생성이되고  해당 [문서](https://external-secrets.io/latest/provider/google-secrets-manager/)에서 처럼 workload identity를 사용하여 gcp인증을 진행해야된다.

특정 서비스 어카운트를 생성하고 해당 어카운트에 workload identity user와 secret manager 엑세스 권한을 넣어주는것을 opentofu로 진행을 했다.

```
resource "google_service_account" "sa" {
  account_id   = "external-secrets"
  display_name = "external-secrets"
  project      = var.project_id
}

resource "google_service_account_iam_binding" "workload_identity_binding" {
  service_account_id = google_service_account.sa.name
  role               = "roles/iam.workloadIdentityUser"
  members            = [
    "serviceAccount:${var.project_id}.svc.id.goog[external-secrets/external-secrets]",
  ]
}

resource "google_secret_manager_secret_iam_member" "secret_manager_access"  {
  project = var.project_id
  secret_id = "projects//secrets/github-token"
  role = "roles/secretmanager.secretAccessor"
  member = "serviceAccount:${google_service_account.sa.email}"
}
```


해당 코드를 작성중 문뜩 개발환경을 말아먹은 때가 떠오르는데

당시 테라폼 코드를 작성중 서비스 어카운트 생성에  [google_project_iam_policy](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_project_iam#google_project_iam_policy)를 사용했었다  해당 문서에도 관련된 경고문구가 있었지만 당시에는 못보았다... 해당 리소스를 쓸경우 덮어씌워지는것도 아닌 내가 작성한 policy대로 초기화가된다. 즉 default service account는 물론 컴퓨트 엔진등 서비스 어카운트 모두 사라지게되니 조심하자.


해당 tf파일을 apply한후

kubernetes External secret과 secret store CRD를 모두 적용해주자

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: github-token
spec:
  refreshInterval: 1h             # rate SecretManager pulls GCPSM
  secretStoreRef:
    kind: ClusterSecretStore
    name: gcp-store               # name of the SecretStore (or kind specified)
  target:
    name: database-credentials    # name of the k8s Secret to be created
    creationPolicy: Owner
  data:
    - secretKey: github-token
      remoteRef:
        key: github-token      # name of the GCPSM secret key    # name of the GCPSM secret key
```

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: gcp-store
  namespace: external-secrets
spec:
  provider:
    gcpsm:
      projectID: "project" # config
```


다음 두개의 내용을 apply를 하게되면

secret이 생성이 되고. 해당 secret은 gcp secret manager에 등록된 시크릿이 보이게된다.

external-secret operator는 helm secret과 고민중에 선택한것으로 helm secret보다는 복잡성이 덜할것, 그리고 argocd를 통하지 않아도 되는 부분과 많은 operator지원으로 특정 CSP제약이 덜하다는게 똑같다

다만 helm secret보다는 복잡성때문에 진행해봤지만 이또한 tofu로 관리를 해야되는 문제 tofu또한 git으로 관리되는 문제로 gitops에서 secret 관리문제로 택했지만 이또한 git관리라 ...

helm secret도 테스트를 해봐야할것같다. 