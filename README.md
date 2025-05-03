# How to Create an Instance of n8n from a Snapshot, Update its Version, and Backup Using Snapshots

This guide provides step-by-step instructions to create a reliable instance of n8n, update its version, and secure its data using snapshots. Follow each step carefully to ensure a smooth process.

## Documentation
For further reference, check the official documentation: https://docs.n8n.io/hosting/installation/docker/#updating

## 1. Add Your User to the Docker Group

To run Docker commands without `sudo`:

```bash
whoami
sudo usermod -aG docker user
```

Log out and log back in for the changes to take effect:

```bash
exit
```

## 2. Verify the Current Version of n8n

Check the n8n version running inside the container:

```bash
docker exec -it n8n n8n --version
```

## 3. Inspect the Container to Find the Mount and Correct Directory

Run the following command to inspect the n8n container and locate the correct mount point:

```bash
docker inspect n8n
```

## 4. Update Domain Name (If Changed)

If the domain name for your n8n instance has changed, update your Nginx configuration:

```bash
sudo nano /etc/nginx/sites-available/n8n
```

Save changes with `Ctrl + O`, press `Enter`, and then exit with `Ctrl + X`.

## 5. Verify the Container ID

List all containers to get the correct Container ID:

```bash
docker ps -a
```

## 6. Backup Configuration Files from the Container

Access the container and list the configuration files:

```bash
docker exec -it --user node n8n sh
ls -la /home/node/.n8n
```

Exit the container:

```bash
exit
```

Backup the configuration files to your local machine:

```bash
docker cp <container_id>:/home/node/.n8n ./n8n_backup
```


Verify the backup file on your local machine:

```bash
ls -la ./n8n_backup
```

## 7. Update n8n to a Specific Version

Pull the desired version of the n8n Docker image:

```bash
docker pull docker.n8n.io/n8nio/n8n:1.73.1
```

## 8. Stop and Remove the Existing Container

Stop and remove the old container:

```bash
docker stop <container_id>
docker rm <container_id>
```

## 9. Create a Docker Volume for Persistent Data

Create a volume to store n8n data persistently:

```bash
docker volume create n8n_data
```

Copy the backup data into the new volume:

```bash
docker run --rm -v n8n_data:/data -v $(pwd)/n8n_backup:/backup busybox sh -c "cp -r /backup/* /data/"
```

Inspect the volume to confirm the data:

```bash
docker volume inspect n8n_data
```

## 10. Adjust Permissions for the Volume

Ensure the `node` user inside the container has proper permissions:

```bash
sudo chown -R 1000:1000 /var/lib/docker/volumes/n8n_data/_data
sudo chmod -R 700 /var/lib/docker/volumes/n8n_data/_data
```


## 11. Create a New n8n Container with the Restored Data

Run a new n8n container with the restored data "n8n_data" and updated version:

```bash
docker run -d --restart unless-stopped \
--name n8n \
-p 5678:5678 \
-e N8N_HOST="your-domain.com" \
-e WEBHOOK_TUNNEL_URL="https://your-domain.com/" \
-e WEBHOOK_URL="https://your-domain.com/" \
-v n8n_data:/home/node/.n8n \ 
n8nio/n8n:1.73.1
```

## 12. Verify the Backup Data in the Container

Start the container and check that the data is properly restored:

```bash
docker exec -it --user node n8n sh -c "ls -la /home/node/.n8n"
```

## Important Additional Information (Must Read)

### How Persistent Volumes Work
The `n8n_data` volume is stored outside of the container, ensuring that all your workflows and configurations are preserved even if the container is stopped or removed. Here's how it works:

1. **Data remains safe when stopping or restarting the container**:
   - Use `docker stop n8n` to stop the application.
   - Use `docker start n8n` to restart it, and your data will still be intact.

2. **Volume persists beyond container deletion**:
   - If you delete the container with `docker rm n8n`, the data in the `n8n_data` volume will remain safe.
   - You can reuse the volume by mapping it to a new container:

```bash
docker run -d --restart unless-stopped \
--name n8n \
-p 5678:5678 \
-e N8N_HOST="your-domain.com" \
-e WEBHOOK_TUNNEL_URL="https://your-domain.com/" \
-e WEBHOOK_URL="https://your-domain.com/" \
-v n8n_data:/home/node/.n8n \  # here is where we mount n8n_data
n8nio/n8n:latest # we call the latest stable version
```

By leveraging Docker volumes, you ensure that your n8n instance remains resilient and your data secure through updates or container changes.


## 🛠️ Important: Version 1.91.1+ Auth and Proxy Changes

Starting from version 1.91.1 (possibly as early as 1.81.2), new security and proxy features were introduced. These changes may block access if your environment isn't correctly configured.

## 🔒 Why Environment Variables Matter in n8n v1.91.1
Starting from version 1.91.1, n8n has implemented stricter security measures and proxy handling. This includes:

Enhanced Security Features: n8n now enforces stricter authentication protocols, which may require explicit configuration of authentication-related environment variables.

Proxy Configuration: Changes in how n8n handles proxies necessitate the explicit setting of certain environment variables to ensure proper routing and accessibility.

These updates mean that relying on default settings or previous configurations may lead to issues such as inaccessible instances or authentication errors.

🛠️ Transitioning to an .env File
To accommodate these changes and maintain a stable n8n environment, it's recommended to manage your configuration through an .env file. This approach offers:
n8n Documentation

Centralized Configuration: All environment variables are stored in a single, manageable file.

Improved Security: Sensitive information, such as authentication credentials, can be handled more securely.

Ease of Updates: Modifying configurations becomes straightforward, reducing the risk of errors during updates or migrations.

## If you're upgrading and encounter login issues (e.g., invalid password), follow these steps:

1. Inspect Current Env Variables


```bash
docker inspect n8n > n8n-config.json
docker inspect n8n | jq '.[0].Config.Env'
```

2. Debug Container Logs

```bash
docker logs -f n8n
```

3. Create an Environment File

```bash
nano n8n.env
```

Paste the following:

```bash
N8N_HOST=your-domain.com
N8N_PROTOCOL=https
N8N_PORT=5678
WEBHOOK_URL=https://your-domain.com/
WEBHOOK_TUNNEL_URL=https://your-domain.com/

N8N_TRUST_PROXY=true
N8N_LOG_LEVEL=info
N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
N8N_RUNNERS_ENABLED=true

N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=your-user-or-email
N8N_BASIC_AUTH_PASSWORD=your-password
```

⚠️ Make sure there are no leading/trailing spaces. Each line must start at the beginning.

4. Re-run n8n with the .env file

```bash
docker stop n8n && docker rm n8n

docker run -d \
--restart unless-stopped \
--name n8n \
-p 5678:5678 \
--env-file n8n.env \
-v n8n_data:/home/node/.n8n \
n8nio/n8n:1.91.1
```



