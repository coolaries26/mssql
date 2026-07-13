# **sql server with docker as container**
The Docker Desktop installer doesn’t let you pick a custom installation path. It always installs into the default location under **`C:\Program Files\Docker\Docker`**. That’s by design; Docker Desktop relies on tight integration with Windows services, WSL2, and networking components, so it needs to live in a predictable place.

Here are your practical options to install docker in wsl, if you want more control:

## 🔧 Using docker in wsl
- **Default Install + Data Relocation**  
  You can install Docker Desktop normally, but move the **Docker data (images, containers, volumes)** to another drive.  
  - In Docker Desktop → *Settings → Resources → Advanced* → change the disk image location.  
  - This lets you keep the heavy data (images, volumes) off your system drive while keeping the binaries in the default path.

- **Use Docker Engine directly in WSL2**  
  Instead of Docker Desktop, you can install the **Linux Docker Engine** inside your WSL distro (e.g., Ubuntu). That way, everything lives in your WSL filesystem, and you can control where it’s installed.  
  - Install with:
    ```bash
    sudo apt-get update
    sudo apt-get install -y docker.io
    ```
  - Enable and start the service:
    ```bash
    sudo service docker start
    ```
  - Add your user to the docker group:
    ```bash
    sudo usermod -aG docker $USER
    ```
  - Now you can run containers directly in WSL without Docker Desktop.
---
# **Portainer the Lightweight docker Managers**  
  If you don’t want Docker Desktop’s overhead, you can run Docker Engine in WSL and manage it with tools like **Portainer** (web UI for Docker).

### 1. Check Docker Service
Make sure Docker Engine is running inside WSL:
```bash
sudo service docker status
```
If not running:
```bash
sudo service docker start
```

### 2. Add Your User to the Docker Group if not added yet
```bash
sudo groupadd docker   # creates group if not already present
sudo usermod -aG docker $USER
```

Then log out and back in (or run `exec su -l $USER`) so the group membership takes effect.

### 3. Verify Permissions
Check the socket:
```bash
ls -l /var/run/docker.sock
```
You should see something like:
```
srw-rw---- 1 root docker ...
```
If the group is `docker` and your user is in that group, you’ll have access.

### 4. Portainer Setup
Now create the volume and run Portainer:
```bash
docker volume create portainer_data

docker run -d -p 9000:9000 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce
```

### 5. Access Portainer
Open a browser and go to:
```
http://localhost:9000
```
You’ll be prompted to set up an admin account and then you can manage containers via the web UI.
- on wsl run `docker ps` and `docker logs <image_name>` to get the setup-token for the admin user setup

⚡ Verify portrainer is accessable , you’ll be able to spin up your SQL Server lab container directly from Portainer’s dashboard.  

# Add **sql server container** . 
## 🔧 Step‑by‑step walkthrough:
- **01 Create a Docker Volume**
    sqlserver needs persistent storage for its database.

    Run in `WSL terminal`

    **Execute:** 
    ```
    docker volume create sqlvolume
    ```

    This ensures Sqlserver settings survive container restarts

- **02 Run sqlserver Container**
    pull sqlserver image using the official image.

    Run in `WSL terminal`
    ```
    docker pull mcr.microsoft.com/mssql/server:2022-latest
    ```
    **Execute:**

    ```bash
    docker run -e "ACCEPT_EULA=Y" \
           -e "SA_PASSWORD=YourStrongPassword" \
           -p 1433:1433 \
           --name sqlserverlab \
           -v "/mnt/D/Downloads:/var/opt/mssql/backup" \
           -d mcr.microsoft.com/mssql/server:2022-latest

    ```
- **03 Access sqlserver**
    on wsl run `ip route` to get the IP to use as connection string from windows ssms

    or on wsl after installing the clint  run 
    `sqlcmd -S localhost\sqllab -U sa -P yourstronpassword -d <db_name>`     


---
# Install **SQL Server client**
---

On newer Ubuntu/WSL releases, `apt-key` has been deprecated. Need to use the **signed‑by** method with GPG keys instead. Here’s the way to install `mssql-tools` so you can get `sqlcmd` working:

### 1. Install prerequisites
```bash
sudo apt-get update
sudo apt-get install -y curl gnupg2 software-properties-common
```

### 2. Add Microsoft’s GPG key and repo
Instead of `apt-key`, do this:

```bash
curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg
sudo install -o root -g root -m 644 microsoft.gpg /etc/apt/trusted.gpg.d/
sudo rm microsoft.gpg
```

Now add the repo (replace `22.04` with your Ubuntu version if different):
```bash
sudo curl https://packages.microsoft.com/config/ubuntu/22.04/prod.list \
     -o /etc/apt/sources.list.d/mssql-release.list
```

### 3. Install SQL tools
```bash
sudo apt-get update
sudo ACCEPT_EULA=Y apt-get install -y mssql-tools unixodbc-dev
```

### 4. Add to PATH
```bash
echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> ~/.bashrc
source ~/.bashrc
```

### 5. Test connection
Now you should have `sqlcmd` available:
```bash
sqlcmd -S localhost -U SA -P 'YourStrong!Passw0rd'
```

Run a quick query:
```sql
SELECT @@VERSION;
GO
```

---

# Restore adventure db and adventureDW db

on wsl run below to copy the backup to docker 

```
sudo docker cp "/mnt/d/Downloads/AdventureWorks2022.bak" sqlserverlab:/var/opt/mssql/backup/AdventureWorks2022.bak
Successfully copied 210MB to sqlserverlab:/var/opt/mssql/backup/AdventureWorks2022.bak

$ sudo docker cp "/mnt/d/Downloads/AdventureWorksDW2022.bak" sqlserverlab:/var/opt/mssql/backup/Adve
ntureWorksDW2022.bak
Successfully copied 102MB to sqlserverlab:/var/opt/mssql/backup/AdventureWorksDW2022.bak
```
**on SSMS**
 in ssms follow db restore 
