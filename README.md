
A small web application lab built with Docker to practice deploying and securing a web service.

The goal of this project was not to build a complex application. I wanted to create something close to a small real-world environment where I could practice:

- Docker and Docker Compose
- Docker networking
- Nginx reverse proxy
- HTTPS/TLS
- Linux permissions
- Container hardening
- PostgreSQL
- Security headers
- Rate limiting
- Health checks
- Basic security testing
- Troubleshooting

## Architecture

The lab has three containers:

```text
                    Client
                      |
                      | HTTPS :8443
                      v
              +---------------+
              |     Nginx     |
              | Reverse Proxy |
              +-------+-------+
                      |
                      | HTTP :5000
                      v
              +---------------+
              | Flask / Gunicorn|
              |      App       |
              +-------+-------+
                      |
                      | PostgreSQL
                      v
              +---------------+
              |   PostgreSQL   |
              |      DB        |
              +---------------+
```

The application is not directly exposed to the host.

Only Nginx is exposed:

```text
Host :8080  -> Nginx :80
Host :8443  -> Nginx :443
```

The Flask application listens internally on port `5000`.

PostgreSQL is also only available inside the Docker network.

## Project Structure

```text
secure-web-lab/
│
├── app/
│   ├── app.py
│   ├── Dockerfile
│   └── requirements.txt
│
├── proxy/
│   ├── nginx.conf
│   └── certs/
│       ├── server.crt
│       └── server.key
│
├── secrets/
│
├── compose.yaml
└── .gitignore
```

Sensitive files such as certificates, private keys and secrets are excluded from Git.

## Application

The application is a small Flask API.

The main endpoint used for testing is:

```text
GET /health
```

A successful response confirms that the application is running and can reach PostgreSQL.

Example:

```text
database reachable
```

Gunicorn is used instead of Flask's development server.

The application container also runs as a non-root user.

## Docker

Docker Compose is used to manage the three services:

```text
proxy
app
db
```

Start the lab:

```bash
sudo docker compose up -d
```

Check the containers:

```bash
sudo docker compose ps
```

Stop the lab:

```bash
sudo docker compose down
```

View logs:

```bash
sudo docker compose logs
```

View logs for one service:

```bash
sudo docker compose logs proxy
sudo docker compose logs app
sudo docker compose logs db
```

## Nginx Reverse Proxy

Nginx is placed in front of the application.

This means clients never communicate directly with Flask.

HTTP traffic on port `8080` is redirected to HTTPS:

```text
http://server:8080
        |
        v
      301
        |
        v
https://server:8443
```

HTTPS traffic is terminated by Nginx and then forwarded to the Flask application.

The application itself does not need to handle TLS.

## TLS

The lab supports:

```text
TLS 1.2
TLS 1.3
```

The certificate is currently a local/self-signed certificate because this is a lab environment.

For a real production deployment, the certificate should come from a trusted CA such as Let's Encrypt.

The private key is not committed to Git.

## Security Headers

Nginx adds several security headers.

The current configuration includes:

```text
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy
```

These help reduce some common browser-based attacks and unwanted content handling.

## Rate Limiting

Nginx also has request rate limiting enabled.

The current configuration uses:

```text
10 requests/second
burst: 20
```

This is mainly to demonstrate how a reverse proxy can limit excessive requests before they reach the application.

During testing, requests above the configured limit returned:

```text
429 Too Many Requests
```

This confirmed that the rate limiting was working.

## Container Hardening

The containers were hardened where practical.

The configuration includes:

- `read_only`
- `cap_drop: ALL`
- non-root application user
- temporary filesystems where required
- minimal Alpine/Slim images
- health checks
- services isolated through the Docker network

The application container uses:

```text
python:3.12-slim
```

The Nginx container uses:

```text
nginx:1.27-alpine
```

PostgreSQL uses:

```text
postgres:16-alpine
```

## Health Checks

The application has a health endpoint:

```bash
curl -k https://localhost:8443/health
```

Expected result:

```text
HTTP 200
database reachable
```

Docker Compose also reports the application and database as healthy.

Example:

```text
app    Up (healthy)
db     Up (healthy)
proxy  Up
```

## Testing

### HTTP Redirect

Test:

```bash
curl -I http://localhost:8080
```

Expected:

```text
HTTP/1.1 301 Moved Permanently
```

The request should be redirected to HTTPS.

### HTTPS

Test:

```bash
curl -k -I https://localhost:8443/health
```

The response should contain:

```text
HTTP 200
```

and the configured security headers.

### Nginx Configuration

Check the configuration:

```bash
sudo docker compose exec proxy nginx -t
```

Expected:

```text
syntax is ok
test is successful
```

Reload after configuration changes:

```bash
sudo docker compose exec proxy nginx -s reload
```

### Nmap

The host was scanned using Nmap.

The expected externally accessible ports for this lab are:

```text
22/tcp     SSH
8080/tcp   HTTP
8443/tcp   HTTPS
```

The application and PostgreSQL ports are not exposed directly to the host.

This is important because users should access the application through Nginx rather than directly connecting to Flask or PostgreSQL.

## Load Testing

I also tested the rate limiting using multiple requests.

A sequential test of 50 HTTPS requests returned:

```text
50/50 HTTP 200
```

A concurrent test initially produced:

```text
28 HTTP 200
72 HTTP 503
```

The Nginx logs showed that requests were being limited.

After fixing the Nginx configuration and reloading it, the test produced:

```text
33 HTTP 200
67 HTTP 429
```

The `429` responses confirmed that Nginx was actively enforcing the configured rate limit.

The exact results depend on the machine and test method, so these numbers are mainly recorded as part of the lab testing rather than as a performance benchmark.

## Troubleshooting

### Docker permission error

Some Docker commands required `sudo` because the current user was not configured for direct access to the Docker socket.

Example:

```bash
sudo docker compose ps
```

### Nginx could not read the TLS key

The Nginx container runs as a non-root user.

The private key initially had restrictive ownership/permissions that prevented Nginx from reading it.

The problem was investigated using:

```bash
namei -l proxy/certs/server.key
```

The permissions and directory ownership were corrected so that the Nginx process could access the certificate key without making the key unnecessarily public.

### Nginx configuration errors

During configuration changes, I encountered syntax errors such as:

```text
unexpected EOF
```

and:

```text
location ... no "{"
```

The configuration was tested before reloading:

```bash
sudo docker compose exec proxy nginx -t
```

This helped catch configuration problems before applying them.

## What I Learned

This project gave me practical experience with several areas that are useful for junior infrastructure and cloud/security roles.

### Docker

I learned how to:

- build images
- run containers
- use Docker Compose
- connect containers through a Docker network
- use health checks
- restrict container capabilities
- run applications as non-root
- troubleshoot container problems

### Nginx

I practiced:

- reverse proxy configuration
- HTTP to HTTPS redirection
- TLS configuration
- security headers
- rate limiting
- configuration testing
- log-based troubleshooting

### Linux

I worked with:

- file permissions
- ownership
- processes
- services
- networking
- SSH
- Docker permissions
- troubleshooting from the command line

### Security

The main lesson was that securing an application is not one setting.

Security was applied in several layers:

```text
Network
   ↓
Nginx
   ↓
TLS
   ↓
HTTP security headers
   ↓
Rate limiting
   ↓
Container hardening
   ↓
Non-root application
   ↓
Database isolation
```

## Limitations

This is a lab environment, not a production deployment.

Some things would need to be improved for production:

## Conclusion

It helped me understand how the different components fit together and how to troubleshoot problems when the configuration doesn't work as expected.
