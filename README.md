# Dockerfile-publish

## 關於

這是一個使用 GitHub Action 建置 Dockerfile Image 的範例，並將 Image 上傳至 Docker Hub 的流程：



1. 在 GitHub 的設定頁面中，新增名稱為 Dockerhub 的 secret，並輸入你的 Docker Hub 帳號密碼，以便於 Action 中使用。

   

2. 在專案的根目錄中建立 Dockerfile 檔案，並按照需求進行編寫。

   

3. 在專案中建立 Github Action Workflow 檔案，例如 `.github/workflows/publish.yml`。在該檔案中，使用 `docker/login-action` 來登入 Docker Hub 帳號，並使用 `docker/build-push-action` 來建置並上傳 Docker Image。在 `on` 段落中，可以設置觸發事件，在本範例中當你 Release 後，會觸發 Action，並上傳 Image 至 Docker Hub。

   

4. 推送你的 Code 到 Github 並新增一個 Git Tag，觸發 Action 自動建置並上傳 Docker Image 至 Docker Hub。



## Dockerfile

```dockerfile
FROM ubuntu:latest
RUN apt-get update && apt-get install -y ssh
RUN mkdir /var/run/sshd
RUN echo 'root:password' | chpasswd
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```

這是一個 Dockerfile，用於建立一個基於最新版 Ubuntu 的 Docker 映像檔。該映像檔包含了一個安裝了 SSH 服務的 Ubuntu 系統，並開啟了 root 帳號的 SSH 登入權限。

以下是每一個指令的解釋：

- `FROM ubuntu:latest`：設定映像檔的基底為 Ubuntu 的最新版本。
- `RUN apt-get update && apt-get install -y ssh`：在映像檔內執行更新 Ubuntu 軟體庫的指令，並安裝 SSH 服務。
- `RUN mkdir /var/run/sshd`：建立 SSHD 的 PID 檔案目錄。
- `RUN echo 'root:password' | chpasswd`：設定 root 帳號的密碼為 `password`。
- `RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config`：修改 SSHD 的設定檔，允許 root 帳號以密碼登入。
- `EXPOSE 22`：設定該映像檔開啟的 port 為 22，也就是 SSH 服務的 port。
- `CMD ["/usr/sbin/sshd", "-D"]`：在啟動映像檔時，預設執行 `/usr/sbin/sshd -D` 這個指令，啟動 SSHD 服務，並保持在前景執行。

## Action

> 參考自GitHub官方文檔:https://docs.github.com/en/actions/publishing-packages/publishing-docker-images

```yaml
# This workflow uses actions that are not certified by GitHub.
# They are provided by a third-party and are governed by
# separate terms of service, privacy policy, and support
# documentation.

# GitHub recommends pinning actions to a commit SHA.
# To get a newer version, you will need to update the SHA.
# You can also reference a tag or branch, but the action may change without warning.

name: Publish Docker image

on:
  release:
    types: [published]

jobs:
  push_to_registry:
    name: Push Docker image to Docker Hub
    runs-on: ubuntu-latest
    steps:
      - name: Check out the repo
        uses: actions/checkout@v3
      
      - name: Log in to Docker Hub
        uses: docker/login-action@65b78e6e13532edd9afa3aa52ac7964289d1a9c1
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      
      - name: Extract metadata (tags, labels) for Docker
        id: meta
        uses: docker/metadata-action@9ec57ed1fcdbf14dcef7dfbe97b2010124a938b7
        with:
          images: my-docker-hub-namespace/my-docker-hub-repository
      
      - name: Build and push Docker image
        uses: docker/build-push-action@f2a1d5e99d037542a71f64918e516c093c6f3fc4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

示例工作流程中，我們使用了Docker login-action和build-push-action動作來構建Docker映像，如果構建成功，則將構建的映像推送到Docker Hub。

要推送到Docker Hub，您需要擁有Docker Hub帳戶，並創建一個Docker Hub存儲庫。有關詳細信息，請參閱Docker文檔中的“[Pushing a Docker container image to Docker Hub](https://docs.docker.com/docker-hub/repos/#pushing-a-docker-container-image-to-docker-hub)”。

`login-action`：

- username和password：這是您的Docker Hub用戶名和密碼。我們建議將Docker Hub用戶名和密碼存儲為密鑰，以便它們不會在工作流程文件中暴露。

`metadata-action`：

- images：您正在構建/推送到Docker Hub的Docker映像的命名空間和名稱，更多參數請查看[@docker/metadata-action](https://github.com/docker/metadata-action#tags-input)。

`build-push-action`：

- tags：您的新映像的標記，格式為DOCKER-HUB-NAMESPACE/DOCKER-HUB-REPOSITORY：VERSION。您可以像下面顯示的那樣設置單個標記，也可以在列表中指定多個標記。
- push：如果設置為true，則如果成功構建映像，將將映像推送到註冊表。
