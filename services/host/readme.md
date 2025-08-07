# Shared KVM Docker Services

This folder contains Docker Compose setup for:

- Samba file sharing
- Nginx static web server

## Environment Variables

Create a `.env` file in this folder with the following content:

```env
SAMBA_NAME=Data
SAMBA_USER=your-username
SAMBA_PASS=your-password
SAMBA_MOUNT=/your/local/folder
NGINX_PORT=7080
```

* `SAMBA_NAME`: Name of the shared folder as it appears to clients.
* `SAMBA_USER`: Username used to authenticate with the Samba server.
* `SAMBA_PASS`: Password for the above user.
* `SAMBA_MOUNT`: Path on the host to be shared via Samba.
* `NGINX_PORT`: External port used to access the Nginx server (mapped to container port 80).
