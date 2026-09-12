# FOMA CMS

Headless CMS for the **Foundation of Mastering Automation (FOMA)** platform. This repository contains the Strapi backend used to manage articles and newsletters that are consumed by the FOMA website and supporting services.

## Overview

FOMA CMS is built with **Strapi 5** and exposes a REST API for editorial content. Content is authored in the Strapi Admin panel, stored in the configured database, and published for consumption by the FOMA frontend.

The CMS is part of the wider FOMA platform:

```text
┌──────────────────────┐
│   FOMA Website       │
│   foma.life          │
└──────────┬───────────┘
           │ REST API
           ▼
┌──────────────────────┐
│      FOMA CMS        │
│      Strapi 5        │
├──────────────────────┤
│ Articles             │
│ Newsletters          │
│ Media                │
│ Admin / RBAC         │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Database             │
│ SQLite / PostgreSQL  │
│ / MySQL config       │
└──────────────────────┘
           │
           │ Newsletter publish event
           ▼
┌──────────────────────┐
│ FOMA AWS Backend     │
│ API Gateway + Lambda │
│ + Amazon SES         │
└──────────────────────┘
```

> **Important:** the newsletter email delivery flow depends on a Strapi webhook configured in the Strapi Admin panel. The webhook is not stored as repository code.

## Technology Stack

| Component | Technology |
|---|---|
| CMS | Strapi 5.41.1 |
| Runtime | Node.js 20–24.x |
| Language | TypeScript / JavaScript |
| Frontend for Admin | React 18 |
| Default local database | SQLite |
| Supported databases | SQLite, PostgreSQL, MySQL |
| API | Strapi REST API |
| Authentication | Strapi Users & Permissions / Admin authentication |
| Cloud plugin | `@strapi/plugin-cloud` |
| Package manager | npm |

The exact versions and runtime constraints are defined in `package.json` and the lockfile.

## Repository Structure

```text
foma-cms/
├── config/
│   ├── admin.ts          # Admin authentication, API token and encryption configuration
│   ├── api.ts            # REST API limits
│   ├── database.ts       # SQLite / PostgreSQL / MySQL database configuration
│   ├── middlewares.ts    # Strapi middleware stack
│   ├── plugins.ts        # Installed Strapi plugins and security settings
│   └── server.ts         # Host, port and application keys
├── src/
│   ├── admin/             # Admin panel customization/configuration
│   └── api/
│       ├── article/       # Article content type, controller, routes and service
│       └── newsletter/    # Newsletter content type, controller, routes and service
├── public/
│   ├── uploads/           # Local media upload location
│   └── robots.txt
├── database/migrations/   # Database migrations
├── .env.example           # Environment variable template
├── package.json
└── package-lock.json
```

## Content Models

### Article

Articles are published content entries intended for the FOMA website.

Fields currently defined:

- `title` — article title
- `description` — short description/summary
- `content` — Strapi Blocks rich content
- `image` — optional media asset
- `category` — optional category
- `date` — article date

Articles use Strapi draft/publish, allowing editorial work to remain unpublished until ready.

### Newsletter

Newsletters are editorial messages that can be published from Strapi and used to trigger the FOMA email distribution workflow.

Fields currently defined:

- `title` — newsletter title
- `description` — short description
- `content` — Strapi Blocks content
- `image` — optional media asset
- `category` — optional category
- `date` — required newsletter date
- `author` — optional author, defaulting to `William Foma`

Newsletters also use Strapi draft/publish.

## Newsletter Publishing Flow

Publishing a newsletter is more than a CMS action: it is an integration event across the FOMA platform.

```text
Editor
  │
  ▼
Strapi Admin
  │
  │ Publish newsletter
  ▼
Strapi Webhook
  │  entry.publish
  ▼
AWS API Gateway
  │  POST /prod/newsletter/publish
  ▼
Lambda: foma-newsletter-publish-prod
  │
  ├── Read OPT-IN contacts from Amazon SES
  ├── Fetch published newsletter content when required
  ├── Send email through Amazon SES
  └── Record send state in DynamoDB
```

### Production webhook

The production FOMA website backend exposes the newsletter publishing endpoint at:

```text
https://7gd3r709wf.execute-api.ap-southeast-1.amazonaws.com/prod/newsletter/publish
```

Configure this endpoint in:

**Strapi Admin → Settings → Webhooks**

Recommended webhook:

- **Name:** `FOMA Newsletter Publisher`
- **URL:** the endpoint above
- **Event:** `Entry → Publish` / `entry.publish`
- **Model/content type:** newsletter

No webhook configuration is committed to this repository, so a fresh environment must be configured explicitly.

### Troubleshooting newsletter delivery

If users are subscribed but do not receive a newsletter:

1. Confirm the Strapi webhook exists and is enabled.
2. Publish a test newsletter from Strapi.
3. Check the `foma-newsletter-publish-prod` Lambda logs in CloudWatch.
4. Confirm the Lambda reports contacts with `OPT_IN` status for the `FOMA-Newsletter` topic.
5. Inspect failed SES sends and delivery/suppression status.
6. Confirm the newsletter can be fetched from the CMS API when the webhook payload does not contain the complete content.

A useful success log from the publisher reports counters such as `contacts`, `sent`, `skipped`, and `failed`.

## Local Development

### Prerequisites

