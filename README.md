### För att installera GitHub CLI kan du följa dessa steg beroende på ditt operativsystem:

### På macOS

1. **Använd Homebrew:**

    ```bash
    brew install gh
    ```

2. **Kontrollera installationen:**

    ```bash
    gh --version
    ```

### På Ubuntu

1. **Lägg till GitHub CLI:s officiella APT-repo:**

    ```bash
    sudo apt update
    sudo apt install gh
    ```

2. **Kontrollera installationen:**

    ```bash
    gh --version
    ```

### På Windows

1. **Använd Windows Package Manager (winget):**

    ```bash
    winget install --id GitHub.cli
    ```

2. **Alternativt kan du ladda ner installationsprogrammet från [GitHub CLI:s releases-sida](https://github.com/cli/cli/releases) och köra det.**

3. **Kontrollera installationen:**

    ```bash
    gh --version
    ```

### På andra Linux-distributioner

Du kan använda [de officiella installationsanvisningarna från GitHub CLI:s dokumentation](https://cli.github.com/manual/installation) för att installera på andra Linux-distributioner eller via andra metoder.

Efter installationen kan du logga in med `gh auth login` och börja använda GitHub CLI.

---


### 1. Skapa ett GitHub-repository med GitHub CLI
Du kan använda GitHub CLI för att skapa ett nytt repository och klona det lokalt:

```bash
# Logga in på GitHub CLI
gh auth login

# Skapa ett nytt repository
gh repo create USERNAME/REPO --public --clone

# Gå in i repo-katalogen
cd REPO
```

Detta skapar ett GitHub-repository utan att behöva gå till det grafiska gränssnittet.

### 2. Generera en GitHub Actions Runner-token via GitHub API
För att automatiskt generera en registreringstoken för din självvärdade runner kan du använda GitHub API. Här är ett Bash-skript som gör detta:

```bash
# Variabler
OWNER="USERNAME"
REPO="REPO"
PAT="YOUR_GITHUB_PERSONAL_ACCESS_TOKEN"  # Generera en personlig åtkomsttoken med rättigheter för repo och workflows

# Skapa en registrerings-token för GitHub Actions-runner
RUNNER_TOKEN=$(curl -XPOST -H "Authorization: token $PAT" https://api.github.com/repos/$OWNER/$REPO/actions/runners/registration-token | jq -r .token)

# Kontrollera om token genererades korrekt
echo "Runner Token: $RUNNER_TOKEN"
```

### 3. Lägg till en Workflow-fil automatiskt
Skapa workflow-filen i din lokala repository och pusha den till GitHub med Git:

```bash
# Skapa en workflow-fil
mkdir -p .github/workflows
cat <<EOF > .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  build:
    runs-on: self-hosted

    steps:
    - name: Checkout repository
      uses: actions/checkout@v2

    - name: Build .NET project
      run: dotnet build
EOF

# Lägg till, commita och pusha workflow-filen
git add .github/workflows/ci.yml
git commit -m "Add CI workflow"
git push origin main
```

### 4. Provisionera VM och installera GitHub Actions-runner
Använd det tidigare nämnda Bash-skriptet för att provisionera din VM och installera GitHub Actions-runner med den genererade token. Uppdatera `cloud-init`-scriptet för att använda din token och repository URL:

```bash
CLOUD_INIT_SCRIPT=$(cat <<EOF
#cloud-config
runcmd:
  - sudo apt-get update
  - sudo apt-get install -y curl jq
  - mkdir /home/$ADMIN_USERNAME/actions-runner && cd /home/$ADMIN_USERNAME/actions-runner
  - curl -o actions-runner-linux-x64-2.308.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.308.0/actions-runner-linux-x64-2.308.0.tar.gz
  - tar xzf ./actions-runner-linux-x64-2.308.0.tar.gz
  - ./config.sh --url https://github.com/$OWNER/$REPO --token $RUNNER_TOKEN --unattended --replace
  - sudo ./svc.sh install
  - sudo ./svc.sh start
EOF
)
```

### 5. Automatisera allt med ett skript
Sammanfoga alla steg i ett enda Bash-skript som skapar repository, genererar en registrerings-token, lägger till en workflow-fil, provisionerar VM, och konfigurerar GitHub Actions-runner:

```bash
#!/bin/bash

# Variabler
RESOURCE_GROUP="myResourceGroup"
LOCATION="westeurope"
VM_NAME="myVM"
VM_SIZE="Standard_B1ms"
ADMIN_USERNAME="azureuser"
SSH_PUBLIC_KEY_PATH="~/.ssh/id_rsa.pub"
OWNER="USERNAME"
REPO="REPO"
PAT="YOUR_GITHUB_PERSONAL_ACCESS_TOKEN"

# Logga in på GitHub
gh auth login

# Skapa GitHub repo
gh repo create $OWNER/$REPO --public --clone
cd $REPO

# Skapa en workflow-fil
mkdir -p .github/workflows
cat <<EOF > .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  build:
    runs-on: self-hosted

    steps:
    - name: Checkout repository
      uses: actions/checkout@v2

    - name: Build .NET project
      run: dotnet build
EOF

# Lägg till, commita och pusha workflow-filen
git add .github/workflows/ci.yml
git commit -m "Add CI workflow"
git push origin main

# Generera en registreringstoken för GitHub Actions-runner
RUNNER_TOKEN=$(curl -XPOST -H "Authorization: token $PAT" https://api.github.com/repos/$OWNER/$REPO/actions/runners/registration-token | jq -r .token)
echo "Runner Token: $RUNNER_TOKEN"

# Provisionera VM och installera GitHub Actions-runner
az group create --name $RESOURCE_GROUP --location $LOCATION
CLOUD_INIT_SCRIPT=$(cat <<EOF
#cloud-config
runcmd:
  - sudo apt-get update
  - sudo apt-get install -y curl jq
  - mkdir /home/$ADMIN_USERNAME/actions-runner && cd /home/$ADMIN_USERNAME/actions-runner
  - curl -o actions-runner-linux-x64-2.308.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.308.0/actions-runner-linux-x64-2.308.0.tar.gz
  - tar xzf ./actions-runner-linux-x64-2.308.0.tar.gz
  - ./config.sh --url https://github.com/$OWNER/$REPO --token $RUNNER_TOKEN --unattended --replace
  - sudo ./svc.sh install
  - sudo ./svc.sh start
EOF
)

az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --image UbuntuLTS \
  --size $VM_SIZE \
  --admin-username $ADMIN_USERNAME \
  --ssh-key-value "$(<$SSH_PUBLIC_KEY_PATH)" \
  --custom-data <(echo "$CLOUD_INIT_SCRIPT")

# Öppna portar
az vm open-port --port 22 --resource-group $RESOURCE_GROUP --name $VM_NAME
az vm open-port --port 80 --resource-group $RESOURCE_GROUP --name $VM_NAME
```

### Sammanfattning:
Detta skript hanterar:
1. Skapande av GitHub-repository.
2. Generering av GitHub Actions-runner-token via GitHub API.
3. Skapande och uppladdning av en workflow-fil.
4. Provisionering av VM på Azure och installation av GitHub Actions-runner.