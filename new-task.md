# TASK 3: GitHub CI/CD su Linux istorijos pagrindu

## 1. Docker įdiegimas ir Rootless vartotojo paruošimas

### Įprastiniai diegimo veiksmai:

```bash
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
apt-cache policy docker-ce
sudo apt install docker-ce
sudo systemctl status docker
```

### Rootless naudotojo sukūrimas:

```bash
sudo usermod -aG docker ${USER}
groups
```

## 2. GitHub Actions Runner diegimas pagal istoriją

### Sukurkite katalogą ir atsisižskite runner:

```bash
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.324.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.324.0/actions-runner-linux-x64-2.324.0.tar.gz
echo "e8e24a3477da17040b4d6fa6d34c6ecb9a2879e800aa532518ec21e49e21d7b4  actions-runner-linux-x64-2.324.0.tar.gz" | shasum -a 256 -c
tar xzf ./actions-runner-linux-x64-2.324.0.tar.gz
```

### Runner konfiguravimas ir paleidimas:

```bash
./config.sh --url https://github.com/RaimundasR/test-workflow-1 --token AIOPGDSBVP5DUY4OQJHJVYTIFSEUS
./run.sh
```

### Runner paslaugos diegimas:

```bash
sudo ./svc.sh install
sudo ./svc.sh start
systemctl status actions.runner.RaimundasR-test-workflow-1.test.service
```

## 3. Repozitorijos ir Docker failų paruošimas

### Kodo įrankiai:

```bash
git clone https://github.com/RaimundasR/test-workflow-1.git
cd test-workflow-1
git checkout -b features/my-first-setup
```

### Sukurkite `Dockerfile`:

```Dockerfile
FROM nginx:latest
WORKDIR /usr/share/nginx/html
RUN rm -rf ./*
COPY index.html .
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Sukurkite `index.html` su individualiu HTML:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Advanced Nginx Docker Image</title>
    <style>
        body { font-family: Arial; text-align: center; margin-top: 50px; }
        h1 { color: rgb(106, 108, 200); }
        p { color: #555; }
    </style>
</head>
<body>
    <h1>Welcome to the Modified Nginx Docker Image!</h1>
    <p>This page is served by a custom Nginx container.</p>
</body>
</html>
```

### Git veiksmai:

```bash
git add --all
git commit -m "feat: first initial commit"
git push
```

## 4. CI/CD Workflows konfigūravimas

### Sukurkite katalogą:

```bash
mkdir -p .github/workflows
```

### Sukurkite `nginx-ci-cd.yml`:

