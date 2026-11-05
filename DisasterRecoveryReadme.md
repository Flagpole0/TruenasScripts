# TrueNAS Configuration Runbook

Generate a Markdown runbook containing TrueNAS configuration, storage health, and operational details, plus a downloadable configuration backup.

The script queries the TrueNAS REST API and organizes the results into a configuration archive and a live-state appendix. Use it for system documentation, audits, troubleshooting, and disaster-recovery preparation.

> [!CAUTION]
> The downloaded configuration backup includes the secret seed and is **not encrypted by this script**. The report can also contain sensitive information when redaction is disabled. Store both files securely, and never commit them to version control or share them publicly.

## Features

- Collects configuration from a range of TrueNAS API endpoints.
- Creates a structured Markdown runbook with a table of contents.
- Adds a deep-dive appendix for pool health, SMART results, encryption, iSCSI, updates, apps, audit activity, and more.
- Redacts many secret-like fields by default, with an interactive option to disable redaction.
- Downloads and validates a TrueNAS configuration backup tar archive.
- Skips unavailable endpoints and records the reason in the report.

## Requirements

- Python 3
- A TrueNAS API key with permission to read the requested configuration and download the configuration backup
- The `requests` package

Install the dependency with:

```bash
python3 -m pip install requests
```

## Usage

Run the script:

```bash
python3 DRHumanReadable
```

If settings are not provided through environment variables, the script prompts for the TrueNAS URL, API key, output directory, and redaction choice.

Alternatively, configure the connection using environment variables:

```bash
export TRUENAS_HOST="https://truenas.example.com"
export TRUENAS_APIKEY="your-api-key"
export TRUENAS_OUTPUT_DIR="$HOME/Documents/truenas-backup"
export TRUENAS_VERIFY_SSL="true"

python3 DRHumanReadable
```

| Variable | Default | Description |
| --- | --- | --- |
| `TRUENAS_HOST` | Prompted | TrueNAS base URL, such as `https://truenas.example.com` |
| `TRUENAS_APIKEY` | Prompted | API key sent as a bearer token |
| `TRUENAS_OUTPUT_DIR` | Prompted when unset or `.` | Directory for generated files |
| `TRUENAS_VERIFY_SSL` | `false` | Set to `true` to verify the server's TLS certificate |
| `TRUENAS_REDACT` | Interactive choice | Set to `false` to disable redaction |

For unattended runs, provide the connection settings in the environment. `TRUENAS_REDACT=false` disables redaction without prompting. Treat the resulting report as containing plaintext secrets.

## Generated files

Files are written to the selected output directory, using a timestamp in their names:

- `truenas_runbook_<timestamp>_redacted.md` — the Markdown report when redaction is enabled
- `truenas_runbook_<timestamp>_UNREDACTED.md` — the report when redaction is disabled
- `truenas_backup_<timestamp>.tar` — the downloaded TrueNAS configuration backup

The tar backup is downloaded with the secret seed included. Redaction applies to the Markdown report; it does **not** encrypt or redact the tar backup.

## Report contents

### Configuration archive

The first part of the report gathers information from API areas including:

- System settings, certificates, and boot environments
- Network configuration, interfaces, routes, and IPMI
- Pools, datasets, disks, snapshots, replication, and scrubs
- SMB, NFS, iSCSI, and WebDAV sharing
- Services and service configuration
- Directory services, users, and groups
- Cron, cloud sync, rsync, and SMART tasks
- Virtual machines, apps, alerts, and reporting
- Two-factor authentication, privileges, and API keys
- Legacy jails and plugins, where available

Some sections request selected fields; others include the endpoint's returned data. The report notes endpoints that are unavailable or fail to respond.

### Deep-dive appendix

Deep-dive collection is enabled by default with `DEEP_DIVE = True` in the script. It adds sections for:

1. Pool health, capacity, and vdev details
2. Disk and SMART health
3. Dataset encryption status
4. Cloud credential inventory
5. iSCSI portal, target, and extent relationships
6. Update and patch status
7. Application details
8. Recent audit activity
9. Dataset quota utilization
10. System dataset and syslog settings

Set `DEEP_DIVE = False` in the script to omit the appendix.

## Redaction and data handling

Redaction is enabled by default for the Markdown report. The script masks values for fields whose names match its configured sensitive-field list and patterns. This is a practical safeguard, not a guarantee that every sensitive value will be detected.

Review generated reports before sharing them. In particular:

- Unredacted reports may contain credentials, keys, or other secrets in plaintext.
- Redacted reports may still include sensitive infrastructure details, such as hostnames, addresses, usernames, and paths.
- The configuration tar archive includes the secret seed and is not encrypted by the script.
- Store outputs in access-controlled, encrypted storage. Do not commit generated files to a repository.

## TLS verification

TLS certificate verification is disabled by default (`TRUENAS_VERIFY_SSL=false`). For normal use, prefer enabling verification:

```bash
export TRUENAS_VERIFY_SSL="true"
```

If certificate verification is disabled, the connection may be vulnerable to interception. Use that setting only when you understand and accept the risk.

## Backup download

The script first attempts the `core.download` API flow and checks that the downloaded content resembles a tar archive. If that path fails, it tries the legacy `/config/save` endpoint. If neither produces a valid archive, the backup is not saved and an error is printed.

## Limitations

- Available endpoints and returned fields can vary between TrueNAS releases.
- Missing or unsupported endpoints are reported as skipped; a skipped section does not necessarily indicate a script failure.
- The script does not encrypt the generated report or backup.
- The configured redaction rules may not identify every sensitive value.

## Troubleshooting

- **Connection or authentication failure:** Check `TRUENAS_HOST`, the API key, network access, and API key permissions.
- **TLS verification error:** Confirm the TrueNAS certificate is trusted, or configure the appropriate certificate trust. Disabling verification should be a last resort.
- **Some sections are skipped:** Check whether the endpoint is supported by your TrueNAS version and whether the API key has access to it.
- **Backup download fails:** Check the API key's permissions and download the configuration manually from the TrueNAS UI if necessary.

## Security note

Run this script only against a TrueNAS system you administer or are authorized to access. Protect the API key and all generated files as sensitive data.
```
