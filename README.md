# VPS Backup to GitHub with n8n

An automated n8n workflow that backs up Docker container directories from a VPS to a GitHub repository with transparent encryption using git-crypt.

## Overview

This workflow monitors a VPS backup directory for new Docker container backups, compresses them into timestamped archives, and automatically pushes them to a GitHub repository with AES-256 encryption. The workflow uses git-crypt to handle encryption transparently, eliminating complex key management typically required for secure backup solutions.

## Features

- Automated detection of new backup folders
- Timestamped compressed archives (tar.gz format)
- Transparent file encryption using git-crypt (AES-256)
- Automatic Git commit and push to GitHub
- Success/failure notifications
- Cleanup of local archives after successful push
- Daily scheduled execution at 9 PM IST
- Stateful tracking to avoid duplicate processing

## Prerequisites

### Required Software

- n8n workflow automation platform
- SSH access to your VPS
- Git installed on VPS
- git-crypt installed on VPS
- GitHub repository with git-crypt initialized
- SSH Password credentials configured in n8n

### VPS Setup

1. **Install git-crypt**
   ```bash
   sudo apt install git-crypt
   ```

2. **Initialize Git repository in backup directory**
   ```bash
   cd /root/Backups/docker-backups
   git init
   git remote add origin git@github.com:yourusername/your-backup-repo.git
   ```

3. **Initialize git-crypt**
   ```bash
   cd /root/Backups/docker-backups
   git-crypt init
   ```

4. **Export git-crypt key** (store securely)
   ```bash
   git-crypt export-key ../git-crypt-key/docker-backups.key
   ```

5. **Configure .gitattributes** (to encrypt all tar.gz files)
   ```bash
   echo "*.tar.gz filter=git-crypt diff=git-crypt" > .gitattributes
   git add .gitattributes
   git commit -m "Configure git-crypt for tar.gz files"
   git push origin main
   ```

## Workflow Architecture

The workflow consists of 17 nodes organized in a sequential pipeline:

### 1. Schedule Trigger
- **Node:** Daily 9PM
- **Type:** Schedule Trigger
- **Configuration:** Runs daily at 21:00 (9 PM IST)
- **Purpose:** Initiates the workflow automatically

### 2. List Backup Folders
- **Node:** List Backup Folders on VPS
- **Type:** SSH
- **Command:** `ls -1d /root/Backups/docker-backups/*/`
- **Purpose:** Retrieves list of all backup directories on VPS

### 3. Identify New Folders
- **Node:** Identify New Folders
- **Type:** Code (JavaScript)
- **Purpose:** Compares current folders with previously synced folders stored in workflow static data
- **Logic:**
  - Parses SSH output into folder list
  - Retrieves previously synced folders from static data
  - Filters out already processed folders
  - Updates static data with current folder list
  - Returns only new folders for processing

### 4. Check for New Folders
- **Node:** Has New Folders?
- **Type:** IF
- **Condition:** `isNewFolder === true`
- **Purpose:** Routes workflow based on whether new folders exist

### 5. No New Folders Path
- **Node:** No New Folders Message
- **Type:** No Operation
- **Purpose:** Terminates workflow gracefully when no new backups are found

### 6. Prepare Archive Details
- **Node:** Prepare Archive Details
- **Type:** Code (JavaScript)
- **Purpose:** Generates timestamp and archive filename
- **Output Format:** `backup-{foldername}-{timestamp}.tar.gz`
- **Timestamp Format:** `YYYYMMDD-HHMMSS`

### 7. Create Compressed Archive
- **Node:** Create Compressed Archive
- **Type:** SSH
- **Command:** `tar -czf "{archiveName}" -C "{folderPath}" .`
- **Purpose:** Creates gzip-compressed tar archive of backup folder

### 8. Verify Archive Created
- **Node:** Verify Archive Created
- **Type:** SSH
- **Command:** `ls -lh "{archiveName}"`
- **Purpose:** Confirms archive was created successfully and retrieves file size

### 9. Parse Archive Info
- **Node:** Parse Archive Info
- **Type:** Code (JavaScript)
- **Purpose:** Extracts file size from ls output and prepares metadata for Git operations

### 10. Git Add Archive File
- **Node:** Git Add Archive File
- **Type:** SSH
- **Command:** `git add "{archiveName}"`
- **Purpose:** Stages the archive file for commit (git-crypt encrypts transparently during this step)

### 11. Git Commit with Message
- **Node:** Git Commit with Message
- **Type:** SSH
- **Command:** `git commit -m "Automated backup: {folderName} - {timestamp} ({fileSize})"`
- **Purpose:** Commits the encrypted archive with descriptive message

### 12. Git Push to GitHub
- **Node:** Git Push to GitHub
- **Type:** SSH
- **Command:** `git push origin main`
- **Purpose:** Pushes encrypted backup to GitHub repository

### 13. Verify Push Success
- **Node:** Verify Push Success
- **Type:** Code (JavaScript)
- **Purpose:** Parses git push output to confirm successful upload
- **Success Indicators:** Checks for "main -> main", "To github.com", or "Writing objects" in output

### 14. Check Push Status
- **Node:** Push Successful?
- **Type:** IF
- **Condition:** `success === true`
- **Purpose:** Routes to success or failure path

### 15a. Success Path: Cleanup Local Archive
- **Node:** Cleanup Local Archive
- **Type:** SSH
- **Command:** `rm -f "{archiveName}"`
- **Purpose:** Removes local archive file after successful push to save disk space

### 15b. Failure Path: Log Push Failure
- **Node:** Log Push Failure
- **Type:** Code (JavaScript)
- **Purpose:** Creates detailed failure log with git output for troubleshooting

