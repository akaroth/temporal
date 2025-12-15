# Cloudflare Tunnel Configuration

This directory contains the Cloudflare Tunnel (cloudflared) configuration for the temporal cluster.

## Architecture

```
Internet → Cloudflare Edge → Cloudflare Tunnel → Traefik → Applications
```

The Cloudflare Tunnel receives traffic from Cloudflare's edge network and forwards it to Traefik, which then routes to the appropriate services.

## Setup Instructions

### 1. Create a Cloudflare Tunnel

1. Go to [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/)
2. Navigate to **Networks** → **Tunnels**
3. Click **Create a tunnel**
4. Choose **Cloudflared** as the connector
5. Name it `temporal-tunnel`
6. Copy the tunnel credentials (AccountTag, TunnelSecret, TunnelID)

### 2. Configure Tunnel Credentials

Update the `secret-template.yaml` file with your actual credentials:

```yaml
stringData:
  credentials.json: |
    {
      "AccountTag": "YOUR_ACCOUNT_TAG",
      "TunnelSecret": "YOUR_TUNNEL_SECRET",
      "TunnelID": "YOUR_TUNNEL_ID",
      "TunnelName": "temporal-tunnel"
    }
```

Then create the secret:

```bash
kubectl apply -f secret-template.yaml --context kind-temporal
```

Or manually create it:

```bash
kubectl create secret generic cloudflared-credentials \
  --from-file=credentials.json=<(echo '{"AccountTag":"...","TunnelSecret":"...","TunnelID":"...","TunnelName":"temporal-tunnel"}') \
  -n cloudflare \
  --context kind-temporal
```

### 3. Configure DNS Routes

In the Cloudflare dashboard:
1. Go to your tunnel configuration
2. Add public hostnames:
   - `*.tikalabs.dev` → Points to your tunnel
   - `tikalabs.dev` → Points to your tunnel

### 4. Deploy via ArgoCD

The ArgoCD application will automatically deploy the tunnel once the secret is created.

## Configuration

The tunnel is configured to:
- Forward all `*.tikalabs.dev` and `tikalabs.dev` traffic to Traefik
- Traefik service is at: `http://traefik.traefik.svc.cluster.local:80`
- Catch-all returns 404 for unmatched routes

## Troubleshooting

### Check tunnel status:
```bash
kubectl logs -n cloudflare deployment/cloudflared --context kind-temporal
```

### Verify tunnel is running:
```bash
kubectl get pods -n cloudflare --context kind-temporal
```

### Test connectivity:
Once DNS is configured, test accessing your services through the tunnel.

## Notes

- The tunnel forwards to Traefik, which then handles internal routing
- Traefik manages all ingress routing internally
- Cloudflare Tunnel only provides the secure connection from Cloudflare's edge to the cluster

