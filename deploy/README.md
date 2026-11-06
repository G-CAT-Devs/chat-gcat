# Chat1 production deployment

This is a separate Open WebUI installation. It **does not** use the existing `chat-webui` container, `academic-chat-assistant_webui_data` volume, or port 8088. The new service uses port 8089 and the `chat-gcat_chat1_data` Docker volume; both requested hostnames point to it.

## Required external setup

1. Publish DNS records for **`chat1.gcat.ir`** and **`chat.1.g-cat.ir`** to the production ingress. As of preparation, neither name resolves. If proxied by Cloudflare/AbrArvan, configure the proxy to reach this server on HTTPS and ensure its origin certificate covers each name. A `*.g-cat.ir` wildcard **does not cover** `chat.1.g-cat.ir`.
2. Supply a valid TLS origin certificate for both names. The checked-in Nginx snippet references `/etc/nginx/ssl/selfsigned.crt` and `.key`, matching the existing proxy convention; those files are not trusted by browsers and the current certificate covers `*.g-cat.ir` only. Replace the certificate paths with suitable certificates if direct HTTPS or Full (strict) is required.
3. GitHub repository `G-CAT-Devs/chat-gcat`: set Actions secrets `CHAT1_VPS_HOST`, `CHAT1_VPS_USER`, `CHAT1_VPS_PORT` (22 here), `CHAT1_DEPLOY_SSH_KEY` (private half of a dedicated SSH key granted access to `/opt/chat-gcat` and Docker), and `CHAT1_SSH_KNOWN_HOSTS` (verified host key entry for the host and port). Restrict SSH access to the smallest necessary privileges; do not use `StrictHostKeyChecking=no`. Configure `production` environment approvals if desired. Ensure Actions have package write permission and the GHCR image `ghcr.io/g-cat-devs/chat-gcat` is public or the VPS has a read-only GHCR credential.
4. Create `/opt/chat-gcat/.env.production` with mode `600` from `.env.production.example` and replace `WEBUI_SECRET_KEY` with `openssl rand -hex 32`. The prepared configuration disables signup; to create the first administrator, enable signup **temporarily on a non-public ingress**, register the first account, then disable signup again. Do not expose an uninitialized instance with open signup. Configure model API credentials in the local env file or the admin UI.

## Deployment behavior

A PR validates Compose, shell syntax and Nginx syntax. A push to `main` builds an amd64 image on GitHub Actions and publishes to GHCR, then ships only the deployment files to `/opt/chat-gcat`, pulls the **exact image digest**, starts the separate Compose service and checks `/health` and container health. A failure restores the previous image when available. The existing chat stack is not restarted. The original upstream Docker/release/PyPI workflows are renamed `.disabled` to avoid publishing upstream artifacts.

The Nginx configuration is `deploy/nginx/chat1.conf`. Install it into an included `/etc/nginx` config path, check `nginx -t` and reload Nginx; this is a one-time server operation, not performed by the image workflow. Do not enable the site until an image is serving port 8089 or it will return 502. After deployment test both names with `curl -f https://<name>/health`, and verify the current chat.gcat.ir / chat.g-cat.ir service still responds.

**Do not** run `docker compose down -v` or re-use the existing chat data volume. Do not git-reset the server checkout; the workflow copies the exact Compose file and deployment script from the built revision.