### 16a. Success Path: Generate Success Summary
- **Node:** Generate Success Summary
- **Type:** Code (JavaScript)
- **Purpose:** Creates comprehensive success report with:
  - Folder name and archive name
  - File size
  - Timestamp
  - GitHub repository URL
  - Encryption confirmation
  - Completion timestamp

### 16b. Failure Path: Send Failure Notification
- **Node:** Send Failure Notification
- **Type:** Execute Workflow
- **Purpose:** Triggers separate error notification workflow with failure details

## Installation

1. **Import the workflow**
   - Open n8n
   - Go to Workflows
   - Click "Import from File" or "Import from URL"
   - Select the `VPS-Backup-to-Github-2.json` file

2. **Configure SSH credentials**
   - Navigate to Credentials in n8n
   - Create new SSH Password credential
   - Enter your VPS connection details (host, port, username, password)
   - Save credential

3. **Update SSH credential references**
   - Open imported workflow
   - For each SSH node, select your configured SSH credential
   - Save workflow

4. **Configure backup path** (if different from `/root/Backups/docker-backups/`)
   - Update path in "List Backup Folders on VPS" node
   - Update path in all subsequent SSH commands that reference the backup directory

5. **Configure error notification** (optional)
   - Update or remove "Send Failure Notification" node
   - If using, create separate error notification workflow and update workflow ID

6. **Activate workflow**
   - Click "Active" toggle to enable scheduled execution

## Important Considerations

### GitHub File Size Limits

GitHub imposes the following limits on file uploads:

- **Hard limit:** 100 MB per file via command line
- **Browser limit:** 25 MB per file
- **Warning threshold:** 50 MB per file
- **Recommended repository size:** Under 1 GB total
- **Maximum practical size:** 5 GB per repository

**For this workflow:**
- Each backup archive is pushed as a single file
- Archives exceeding 100 MB will be rejected by GitHub
- Monitor your Docker container backup sizes to ensure they stay within limits
- Consider implementing archive splitting for large backups
- Alternative: Use Git LFS (Large File Storage) for files between 100 MB and 2 GB

### git-crypt Benefits

git-crypt provides several advantages for this backup workflow:

1. **Transparent Encryption**
   - Files are encrypted automatically during `git add`
   - No manual encryption/decryption commands needed
   - Works seamlessly with standard Git workflow

2. **Simplified Key Management**
   - Single symmetric key for entire repository
   - No need for complex certificate infrastructure
   - Key can be exported and backed up securely
   - Easy to share with authorized team members

3. **Selective Encryption**
   - Only files matching .gitattributes patterns are encrypted
   - Repository metadata remains readable
   - Commit history is not encrypted

4. **AES-256 Security**
   - Industry-standard encryption (AES-256 in CTR mode)
   - Cryptographically secure for sensitive backup data
   - No performance overhead for small to medium files

5. **No Complex Setup**
   - Much simpler than GPG-based encryption workflows
   - No key servers or public/private key pairs required
   - Eliminates permission issues common with other encryption methods

**Alternative approaches and their drawbacks:**
- **Manual encryption scripts:** Requires encryption/decryption in workflow, adding complexity
- **GPG encryption:** Complex key management, requires keyring setup
- **SSH keys with restricted repos:** Only provides access control, not encryption
- **Third-party encryption services:** Additional dependencies and API management

## Workflow Customization

### Change Schedule
Edit the "Daily 9PM" Schedule Trigger node:
```javascript
{
  "rule": {
    "interval": [
      {
        "triggerAtHour": 21  // Change to desired hour (24-hour format)
      }
    ]
  }
}
```

### Modify Backup Path
Update the following variables in SSH nodes:
- List Backup Folders: `/root/Backups/docker-backups/*/`
- All tar/git commands: `/root/Backups/docker-backups`

### Adjust Archive Naming
Modify the "Prepare Archive Details" Code node:
```javascript
const archiveName = `backup-${folderName}-${timestamp}.tar.gz`;
// Change to your preferred naming convention
```

### Change Git Branch
Update "Git Push to GitHub" command:
```bash
git push origin main
# Change 'main' to your target branch
```

## Troubleshooting

### Common Issues

**Problem:** SSH connection fails
- **Solution:** Verify SSH credentials in n8n, check VPS firewall rules, ensure SSH service is running

**Problem:** Git push rejected
- **Solution:** Check GitHub repository permissions, verify git remote URL, ensure git-crypt is unlocked

**Problem:** Archive size exceeds GitHub limits
- **Solution:** Implement archive splitting, use Git LFS, or reduce backup scope

**Problem:** Static data not persisting
- **Solution:** Ensure workflow is saved after first run, check n8n database connectivity

**Problem:** git-crypt not encrypting files
- **Solution:** Verify .gitattributes configuration, check git-crypt initialization, ensure pattern matches archive files

### Debug Mode

To troubleshoot workflow execution:
1. Click on any node
2. View execution data in right panel
3. Check stdout/stderr for SSH nodes
4. Examine JSON output for Code nodes

## Security Best Practices

1. **Secure git-crypt key storage**
   - Store exported key on encrypted external media
   - Keep multiple secure backups
   - Never commit key to repository

2. **SSH credential management**
   - Use SSH keys instead of passwords when possible
   - Rotate credentials periodically
   - Limit SSH user permissions to backup directory only

3. **GitHub repository access**
   - Use private repositories for backups
   - Enable two-factor authentication
   - Review repository access logs regularly
   - Use dedicated deploy keys with minimal permissions

4. **VPS security**
   - Keep system packages updated
   - Configure firewall rules
   - Monitor unauthorized access attempts
   - Regularly audit backup directory permissions

-----
