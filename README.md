# Challenge 1

this is the script

```
#!/bin/bash

# Configuration
BACKUP_DIR="${BACKUP_DIR:-/opt/docker/backups}"
LOG_FILE="${LOG_FILE:-/var/log/docker-backup.log}"
DATE=$(date +%Y%m%d_%H%M%S)

# Logging function
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Create backup directory if it doesn't exist
mkdir -p "$BACKUP_DIR"

# Function to backup all Docker volumes
backup_volumes() {
    log "Starting backup process..."
    
    # Get list of all Docker volumes
    volumes=$(docker volume ls -q)
    
    for volume in $volumes; do
        backup_file="$BACKUP_DIR/${volume}_${DATE}.tar.gz"
        
        log "Backing up volume: $volume"
        
        # Create a temporary container to mount the volume and create backup
        if docker run --rm \
            -v "$volume":/source:ro \
            -v "$BACKUP_DIR":/backup \
            alpine \
            tar czf "/backup/$(basename "$backup_file")" -C /source .; then
            
            log "Successfully created backup: $backup_file"
        else
            log "ERROR: Failed to backup volume: $volume"
        fi
    done
    
    # Clean up old backups (keep last 7 days)
    find "$BACKUP_DIR" -name "*.tar.gz" -mtime +7 -delete
    
    log "Backup process completed"
}

# Function to restore volumes
restore_volumes() {
    local volume_name="$1"
    
    log "Starting restore process..."
    
    if [ -z "$volume_name" ]; then
        log "ERROR: Please specify a volume name to restore"
        exit 1
    fi
    
    # Find the latest backup for the specified volume
    latest_backup=$(ls -t "$BACKUP_DIR"/${volume_name}_*.tar.gz 2>/dev/null | head -n1)
    
    if [ -z "$latest_backup" ]; then
        log "ERROR: No backup found for volume: $volume_name"
        exit 1
    fi
    
    log "Restoring from backup: $latest_backup"
    
    # Check if volume exists, create if it doesn't
    if ! docker volume inspect "$volume_name" >/dev/null 2>&1; then
        docker volume create "$volume_name"
    fi
    
    # Restore data to volume
    if docker run --rm \
        -v "$volume_name":/destination \
        -v "$BACKUP_DIR":/backup \
        alpine \
        sh -c "cd /destination && tar xzf /backup/$(basename "$latest_backup")"; then
        
        log "Successfully restored volume: $volume_name"
    else
        log "ERROR: Failed to restore volume: $volume_name"
        exit 1
    fi
    
    log "Restore process completed"
}

# Function to list available backups
list_backups() {
    log "Available backups:"
    ls -lh "$BACKUP_DIR" | grep ".tar.gz"
}

# Main script logic
case "$1" in
    backup)
        backup_volumes
        ;;
    restore)
        restore_volumes "$2"
        ;;
    list)
        list_backups
        ;;
    *)
        echo "Usage: $0 {backup|restore <volume_name>|list}"
        exit 1
        ;;
esac

exit 0

```
and the cronjob work
```
# Open crontab editor
crontab -e

# Add this line to run backup daily at midnight
0 0 * * * /path/to/docker-backup.sh backup >> /var/log/docker-backup.log 2>&1
```
I used [Cluade](https://claude.ai/) for this purpose.

This is the script for according to the challenge which is capable of doing :
-   Automated backup of all Docker volumes
-   Compressed backups with timestamps
-   Backup rotation (keeps last 7 days)
-   Restore functionality for specific volumes
-   Detailed logging
-   Error handling and status reporting
-   Non-disruptive operation (uses read-only mounts for backup)
```command
# Make the script executable
chmod +x docker-backup.sh

# Create required directories
sudo mkdir -p /opt/docker/backups
sudo touch /var/log/docker-backup.log
sudo chown -R $(whoami):$(whoami) /opt/docker/backups /var/log/docker-backup.log 
```

i have already created a volume by name `test_volume` and `my_data_volume` and mounted them to containers `test` and 
`test2` respectivley and both have ubuntu as image.

`test` have a `test.txt` file at /home/app directory and `test2` have a `test.txt` file at /app directory.

the content of both of test file


Now time for checking the script
