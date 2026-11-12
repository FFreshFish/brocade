# External fleet subscription API

The external API exports the subscriptions that Brocade currently serves across all tenants and users. It is intended for a separately operated subscription-management panel.

## Enable the API

The API is disabled unless the Brocade server process has a valid `BROCADE_EXTERNAL_API_TOKEN` environment variable. The value must be 32–512 bytes and use HTTP Bearer-token-safe characters.

Generate a token on the server:

```sh
openssl rand -hex 32
```

Set the generated value in the environment used by the Brocade service. For systemd, use the deployment's existing environment file or a service drop-in; do not put the token in the repository, database, frontend build variables, or a URL. Restart the Brocade service after changing its process environment.

The token is never returned by the status endpoint or stored in the application database. Use HTTPS for every request.

## Authentication

Send the token in the HTTP Authorization header:

```http
Authorization: Bearer <token>
```

Requests without a configured token or with an invalid token receive `401 Unauthorized`. API calls are throttled to 60 requests per source IP per minute, including failed authentication attempts. Responses use no-store cache headers.

## Endpoints

All endpoints are GET requests.

| Endpoint | Description |
| --- | --- |
| `/api/external/v1/nodes` | Structured JSON for all active users and their subscription artifacts |
| `/api/external/v1/nodes?format=clash` | One merged Clash/Mihomo YAML configuration |
| `/api/external/v1/nodes?format=uri` | One newline-separated list of complete node URIs |
| `/api/external/v1/health` | Authenticated health check |

The system-admin-only console status endpoint is `/external-api/status`. It reports whether the environment variable is valid, the public Agent origin (`base_url`), the API path (`base_path`), and supported formats; it never returns the token. Use the returned Agent origin, not necessarily the Console page's origin, because deployments may split the Agent and Console listeners. If it is the default loopback address (127.0.0.1), configure a publicly reachable Agent URL before giving the endpoint to an external service.

## JSON response

The JSON response has this envelope:

```json
{
  "format": "json",
  "revision": 123,
  "users": [
    {
      "tenant_id": "tenant-id",
      "user_id": "user-id",
      "revision": 123,
      "nodes": [],
      "external_proxies": [],
      "front_groups": []
    }
  ]
}
```

The `nodes` entries are serialized Brocade subscription artifacts. They include connection addresses, ports, credentials, security settings (TLS/Reality/AnyTLS/Hysteria2/MTProto), and TCP/XHTTP transport settings. `external_proxies` and `front_groups` retain the routing dependencies used by fronted nodes. These fields contain live credentials: treat the response as a secret and do not expose it to browsers or logs.

The user list is built from the latest successful published serving projection, not directly from draft database rows. Each user's service start/expiry window is checked at request time. Grant and ingress eligibility comes from the published projection. A projection failure aborts the whole response rather than silently omitting that user. The top-level revision is preserved even when no users currently have active entries.

## Format compatibility

- **JSON** preserves the structured artifacts and all supported node kinds.
- **Clash/Mihomo** merges every user's proxies and front groups, prefixing names to avoid cross-user collisions. MTProto cannot be represented by this serializer; ambiguous or unresolved front-group references are rejected. An empty fleet is rejected instead of returning a misleading empty config.
- **URI** uses the complete URI serializer for every user's artifact. If any user's entries require front-proxy routing or self-signed TLS trust that a URI cannot preserve, the entire request is rejected; it never returns a partial list.
- **sing-box** is not currently supported by this API.

Unsupported format values receive `406 Not Acceptable`. Compatibility failures also receive `406` with an `incompatible_nodes` error. An empty JSON fleet remains a valid response with its current revision; empty Clash and URI exports are rejected.

## Example

```sh
export BROCADE_EXTERNAL_API_TOKEN='the-token-configured-on-the-server'
curl --fail --silent --show-error \
  -H "Authorization: Bearer $BROCADE_EXTERNAL_API_TOKEN" \
  'https://brocade.example/api/external/v1/nodes?format=json'
```
