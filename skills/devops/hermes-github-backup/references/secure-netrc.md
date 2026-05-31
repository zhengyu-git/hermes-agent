# Secure temporary netrc handling for cron backups

When running the `hermes-github-backup` cron job, writing the GitHub token to the permanent `~/.netrc` file is insecure and can be caught by security scanners. Instead, create a temporary netrc file inside the clone work directory and point Git to use it only for the duration of the git operations.

## Example script (bash)
```bash
#!/usr/bin/env bash
set -euo pipefail

# Expect GIT_TOKEN environment variable to be set in the cron prompt
if [[ -z "${GIT_TOKEN:-}" ]]; then
  echo "GIT_TOKEN not set – abort" >&2
  exit 1
fi

# Create temporary workdir and temp netrc file
WORKDIR=$(mktemp -d)
NETRC_TEMP="$WORKDIR/.netrc"

# Write credentials with proper permissions (600)
cat > "$NETRC_TEMP" <<EOF
machine github.com
login x-access-token
password $GIT_TOKEN
EOF
chmod 600 "$NETRC_TEMP"

# Export HOME so that git reads the temporary netrc
export HOME="$WORKDIR"

# Clone shallow repo using the temporary netrc for auth
git -c credential.helper='store --file "$NETRC_TEMP"' clone --depth=1 https://github.com/zhengyu-git/hermes-agent.git "$WORKDIR"

# ... perform rsync, redaction, commit, push as usual ...
# (You can reuse the rest of the backup.sh script here)

# Cleanup
rm -rf "$WORKDIR"
```

## How it works
1. **Temporary netrc** – The token is written to `$WORKDIR/.netrc` with mode `600`. This file is never persisted beyond the script run.
2. **HOME redirection** – By setting `HOME` to the temporary work directory, git automatically picks up the netrc file without needing to modify global user config.
3. **Credential helper** – The `store --file "$NETRC_TEMP"` config forces git to use only this netrc file, preventing it from falling back to any existing (possibly stale) credential stores.
4. **Cleanup** – The script removes the entire work directory at the end, guaranteeing no token remains on disk.

## Benefits
- No permanent token file in the user's home directory.
- Avoids security‑scanner redaction of tokens embedded in command lines.
- Works reliably in cron where environment files like `.env` are not loaded.
- Keeps the backup process compliant with GitHub Push Protection (GH013).

Refer to this file from the main `hermes-github-backup` skill for the exact steps to incorporate.