- Node.js `20.x` or newer, up to the supported `24.x` range
- npm `6+`
- A local SQLite database for the simplest setup, or PostgreSQL/MySQL for environments matching production

### Install dependencies

```bash
npm ci
```

### Configure environment variables

Copy the example file:

```bash
cp .env.example .env
```

Generate real, unique secrets for every environment. Do not reuse development secrets in production.

At minimum, configure the values represented in `.env.example`:

```text
HOST=0.0.0.0
PORT=1337
APP_KEYS=...
API_TOKEN_SALT=...
ADMIN_JWT_SECRET=...
TRANSFER_TOKEN_SALT=...
JWT_SECRET=...
ENCRYPTION_KEY=...
```

For PostgreSQL or MySQL, also set the database variables supported by `config/database.ts`.

### Start Strapi

Development mode with auto-reload:

```bash
npm run develop
```

Production-style start:

```bash
npm run build
npm run start
```

The default Strapi port is `1337`.

## Database Configuration

The application supports three database clients through `config/database.ts`:

- **SQLite** — default for local development; data is stored under `.tmp/data.db` unless `DATABASE_FILENAME` is overridden.
- **PostgreSQL** — suitable for hosted/production deployments.
- **MySQL** — supported through the same environment-driven configuration.

Connection pools, SSL settings and connection timeout are configurable through environment variables.

For production, use a managed database with backups, monitoring, controlled network access and TLS where applicable.

## REST API

The REST configuration currently uses:

- Default page limit: `25`
- Maximum page limit: `100`
- Response count enabled

Example content endpoints:

```text
GET /api/articles
GET /api/articles/:id
GET /api/newsletters
GET /api/newsletters/:id
```

Exact accessibility depends on Strapi Users & Permissions settings and the permissions assigned to public/authenticated roles.

## Security

The project already follows several sound practices:

- Secrets are supplied through environment variables.
- Admin JWT signing, API token salts and transfer token salts are externally configured.
- Encryption key is externally configured.
- Strapi's security middleware is enabled.
- The repository does not require committing production credentials.
- Database selection and sensitive connection options are environment-driven.

### Production security requirements

Before exposing the CMS publicly:

- Replace every placeholder secret from `.env.example`.
- Never commit `.env` or production credentials.
- Restrict admin access to trusted users and roles.
- Review public API permissions and expose only content that should be public.
- Prefer PostgreSQL/MySQL with TLS for production rather than local SQLite.
- Protect the Strapi admin endpoint with strong authentication and appropriate network controls.
- Keep Strapi and npm dependencies regularly updated.
- Back up the production database and uploaded media.
- Configure and secure the newsletter webhook. Treat it as an integration endpoint, not as a general-purpose public API.

## Webhooks and Integrations

Webhook configuration is external to the repository. This distinction matters operationally: deploying this repository does **not** automatically create the Strapi newsletter webhook.

The newsletter integration expects the publishing event to reach the FOMA AWS backend. The AWS side then handles recipient selection, Amazon SES sending, and duplicate-send protection.

## Media and Uploads

Strapi local uploads are stored beneath:

```text
public/uploads/
```

The repository keeps the directory structure with `.gitkeep`; generated media should not be treated as application source code.

For production, use a durable, backed-up storage strategy appropriate to the deployment architecture rather than relying on ephemeral application storage.

## Upgrading Strapi

The repository provides the following helper commands:

```bash
npm run upgrade
npm run upgrade:dry
```

Use the dry-run option before a real upgrade and review release notes, migration requirements, dependency changes and compatibility issues before updating production.

## Operational Checklist

### Before publishing an article

- Content is complete and reviewed.
- Image/media references resolve correctly.
- The publication date is correct.
- The entry is published only when ready.

### Before publishing a newsletter

- Newsletter content is reviewed.
- The Strapi webhook is enabled.
- The AWS publisher endpoint is reachable.
- Subscriber records are opted in to the FOMA newsletter topic.
- The SES sending identity is verified and operational.

### After publishing a newsletter

- Check the Strapi webhook delivery result.
- Check Lambda CloudWatch logs.
- Verify the number of contacts processed and emails sent.
- Investigate any failed or skipped recipients.
- Confirm delivery in a controlled test inbox.

## Relationship to `foma-website`

`foma-cms` is the editorial/content-management side of the FOMA platform. The companion repository `foma-website` contains the public website and the AWS serverless backend responsible for registration, newsletter subscription and newsletter distribution.

Keeping the responsibilities separated provides a clean architecture:

```text
foma-cms
  └── Content authoring + publishing

foma-website
  ├── Public frontend
  └── AWS serverless backend
      ├── Newsletter subscription
      ├── Newsletter publication handler
      ├── DynamoDB state
      └── Amazon SES delivery
```

## Contributing

Use small, focused commits and keep content-model changes, configuration changes and dependency upgrades reviewable.

Before opening a change:

```bash
npm ci
npm run build
```

Also verify the affected content type and any external integrations in a non-production environment before publishing.

## License

No explicit license file is currently defined in this repository. Treat the repository as **all rights reserved** unless a license is added by the project owner.

## Project

**Foundation of Mastering Automation (FOMA)**

- Website: https://foma.life
- CMS repository: https://github.com/Fomainspi/foma-cms
- Website/backend repository: https://github.com/Fomainspi/foma-website
