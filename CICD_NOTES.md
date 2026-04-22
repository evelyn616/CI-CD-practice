# CI/CD 學習筆記

## 完整流程概覽

```
git push
   ↓
GitHub Actions 觸發
   ↓
pytest 自動跑測試
   ↓
Build Docker image
   ↓
推送到 AWS ECR
   ↓
SSH 進 EC2 拉最新 image
   ↓
Container 跑起來，對外服務
```

---

## 一、專案結構

```
ci-cd-practice/
├── app.py                        # Flask API
├── test_app.py                   # pytest 測試
├── requirements.txt              # Python 依賴
├── Dockerfile                    # Docker image 定義
└── .github/
    └── workflows/
        └── ci.yml                # GitHub Actions workflow
```

---

## 二、Git Workflow

### 概念
- 不直接 push 到 main，改用 feature branch → PR → merge 的流程
- 保護 main branch，避免壞掉的 code 進入正式環境

### 常用指令

```bash
git init                                  # 初始化 repo
git add .                                 # 加入所有檔案到暫存區
git commit -m "feat: 說明"                # 儲存版本
git branch -M main                        # 將預設 branch 改名為 main
git remote add origin <GitHub repo URL>   # 設定遠端 repo
git push -u origin main                   # 第一次推上 GitHub

git checkout -b feature/xxx               # 建立並切換到新 branch
git push -u origin feature/xxx            # 推 feature branch 上去
git checkout main                         # 切回 main
git pull origin main                      # 拉最新的 main
```

### PR 流程
1. push feature branch 到 GitHub
2. GitHub 上點「Compare & pull request」
3. 填寫說明，按「Create pull request」
4. 確認後按「Merge pull request」

---

## 三、GitHub Actions CI

### 概念
- 放在 `.github/workflows/*.yml` 的檔案會被 GitHub 自動偵測
- push 或 PR 時自動觸發，在 GitHub 的虛擬機上執行

### ci.yml 完整內容

```yaml
name: CI

on:
  push:
    branches: ["**"]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest test_app.py -v

  build:
    runs-on: ubuntu-latest
    needs: test   # 測試通過才執行

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push Docker image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: harper-test
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

      - name: Open SSH port
        run: |
          MY_IP=$(curl -s https://checkip.amazonaws.com)/32
          aws ec2 authorize-security-group-ingress \
            --group-id ${{ secrets.EC2_SG_ID }} \
            --protocol tcp --port 22 --cidr $MY_IP

      - name: Deploy to EC2
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin 851662735658.dkr.ecr.ap-northeast-2.amazonaws.com
            docker stop ci-cd-practice || true
            docker rm ci-cd-practice || true
            docker pull 851662735658.dkr.ecr.ap-northeast-2.amazonaws.com/harper-test:${{ github.sha }}
            docker run -d --name ci-cd-practice -p 5000:5000 851662735658.dkr.ecr.ap-northeast-2.amazonaws.com/harper-test:${{ github.sha }}

      - name: Close SSH port
        if: always()   # 不管成功失敗都執行
        run: |
          MY_IP=$(curl -s https://checkip.amazonaws.com)/32
          aws ec2 revoke-security-group-ingress \
            --group-id ${{ secrets.EC2_SG_ID }} \
            --protocol tcp --port 22 --cidr $MY_IP
```

---

## 四、AWS 設定

### ECR（Elastic Container Registry）
- 用來存放 Docker image 的地方（AWS 版 Docker Hub）
- 建立：AWS Console → ECR → Create repository

### IAM User（給 GitHub Actions 用）
- 建立專用 user：`github-actions-ecr`
- 附加權限：`AmazonEC2ContainerRegistryPowerUser`、`AmazonEC2FullAccess`
- 產生 Access Key，存到 GitHub Secrets

### EC2
- AMI：Amazon Linux 2023
- Instance type：t2.micro
- 安裝 Docker：
```bash
sudo yum update -y
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
```
- 安裝 AWS CLI 並設定：
```bash
sudo yum install -y aws-cli
aws configure
```

### Security Group 設定
| Port | 用途 | Source |
|------|------|--------|
| 22 | SSH（由 workflow 動態開關） | GitHub Actions IP |
| 5000 | 應用程式對外服務 | 0.0.0.0/0 |

---

## 五、GitHub Secrets 清單

| Secret 名稱 | 說明 |
|-------------|------|
| `AWS_ACCESS_KEY_ID` | IAM user 的 Access Key ID |
| `AWS_SECRET_ACCESS_KEY` | IAM user 的 Secret Access Key |
| `AWS_REGION` | `ap-northeast-2` |
| `EC2_HOST` | EC2 的 Public IP |
| `EC2_USER` | `ec2-user` |
| `EC2_SSH_KEY` | `.pem` 檔案的完整內容 |
| `EC2_SG_ID` | Security Group ID（格式：sg-xxxxxxxxx） |

---

## 六、重要觀念

### CI vs CD
- **CI（持續整合）**：自動跑測試、build image，確保 code 品質
- **CD（持續部署）**：自動把通過測試的 code 部署到伺服器

### `needs` 關鍵字
```yaml
needs: test  # 這個 job 必須等 test job 成功才會執行
```

### `if: always()`
```yaml
if: always()  # 不管前面的步驟成功或失敗，這個步驟都會執行
```
常用於清理工作，例如確保 SSH port 一定會被關掉。

### Image Tag 用 commit hash
```yaml
IMAGE_TAG: ${{ github.sha }}
```
每個 commit 都有唯一的 hash，用它當 tag 可以追蹤每個 image 對應哪個版本的 code。