```yaml
name: Nginx Docker Build and Deploy

on:
  push:
    branches:
      - "features/**"
      - "feature/**"
      - main
  pull_request:
    branches:
      - main
    types: [opened, reopened, ready_for_review]

jobs:
  build-development:
    runs-on: bka-label
    timeout-minutes: 5
    environment: dockerhub-registry
    if: github.event_name == 'push' && github.ref != 'refs/heads/main'
    env:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
    outputs:
      DOCKER_TAG: ${{ steps.docker_tag_step.outputs.DOCKER_TAG }}
    steps:

      - name: Checkout the Latest Code
        uses: actions/checkout@v4
        with:
          ref: ${{ github.ref }}
          fetch-depth: 0

      - name: Define Docker Image Name
        run: |
          IMAGE_NAME="$DOCKER_USERNAME/my-nginx-app"
          echo "IMAGE_NAME=$IMAGE_NAME" >> $GITHUB_ENV
          echo "✅ IMAGE_NAME set to: $IMAGE_NAME"

      - name: Ensure Directory Exists for Docker Version
        run: |
          VERSION_FILE="${RUNNER_TEMP}/docker_version.txt"
          mkdir -p "$(dirname "$VERSION_FILE")"
          touch "$VERSION_FILE"
          echo "VERSION_FILE=$VERSION_FILE" >> $GITHUB_ENV

      - name: Check and Set Docker Tag
        id: docker_tag_step
        run: |
          VERSION_FILE="$HOME/docker_version.txt"
          if [[ -f "$VERSION_FILE" ]]; then
            LAST_VERSION=$(cat "$VERSION_FILE" | sed 's/v//')
            NEXT_VERSION=$((LAST_VERSION + 1))
          else
            NEXT_VERSION=1
          fi
          echo "v$NEXT_VERSION" > "$VERSION_FILE"
          echo "DOCKER_TAG=v$NEXT_VERSION" >> $GITHUB_ENV
          echo "✅ DOCKER_TAG set to: v$NEXT_VERSION"

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
        with:
          driver: docker-container

      - name: Build and Push Docker Image to Docker Hub
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ env.IMAGE_NAME }}:${{ env.DOCKER_TAG }},${{ env.IMAGE_NAME }}:latest
          no-cache: true

  deploy-development:
    runs-on: bka-label
    timeout-minutes: 5
    environment: dockerhub-registry
    needs: build-development
    if: github.event_name == 'push' && github.ref != 'refs/heads/main'
    steps:
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Pull and Run Latest Development Image
        run: |
          IMAGE_NAME="${{ secrets.DOCKER_USERNAME }}/my-nginx-app"
          docker stop nginx-dev || true
          docker rm -f nginx-dev || true
          docker rmi -f $(docker images -q "$IMAGE_NAME") || true
          docker pull "$IMAGE_NAME:latest"
          docker run -d --name nginx-dev -p 8082:80 "$IMAGE_NAME:latest"

      - name: Cleanup Docker Cache
        run: docker system prune -af
        if: always()

  deploy-staging:
    runs-on: bka-label
    timeout-minutes: 5
    environment: dockerhub-registry
    if: github.event_name == 'pull_request'
    steps:
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      - name: Install jq
        run: sudo apt-get update && sudo apt-get install -y jq
      - name: Get Latest v<num> Docker Tag
        run: |
          IMAGE_NAME="${{ secrets.DOCKER_USERNAME }}/my-nginx-app"
          DOCKER_TAG=$(curl -s "https://hub.docker.com/v2/repositories/${IMAGE_NAME}/tags?page_size=20" | \
          jq -r '[.results[] | select(.name | test("^v[0-9]+$"))] | sort_by(.last_updated) | reverse | .[0].name')
          echo "DOCKER_TAG=$DOCKER_TAG" >> $GITHUB_ENV
          echo "✅ Using latest Docker tag: $DOCKER_TAG"

      - name: Pull and Run Latest Staging Image
        run: |
          IMAGE_NAME="${{ secrets.DOCKER_USERNAME }}/my-nginx-app"
          docker pull "$IMAGE_NAME:$DOCKER_TAG"
          docker stop nginx-staging || true
          docker rm -f nginx-staging || true
          docker rmi -f $(docker images -q "$IMAGE_NAME") || true
          docker run -d --name nginx-staging -p 8081:80 "$IMAGE_NAME:$DOCKER_TAG"

  deploy-production:
    runs-on: bka-label
    timeout-minutes: 5
    environment: dockerhub-registry
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      - name: Install jq
        run: sudo apt-get update && sudo apt-get install -y jq
      - name: Get Latest v<num> Docker Tag
        run: |
          IMAGE_NAME="${{ secrets.DOCKER_USERNAME }}/my-nginx-app"
          DOCKER_TAG=$(curl -s "https://hub.docker.com/v2/repositories/${IMAGE_NAME}/tags?page_size=20" | \
          jq -r '[.results[] | select(.name | test("^v[0-9]+$"))] | sort_by(.last_updated) | reverse | .[0].name')
          echo "DOCKER_TAG=$DOCKER_TAG" >> $GITHUB_ENV
          echo "✅ Using latest Docker tag: $DOCKER_TAG"

      - name: Pull and Run Latest Production Image
        run: |
          IMAGE_NAME="${{ secrets.DOCKER_USERNAME }}/my-nginx-app"
          docker pull "$IMAGE_NAME:$DOCKER_TAG"
          docker stop nginx-prod || true
          docker rm -f nginx-prod || true
          docker rmi -f $(docker images -q "$IMAGE_NAME") || true
          docker run -d --name nginx-prod -p 8080:80 "$IMAGE_NAME:$DOCKER_TAG"
```

### Git veiksmai:

```bash
git add .github/workflows/nginx-ci-cd.yml
git commit -m "ci: add nginx CI/CD workflow"
git push origin features/my-first-setup
```

---

> Remtasi komandomis iš Linux `history`, kurias naudojo vartotojas "tenant" prie projekto `test-workflow-1`, diegiant Docker, GitHub Actions runner ir sukuriant CI/CD procesą.
