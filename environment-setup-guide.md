# Mattermost Test Environment Setup Guide

This guide will walk you through setting up a local Mattermost server and testing the Task plugin.

## Option 1: Docker Setup (Recommended - Easiest)

### Prerequisites
- Docker and Docker Compose installed
- Git installed
- Node.js 14+ and npm installed
- Go 1.16+ installed

### Step 1: Clone and cd into the git directory

### Step 2: Set Up Mattermost Server

1. Start Mattermost:

    **For Apple Silicon (M1/M2/M3):**
    ```bash
    # Set environment to use AMD64 (Rosetta emulation)
    export DOCKER_DEFAULT_PLATFORM=linux/amd64
    
    # Start services (will use AMD64 with Rosetta)
    docker-compose up -d
    ```

    **For Intel/AMD processors:**
    ```bash
    docker-compose up -d
    ```

2. Wait about 30 seconds for Mattermost to fully start, then open your browser to:
    ```
    http://localhost:8065
    ```

3. Create your admin account:
   - Fill in email, username, and password
   - Create a team (e.g., "Test Team")
   - Create a channel or use "Town Square"

### Step 3: Set Up Plugin Development Environment

Initialize Go module in server directory:
```bash
cd plugin/server
go mod init github.com/yourusername/mattermost-channel-task/server
go get github.com/mattermost/mattermost-server/v6/model
go get github.com/mattermost/mattermost-server/v6/plugin
cd ..
```

### Step 4: Build and Install Plugin

1. Build the plugin:
    ```bash
    # From the plugin directory
    make clean
    make all
    ```

2. The plugin bundle will be created at `dist/com.mattermost.channel-task-1.0.0.tar.gz`

3. Install the plugin in Mattermost:
   - Go to http://localhost:8065
   - Click on the menu (9 dots) → System Console
   - Navigate to Plugins → Plugin Management
   - Click "Upload Plugin"
   - Select the `.tar.gz` file
   - Click "Enable" next to the plugin

### Step 5: Test the Plugin

1. Go back to your team
2. Navigate to any channel (e.g., "Town Square")
3. Look for the checkmark icon in the App Bar on the right
4. Click it to open the task sidebar
5. Try:
   - Adding a task
   - Creating a group
   - Assigning a task to yourself
   - Checking off a task
   - Deleting a task

## Option 2: Local Development Server Setup

If you want a full development environment:

1. Clone the Mattermost server:
    ```bash
    git clone https://github.com/mattermost/mattermost-server.git
    cd mattermost-server
    ```

2. Follow the official Mattermost development setup guide:
https://developers.mattermost.com/contribute/developer-setup/

## Troubleshooting

### Platform mismatch error (ARM64/Apple Silicon)
**The issue:** Mattermost doesn't currently provide native ARM64 Docker images, so we need to use AMD64 with Rosetta emulation.

**Solution:**
1. Make sure Docker Desktop has Rosetta enabled:
   - Open Docker Desktop
   - Go to Settings → General
   - Enable "Use Rosetta for x86/amd64 emulation on Apple Silicon"
   - Restart Docker Desktop

2. Clean up and restart:
    ```bash
    docker-compose down
    docker system prune -a  # This removes old images
    export DOCKER_DEFAULT_PLATFORM=linux/amd64
    docker-compose up -d
    ```

3. Use the ARM64 version of docker-compose.yml (without platform specifications) so Docker automatically uses Rosetta

### Plugin doesn't appear
- Check Docker logs: `docker-compose logs mattermost`
- Verify plugin is enabled in System Console
- Check plugin upload size limits in System Console → Environment → File Storage

### Build errors
- Ensure Go version is 1.16+: `go version`
- Ensure Node.js version is 14+: `node --version`
- Clear build cache: `make clean` then `make all`

### Plugin won't enable
- Check Mattermost logs in System Console → Server Logs
- Verify `plugin.json` is valid JSON
- Ensure minimum server version matches your Mattermost version

### Task list doesn't open
- Open browser console (F12) and check for JavaScript errors
- Verify webapp bundle was built correctly
- Check Network tab for failed API requests

## Next Steps

Once you have the plugin working, you can:
- Modify the UI styling
- Add more features (due dates, priorities, etc.)
- Implement notifications
- Add slash commands for quick task management

## Useful Commands

```bash
# Rebuild and reinstall plugin quickly
make clean && make all && docker cp dist/com.mattermost.channel-task-1.0.0.tar.gz <container_id>:/tmp/

# View Mattermost logs
docker-compose logs -f mattermost

# Restart Mattermost
docker-compose restart mattermost

# Stop everything
docker-compose down

# Start fresh (removes all data)
docker-compose down -v
```
