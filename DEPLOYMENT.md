# Azure Deployment Steps

## 1. Create Azure resources (one-time)

```bash
az group create --name fastapi-rg --location eastus

az appservice plan create \
  --name fastapi-plan \
  --resource-group fastapi-rg \
  --sku B1 --is-linux

az webapp create \
  --resource-group fastapi-rg \
  --plan fastapi-plan \
  --name fastapi-devops-app \
  --runtime "PYTHON:3.11"
```

The web app name (`fastapi-devops-app`) must match `AZURE_WEBAPP_NAME` in `.github/workflows/azure-deploy.yml`.

## 2. Configure startup command

In Azure Portal: **Web App → Configuration → General settings → Startup Command**, paste:

```
gunicorn -w 4 -k uvicorn.workers.UvicornWorker app:app --bind=0.0.0.0:8000
```

Or via CLI:

```bash
az webapp config set \
  --resource-group fastapi-rg \
  --name fastapi-devops-app \
  --startup-file "gunicorn -w 4 -k uvicorn.workers.UvicornWorker app:app --bind=0.0.0.0:8000"
```

## 3. Wire up GitHub Actions

1. In Azure Portal: **Web App → Get publish profile** (downloads `.PublishSettings`).
2. In GitHub repo: **Settings → Secrets and variables → Actions → New repository secret**.
   - Name: `AZURE_WEBAPP_PUBLISH_PROFILE`
   - Value: paste the full XML contents of the publish profile.
3. Push to `main` — the workflow deploys automatically.

## 4. Verify

```
https://fastapi-devops-app.azurewebsites.net/
```

Expected response:

```json
{"message": "FastAPI deployed on Azure using GitHub Actions"}
```

## Requirements validation

Pinned versions in `requirements.txt`:

- `fastapi==0.115.0` — web framework
- `uvicorn[standard]==0.30.6` — ASGI server (used by the worker class)
- `gunicorn==23.0.0` — process manager invoked by the startup command

All three are required: gunicorn manages workers, uvicorn provides the ASGI worker class, fastapi is the app.
