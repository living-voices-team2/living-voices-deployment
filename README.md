
# Living Voices Deployment

Shared development infrastructure for the Living Voices project.

## Prerequisites

- Docker Desktop
- Docker Compose
- Java 21
- Git
- HTTPie or cURL

## 1. Configure Keycloak Hostname

Add the following entry to your system hosts file:

    127.0.0.1 keycloak

### macOS and Linux

    echo "127.0.0.1 keycloak" | sudo tee -a /etc/hosts

Skip this command if the entry already exists.

### Windows

Open Notepad as Administrator and edit:

    C:\Windows\System32\drivers\etc\hosts

Add the hostname entry and save the file.

## 2. Start the Platform

From the root of this repository, run:

    docker compose -f docker/docker-compose.yml up -d

Verify that all containers are running:

    docker compose -f docker/docker-compose.yml ps

## 3. Platform Services

| Service | Host | Port |
|---|---|---|
| PostgreSQL | localhost | 5432 |
| Redis | localhost | 6379 |
| RabbitMQ | localhost | 5672 |
| RabbitMQ Management | localhost | 15672 |
| Keycloak | keycloak | 8080 |

### PostgreSQL

Username: user

Password: password

The following databases are initialized automatically on a fresh PostgreSQL data directory:

- recording
- transcription
- dictionary

Connection URLs:

    jdbc:postgresql://localhost:5432/recording
    jdbc:postgresql://localhost:5432/transcription
    jdbc:postgresql://localhost:5432/dictionary

### Redis

    localhost:6379

### RabbitMQ

Username: user

Password: password

Management dashboard:

    http://localhost:15672

### Keycloak

Administration console:

    http://keycloak:8080/admin/

Administrator username: admin

Administrator password: password

Realm: LivingVoices

Issuer URI:

    http://keycloak:8080/realms/LivingVoices

## 4. Authentication Configuration

### Development Users

| Username | Password | Role |
|---|---|---|
| learner1 | password | learner |
| committee1 | password | committee |

### Edge Service Client

- Client ID: edge-service
- Client secret: living-voices-dev-edge-secret
- Client type: Confidential

### Testing Client

- Client ID: living-voices-test
- Client type: Public
- Direct access grants: Enabled

Both clients include a realm role mapper that exposes roles
through the JWT `roles` claim.

## 5. Generate Test Access Tokens

### Learner Token

    LEARNER_TOKEN=$(curl -s -X POST \
      http://keycloak:8080/realms/LivingVoices/protocol/openid-connect/token \
      -d "grant_type=password" \
      -d "client_id=living-voices-test" \
      -d "username=learner1" \
      -d "password=password" \
      | python3 -c 'import json,sys; print(json.load(sys.stdin)["access_token"])')

### Committee Token

    COMMITTEE_TOKEN=$(curl -s -X POST \
      http://keycloak:8080/realms/LivingVoices/protocol/openid-connect/token \
      -d "grant_type=password" \
      -d "client_id=living-voices-test" \
      -d "username=committee1" \
      -d "password=password" \
      | python3 -c 'import json,sys; print(json.load(sys.stdin)["access_token"])')

These commands require Python 3 and a compatible shell.

Access tokens expire after five minutes.

## 6. Verify Infrastructure

Check PostgreSQL databases:

    docker exec postgres psql -U user -c '\l'

Check Redis:

    docker exec redis redis-cli ping

Expected Redis response:

    PONG

Check Keycloak logs:

    docker logs --tail 100 keycloak

Verify the Keycloak issuer:

    curl -s http://keycloak:8080/realms/LivingVoices/.well-known/openid-configuration

## 7. Stop the Platform

Stop all platform containers without removing them:

    docker compose -f docker/docker-compose.yml stop

Restart the platform:

    docker compose -f docker/docker-compose.yml up -d

## 8. Troubleshooting

### Port Already in Use

Ensure that another local PostgreSQL, Redis, RabbitMQ,
or Keycloak instance is not occupying the required ports.

On macOS or Linux, check PostgreSQL port usage:

    sudo lsof -nP -iTCP:5432 -sTCP:LISTEN

### Keycloak Hostname Not Resolving

Confirm that the hosts file contains:

    127.0.0.1 keycloak

### Keycloak Realm Not Imported

Check the Keycloak container logs and confirm that
keycloak/realm-config.json exists.

Keycloak startup import skips realms that already exist.
Changes to the JSON file may require a fresh local realm
or manual configuration updates.

## 9. Security Notice

All credentials, client secrets, test accounts, and configuration
settings in this repository are intended exclusively for local
development and testing.

Do not expose these services publicly or reuse these credentials
in a production environment.
