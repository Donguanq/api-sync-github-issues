# GitHub Issues Sync — n8n + PostgreSQL

A self-hosted automation workflow that synchronizes open GitHub issues into PostgreSQL using n8n. The project demonstrates API integration, pagination, data normalization, idempotent database writes, synchronization validation, error handling, and Telegram notifications.

## Project Overview

The workflow retrieves open issues from the [`n8n-io/n8n`](https://github.com/n8n-io/n8n) repository, transforms the API response into normalized records, and upserts them into PostgreSQL.

It also detects issues that were previously open but are missing from a complete current result set. Instead of assuming that a missing issue was closed, it fetches the issue directly from GitHub and updates its stored status.

The workflow supports both manual execution and scheduled runs.

## Architecture

```text
Manual Trigger ───────┐
                      ├──> GitHub HTTP Request
Schedule Trigger ─────┘            |
                                   v
                            Normalize Issues
                                   |
                                   v
                         Upsert GitHub Issues
                                   |
                                   v
                          Verify Sync Count
                                   |
                                   v
                            IF Sync Complete
                                   |
                                   v
                           Find Missing Issues
                                   |
                                   v
                         GET Missing Issue Status
                                   |
                                   v
                       Update Verified Issue Status
```

Error handling is implemented in a separate workflow:

```text
Error Trigger
     |
     v
Telegram Send Message
```

The main workflow uses the error workflow configured in n8n's workflow settings.

## Key Features

- **REST API integration:** Retrieves issues using GitHub's Search API.
- **Pagination:** Requests up to 100 results per page and combines the returned pages.
- **Data normalization:** Converts API responses into individual n8n items and maps the required fields.
- **Completeness validation:** Checks the API response, detects incomplete results, and verifies that the number of retrieved records matches GitHub's reported count.
- **Idempotent upserts:** Uses PostgreSQL `INSERT ... ON CONFLICT DO UPDATE` to update existing records instead of creating duplicates.
- **Sync tracking:** Uses `sync_run_id` to associate records with a particular execution.
- **Missing issue detection:** Identifies previously open issues absent from the current complete result set and checks their actual status through GitHub's API.
- **Retry handling:** Configures retry attempts for temporary HTTP Request and PostgreSQL upsert failures.
- **Error notifications:** Sends Telegram notifications for unhandled workflow errors.
- **Scheduled execution:** Supports automated daily synchronization alongside manual runs.

## Data Model

The PostgreSQL table is named `github_issues_sync`.

| Column | Purpose |
|---|---|
| `repository` | Repository identifier |
| `github_id` | GitHub's unique issue identifier |
| `issue_number` | Issue number within the repository |
| `title` | Issue title |
| `state` | Current issue state |
| `author_login` | GitHub username of the author |
| `created_at` | Creation timestamp |
| `updated_at` | Last update timestamp reported by GitHub |
| `closed_at` | Closing timestamp, when applicable |
| `html_url` | Link to the issue |
| `comments_count` | Number of comments |
| `locked` | Whether the issue is locked |
| `labels` | JSONB array of label names |
| `body` | Issue description |
| `sync_run_id` | n8n execution identifier used for sync tracking |
| `last_synced_at` | Last time the record was synchronized |

A composite primary key on `(repository, github_id)` prevents duplicate records for the same GitHub issue. A unique constraint on `(repository, issue_number)` provides an additional integrity check.

## Synchronization Logic

1. Request open issues from GitHub using the query `repo:n8n-io/n8n is:issue state:open -is:pr`.
2. Fetch pages and combine the returned results.
3. Validate the response and normalize each issue.
4. Upsert all retrieved issues into PostgreSQL, tagging them with the current execution ID.
5. Verify that the number of records tagged by this execution matches GitHub's reported result count.
6. Only after successful validation, identify previously open records absent from the current run.
7. Fetch the current GitHub status of each missing issue and update the corresponding database record.

Missing issues are not marked closed solely because they were absent from the search results.

## Testing and Validation

The workflow was tested with the following scenarios:

- Retrieved and normalized 350 issues in a test run.
- Inserted the records into PostgreSQL successfully.
- Re-ran the workflow and verified that existing records were updated rather than duplicated.
- Verified that all 350 records were tagged with the current `sync_run_id`.
- Verified the total row count in PostgreSQL.
- Confirmed that the missing-issue query returned no records when the current result set was complete and unchanged.
- Tested the separate error workflow using a `Stop And Error` node and confirmed that a Telegram notification was received.

The value of 350 represents the result count observed during testing, not a fixed expected count for future runs.

## Technology Stack

- n8n — workflow orchestration
- GitHub REST API — issue retrieval
- PostgreSQL — persistent storage
- SQL — table design, parameterized queries, upserts, and validation
- JavaScript — pagination aggregation and data transformation in a Code node
- Telegram — error notifications
- Docker — self-hosted environment

## Setup

1. Run n8n and PostgreSQL.
2. Create the `github_issues_sync` table using the database schema used by the workflow.
3. Configure PostgreSQL credentials in n8n.
4. Configure any required Telegram credentials and the destination chat.
5. Import the main workflow and the error workflow from their exported JSON files.
6. Reassign credentials where necessary and configure the main workflow's error workflow.
7. Verify the GitHub API request and PostgreSQL connection.
8. Test a manual execution and confirm the synchronization count.
9. Publish or activate the main workflow to enable scheduled execution.

Do not commit credentials, access tokens, passwords, or other secrets to the repository.

## Limitations and Future Improvements

- GitHub's Search API limits the number of results available through a single search query. If the number of matching issues exceeds the API's search-result limit, the workflow must use a different retrieval strategy rather than silently accepting a partial synchronization.
- Retry policies cover a limited number of attempts. More advanced handling could distinguish rate limits, transient server errors, and permanent errors.
- A production deployment should include monitoring of failed executions and periodic verification of database state.
- Additional improvements could include automated tests, structured run logs, configurable repository filters, and a dedicated sync-run audit table.

## Skills Demonstrated

REST API integration, pagination, JSON processing, JavaScript transformations, SQL, PostgreSQL data modeling, idempotency, data integrity validation, error handling, retry strategies, and workflow automation with n8n.

## Screenshots

### Workflow Overview
![Workflow overview](screenshots/workflow-overview.png)

### Synchronization Validation
![PostgreSQL sync validation](screenshots/postgres-results.png)

### Error Notifications
![Telegram error notification](screenshots/telegram-error-notification.png)
