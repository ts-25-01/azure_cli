# Einführung in die Azure CLI
Die Azure CLI ist ein Kommandozeilenwerkzeug, mit dem du Azure-Ressourcen erstellen, konfigurieren und verwalten kannst – ohne Portal.
Sie funktioniert unter Linux, macOS, Windows und sogar in der Cloud Shell.
```bash
# Windows (PowerShell)
choco install azure-cli

# macOS
brew install azure-cli

# Linux (Debian/Ubuntu)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```
Anmeldung mit
```bash
az login
```
### Grundbefehle der Azure CLI
| Befehl                    | Bedeutung                               |
| ------------------------- | --------------------------------------- |
| `az group`                | Ressourcengruppen verwalten             |
| `az vm`                   | Virtuelle Maschinen erstellen/verwalten |
| `az storage account`      | Storage Accounts verwalten              |
| `az storage blob`         | Blobs hochladen/verwalten               |
| `az deployment group/sub` | ARM/Bicep Deployments (IaC)             |

## Aufgabe 1: Eine Linux VM starten
Wir erstellen uns erstmal einen SSH-Key (https://learn.microsoft.com/en-us/azure/virtual-machines/linux/mac-create-ssh-keys)
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/azure_rsa
```

Finde zunächst den Namen deiner eigenen Ressourcengruppe heraus mit `az group list` und trage den Namen hinter --resource-group ein!
```bash
az vm create \
  --resource-group helen \
  --name myVm01 \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/azure_rsa.pub
```
**Achtung**: Ersetze die Ressourcengruppe durch den Namen deiner eigenen!

Wir können nachträglich noch den Port 22 freigeben:
```bash
az vm open-port \
  --port 22 \
  --resource-group helen \
  --name myVm01
```
Jetzt kannst du dich verbinden mit `ssh -i <Pfad zum Key> azureuser@public-ip-der-vm`
#### VM stoppen (nur stoppen, vllt weitere Kosten)
```bash
az vm stop \
  --resource-group helen \
  --name myVm01
```
#### VM vollständig freigeben (empfohlen, keine Kosten mehr)
```bash
az vm deallocate \
  --resource-group helen \
  --name myVm01
```
#### VM löschen
```bash
az vm delete \
  --resource-group helen \
  --name myVm01 
```

## Aufgabe 2: Eine VM erstellen und NGINX per Cloud-Init installieren
1. Erstelle eine neue Datei mit dem Namen `cloud-init-nginx.yaml` (so wie User Data Script in AWS)
```yaml
#cloud-config
package_upgrade: true
packages:
  - nginx
runcmd:
  - systemctl enable nginx
  - systemctl start nginx
write_files:
  - path: /var/www/html/index.html
    content: |
      <h1>Hallo von Azure! NGINX läuft</h1>
```
2. Starte dann die Maschine mit folgenden Befehl:
```bash
az vm create \
  --resource-group helen \
  --name nginx-vm \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --nic-delete-option delete \
  --os-disk-delete-option delete 
  --custom-data cloud-init-nginx.yaml
```
Notiere dir die public ip!
3. Öffne dann den Port 80 auf der Maschine
```bash
az vm open-port \
  --resource-group helen \
  --name nginx-vm \
  --port 80

```
(Iwas hab mit Cloud init nicht funktioniert TODO)
4. Deallocaten wieder:
```bash
az vm deallocate \
  --resource-group helen \
  --name nginx-vm
```
## Aufgabe 3: VM mit Docker installieren und einen Webserver drauf installieren, wir probieren Tetris
1. Cloud init Script
```bash
#cloud-config
package_update: true
packages:
  - docker.io
runcmd:
  # Docker aktivieren
  - systemctl enable docker
  - systemctl start docker

  # Beispiel: cooles Webserver-Projekt (Tetris)
  - docker run -d \
      --name tetris \
      -p 80:80 \
      uzyexe/tetris
```
2. Dann VM starten mit
```bash
az vm create \
  --resource-group helen \
  --name docker-tetris-vm \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --custom-data cloud-init-docker-tetris.yaml
```
3. Dann Port freigeben
```bash
az vm open-port \
  --resource-group helen \
  --name docker-tetris-vm \
  --port 80
```
4. VM deallocaten mit
```bash
az vm deallocate \
  --resource-group helen \
  --name docker-tetris-vm
```
## Aufgabe 4: Static Website Hosting auf Blob Storage
1. Storage Account erstellen
```bash
az storage account create \
  --name mystorage$RANDOM \
  --resource-group helen \
  --location westeurope \
  --sku Standard_LRS \
  --kind StorageV2
```
2. Webseite-Funktion aktivieren
```bash
az storage blob service-properties update \
  --account-name mystorage24060 \
  --static-website \
  --index-document index.html \
```
3. Datei hochladen
```bash
az storage blob upload \
  --account-name mystorage24060 \
  --container-name '$web' \
  --name index.html \
  --file index.html 
```
4. URL der Webseite abrufen
```bash
az storage account show \
  --name mystorage24060 \
  --query "primaryEndpoints.web" \
  --output tsv
```

## Aufgabe 5: Azure Functions
1. Azure Functions Core Tools installieren
```bash
# MacOs
brew tap azure/functions
brew install azure-functions-core-tools@4

# Windows
choco install azure-functions-core-tools

# Linux
curl -sL https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg
sudo mv microsoft.gpg /etc/apt/trusted.gpg.d/microsoft.gpg
sudo add-apt-repository "deb https://packages.microsoft.com/repos/azure-cli/ $(lsb_release -cs) main"
sudo apt-get update
sudo apt-get install azure-functions-core-tools-4

```
brauchen wir nicht mehr haha... eig haben wir das bereits über npm installieren, aber passt schon
2. Danach Funktion anlegen mit
```bash
func init funemoji --javascript
```
3. Funktion dann erstellen mit
```bash
cd funemoji
func new --name EmojiFunction --template "HTTP trigger"
```
Wähle Node und Javascript aus
4. Funktion lokal testen mit:
```bash
func start
```
5. Funktion in Storage Account anlegen
```bash
az functionapp create \
  --name testfunctionhelen \
  --resource-group helen \
  --consumption-plan-location westeurope \
  --runtime node \
  --runtime-version 22 \
  --functions-version 4 \
  --storage-account mystorage24060
```
Verwende einfach den Storage Account aus dem vorherigen Beispiel ☺️
5. Funktion deployen
```bash
func azure functionapp publish FUNAPPNAME
```
