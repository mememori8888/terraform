3. Terraform（マルチテナント・高速インフラ複製）
💡 役割とメリット
インフラ構成（Google Cloud Run、ネットワーク、IAMセキュリティ等）をすべてコード化（IaC）します。

5分で新規テナント環境を複製: エンタープライズ顧客やOEM販売パートナーから「セキュリティの関係上、他社と共有のサーバーではなく、自社専用の独立したクラウド環境（コンテナ）を用意してほしい」と言われた際、ブラウザをポチポチ操作することなく、コマンド一発で寸分違わぬ安全な環境を自動構築・デプロイできます。

💻 サンプルコード (HCL / Terraform main.tf)
Terraform
provider "google" {
  project = var.gcp_project_id
  region  = "asia-northeast1" # 東京リージョン（国内データ保存の担保）
}

# 共通レジ基盤（FastAPI）をステートレスコンテナとしてデプロイ
resource "google_cloud_run_v2_service" "common_register_backend" {
  name     = "ai-common-register-${var.tenant_id}" # テナントごとに名前を完全隔離
  location = "asia-northeast1"

  template {
    scaling {
      max_instance_count = 10  # 急激なスパイクにも自動で水平スケール
      min_instance_count = 0   # アクセスがない時は0台になりインフラコストを完全カット（コールドスタート対応）
    }

    containers {
      image = "asia-northeast1-docker.pkg.dev/${var.gcp_project_id}/register-repo/backend:latest"
      
      # 環境変数による厳格なテナント・コンテキスト制御（フェイルセーフ）
      env {
        name  = "TENANT_ID"
        value = var.tenant_id
      }
      env {
        name  = "LAGO_API_URL"
        value = "[https://api.getlago.com/api/v1](https://api.getlago.com/api/v1)"
      }
      env {
        name  = "ENCRYPTION_SECRET_KEY"
        value = var.encryption_secret_key # SQLiteの透過的暗号化に使用する鍵を注入
      }
    }
  }
}

# 外部からのアクセス制限と認証（API Key HeaderとIAMの2重ガード）
resource "google_cloud_run_v2_service_iam_member" "public_access" {
  name     = google_cloud_run_v2_service_common_register_backend.name
  location = google_cloud_run_v2_service_common_register_backend.location
  role     = "roles/run.invoker"
  member   = "allUsers" # エンドポイント自体は公開し、アプリケーション層のX-API-KEYで弾く
}
