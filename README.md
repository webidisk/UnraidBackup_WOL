Wake-on-LAN Template Release: v1.0.0-wol-template

This is a separate template branch for backup servers without BMC/IPMI. It keeps the reporting, no-delete policy, Docker handling, PID-lock fix, critical-share priority, and clean shutdown behavior from the current backup solution.

The template uses placeholders instead of your server-specific IPs, MAC addresses, or usernames.

Downloads
Main WOL backup script — weekly_unraid_backup_wol_template_v1.0.0.sh
WOL report helper — weekly_unraid_report_wol_template_v1.0.0.sh
Step-by-step README — Markdown
Step-by-step README — Plain Text
Complete WOL template bundle — ZIP
Placeholders Used
REPLACE_WITH_SOURCE_SERVER_IP
REPLACE_WITH_BACKUP_SERVER_IP
REPLACE_WITH_BACKUP_SERVER_MAC
REPLACE_WITH_NETWORK_BROADCAST_IP
REPLACE_WITH_SOURCE_SERVER_NETWORK_INTERFACE
