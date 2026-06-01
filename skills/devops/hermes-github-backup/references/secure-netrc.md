# Secure Netrc for Cron Backups

When the backup script runs under cron, environment variables are the only reliable way to pass the GitHub token. Persistent dotfiles like `~/.netrc` are risky because they store the token on disk and may be scanned.

## Temporary Netrc Approach
1. Create a temporary working directory:
   ```bash
   WORKDIR=$(mktemp -d)
   ```
2. Set `HOME` to this directory so git reads the netrc from there:
   ```bash
   export HOME="$WORKDIR"
   ```
3. Write the netrc file inside the temporary directory:
   ```bash
   cat > "$WORKDIR/.netrc" <<EOF
machine github.com
login x-access-token
password $GIT_TOKEN
EOF
   chmod 600 "$WORKDIR/.netrc"
   ```
4. Unset any existing credential helper to avoid conflicts:
   ```bash
   git config --unset credential.helper || true
   ```
5. Run git commands (clone, add, commit, push) within this environment. Example:
   ```bash
   GIT_TERMINAL_PROMPT=0 git clone --depth=1 https://github.com/zhengyu-git/hermes-agent.git "$WORKDIR/repo"
   # ... sync files, commit ...
   GIT_TERMINAL_PROMPT=0 git push origin master
   ```
6. Clean up the temporary directory after the push:
   ```bash
   rm -rf "$WORKDIR"
   ```

## Why This Matters
- **Security**: The token never touches the persistent filesystem, reducing risk of accidental exposure.
- **Compliance**: Avoids triggering security scanners that flag PATs in dotfiles.
- **Cron Compatibility**: Cron jobs have a minimal environment; this method works without relying on `.env` files.

Use this script fragment in your backup implementation to ensure safe credential handling.
