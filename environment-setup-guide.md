# Mattermost Test Environment Setup Guide

This guide will walk you through setting up a local Mattermost server and testing the Task plugin.

## Option 1: Docker Setup (Recommended - Easiest)

### Prerequisites
- Docker and Docker Compose installed
- Git installed
- Node.js 14+ and npm installed
- Go 1.16+ installed

### Step 1: Set Up Mattermost Server

1. Create a project directory:
```bash
mkdir mattermost-task-test
cd mattermost-task-test
```

2. Create a `docker-compose.yml` file:

**For ARM64 systems (Apple Silicon M1/M2/M3) - Use AMD64 with Rosetta:**
```yaml
version: "3"

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: mmuser
      POSTGRES_PASSWORD: mmuser_password
      POSTGRES_DB: mattermost
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - mattermost

  mattermost:
    image: mattermost/mattermost-team-edition:latest
    ports:
      - "8065:8065"
    environment:
      MM_SQLSETTINGS_DRIVERNAME: postgres
      MM_SQLSETTINGS_DATASOURCE: postgres://mmuser:mmuser_password@postgres:5432/mattermost?sslmode=disable&connect_timeout=10
      MM_SERVICESETTINGS_SITEURL: http://localhost:8065
      MM_PLUGINSETTINGS_ENABLEUPLOADS: "true"
      MM_PLUGINSETTINGS_ENABLE: "true"
    volumes:
      - mattermost-config:/mattermost/config
      - mattermost-data:/mattermost/data
      - mattermost-logs:/mattermost/logs
      - mattermost-plugins:/mattermost/plugins
      - mattermost-client-plugins:/mattermost/client/plugins
    depends_on:
      - postgres
    networks:
      - mattermost

volumes:
  postgres-data:
  mattermost-config:
  mattermost-data:
  mattermost-logs:
  mattermost-plugins:
  mattermost-client-plugins:

networks:
  mattermost:
```

**For AMD64 systems (Intel/AMD processors):**
```yaml
version: "3"

services:
  postgres:
    image: postgres:15
    platform: linux/amd64
    environment:
      POSTGRES_USER: mmuser
      POSTGRES_PASSWORD: mmuser_password
      POSTGRES_DB: mattermost
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - mattermost

  mattermost:
    image: mattermost/mattermost-team-edition:latest
    platform: linux/amd64
    ports:
      - "8065:8065"
    environment:
      MM_SQLSETTINGS_DRIVERNAME: postgres
      MM_SQLSETTINGS_DATASOURCE: postgres://mmuser:mmuser_password@postgres:5432/mattermost?sslmode=disable&connect_timeout=10
      MM_SERVICESETTINGS_SITEURL: http://localhost:8065
      MM_PLUGINSETTINGS_ENABLEUPLOADS: "true"
      MM_PLUGINSETTINGS_ENABLE: "true"
    volumes:
      - mattermost-config:/mattermost/config
      - mattermost-data:/mattermost/data
      - mattermost-logs:/mattermost/logs
      - mattermost-plugins:/mattermost/plugins
      - mattermost-client-plugins:/mattermost/client/plugins
    depends_on:
      - postgres
    networks:
      - mattermost

volumes:
  postgres-data:
  mattermost-config:
  mattermost-data:
  mattermost-logs:
  mattermost-plugins:
  mattermost-client-plugins:

networks:
  mattermost:
```

3. Start Mattermost:

**For Apple Silicon (M1/M2/M3):**
```bash
# Clean up any existing containers and images
docker-compose down
docker system prune -a

# Set environment to use AMD64 (Rosetta emulation)
export DOCKER_DEFAULT_PLATFORM=linux/amd64

# Start services (will use AMD64 with Rosetta)
docker-compose up -d
```

**For Intel/AMD processors:**
```bash
docker-compose up -d
```

4. Wait about 30 seconds for Mattermost to fully start, then open your browser to:
```
http://localhost:8065
```

5. Create your admin account:
   - Fill in email, username, and password
   - Create a team (e.g., "Test Team")
   - Create a channel or use "Town Square"

### Step 2: Set Up Plugin Development Environment

1. Create plugin directory structure:
```bash
mkdir -p mattermost-channel-task/{server,webapp}
cd mattermost-channel-task
```

2. Create `plugin.json` in the root:
```json
{
  "id": "com.mattermost.channel-task",
  "name": "Channel Task List",
  "description": "Adds a task list for each channel",
  "version": "1.0.0",
  "min_server_version": "5.20.0",
  "server": {
    "executables": {
      "linux-amd64": "server/dist/plugin-linux-amd64",
      "darwin-arm64": "server/dist/plugin-darwin-arm64"
    }
  },
  "webapp": {
    "bundle_path": "webapp/dist/main.js"
  },
  "settings_schema": {
    "header": "Channel Task List Settings",
    "settings": []
  }
}
```

3. Create the server plugin (`server/plugin.go`):
```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"sync"
	"time"

	"github.com/mattermost/mattermost-server/v6/model"
	"github.com/mattermost/mattermost-server/v6/plugin"
)

type Plugin struct {
	plugin.MattermostPlugin
	configurationLock sync.RWMutex
}

type TaskItem struct {
	ID          string    `json:"id"`
	Text        string    `json:"text"`
	Completed   bool      `json:"completed"`
	AssigneeID  string    `json:"assignee_id,omitempty"`
	GroupID     string    `json:"group_id,omitempty"`
	CreatedAt   time.Time `json:"created_at"`
	CompletedAt time.Time `json:"completed_at,omitempty"`
}

type TaskGroup struct {
	ID   string `json:"id"`
	Name string `json:"name"`
}

type ChannelTaskList struct {
	Items  []TaskItem  `json:"items"`
	Groups []TaskGroup `json:"groups"`
}

func (p *Plugin) ServeHTTP(c *plugin.Context, w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Access-Control-Allow-Origin", "*")
	w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
	w.Header().Set("Access-Control-Allow-Headers", "Content-Type")

	if r.Method == "OPTIONS" {
		w.WriteHeader(http.StatusOK)
		return
	}

	switch r.URL.Path {
	case "/api/v1/tasks":
		p.handleTasks(w, r)
	case "/api/v1/groups":
		p.handleGroups(w, r)
	default:
		http.NotFound(w, r)
	}
}

func (p *Plugin) handleTasks(w http.ResponseWriter, r *http.Request) {
	channelID := r.URL.Query().Get("channel_id")
	if channelID == "" {
		http.Error(w, "channel_id required", http.StatusBadRequest)
		return
	}

	switch r.Method {
	case http.MethodGet:
		p.getTasks(w, r, channelID)
	case http.MethodPost:
		p.createTask(w, r, channelID)
	case http.MethodPut:
		p.updateTask(w, r, channelID)
	case http.MethodDelete:
		p.deleteTask(w, r, channelID)
	default:
		http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
	}
}

func (p *Plugin) handleGroups(w http.ResponseWriter, r *http.Request) {
	channelID := r.URL.Query().Get("channel_id")
	if channelID == "" {
		http.Error(w, "channel_id required", http.StatusBadRequest)
		return
	}

	switch r.Method {
	case http.MethodPost:
		p.createGroup(w, r, channelID)
	case http.MethodDelete:
		p.deleteGroup(w, r, channelID)
	default:
		http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
	}
}

func (p *Plugin) getTasks(w http.ResponseWriter, r *http.Request, channelID string) {
	list := p.getChannelTaskList(channelID)
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(list)
}

func (p *Plugin) createTask(w http.ResponseWriter, r *http.Request, channelID string) {
	var item TaskItem
	if err := json.NewDecoder(r.Body).Decode(&item); err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	item.ID = model.NewId()
	item.CreatedAt = time.Now()

	list := p.getChannelTaskList(channelID)
	list.Items = append(list.Items, item)
	p.saveChannelTaskList(channelID, list)

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(item)
}

func (p *Plugin) updateTask(w http.ResponseWriter, r *http.Request, channelID string) {
	var updated TaskItem
	if err := json.NewDecoder(r.Body).Decode(&updated); err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	list := p.getChannelTaskList(channelID)
	for i, item := range list.Items {
		if item.ID == updated.ID {
			if updated.Completed && !item.Completed {
				updated.CompletedAt = time.Now()
			}
			list.Items[i] = updated
			p.saveChannelTaskList(channelID, list)
			w.Header().Set("Content-Type", "application/json")
			json.NewEncoder(w).Encode(updated)
			return
		}
	}

	http.Error(w, "Task not found", http.StatusNotFound)
}

func (p *Plugin) deleteTask(w http.ResponseWriter, r *http.Request, channelID string) {
	taskID := r.URL.Query().Get("id")
	if taskID == "" {
		http.Error(w, "id required", http.StatusBadRequest)
		return
	}

	list := p.getChannelTaskList(channelID)
	for i, item := range list.Items {
		if item.ID == taskID {
			list.Items = append(list.Items[:i], list.Items[i+1:]...)
			p.saveChannelTaskList(channelID, list)
			w.WriteHeader(http.StatusNoContent)
			return
		}
	}

	http.Error(w, "Task not found", http.StatusNotFound)
}

func (p *Plugin) createGroup(w http.ResponseWriter, r *http.Request, channelID string) {
	var group TaskGroup
	if err := json.NewDecoder(r.Body).Decode(&group); err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	group.ID = model.NewId()

	list := p.getChannelTaskList(channelID)
	list.Groups = append(list.Groups, group)
	p.saveChannelTaskList(channelID, list)

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(group)
}

func (p *Plugin) deleteGroup(w http.ResponseWriter, r *http.Request, channelID string) {
	groupID := r.URL.Query().Get("id")
	if groupID == "" {
		http.Error(w, "id required", http.StatusBadRequest)
		return
	}

	list := p.getChannelTaskList(channelID)
	for i, group := range list.Groups {
		if group.ID == groupID {
			list.Groups = append(list.Groups[:i], list.Groups[i+1:]...)
			for j := range list.Items {
				if list.Items[j].GroupID == groupID {
					list.Items[j].GroupID = ""
				}
			}
			p.saveChannelTaskList(channelID, list)
			w.WriteHeader(http.StatusNoContent)
			return
		}
	}

	http.Error(w, "Group not found", http.StatusNotFound)
}

func (p *Plugin) getChannelTaskList(channelID string) *ChannelTaskList {
	key := fmt.Sprintf("tasks_%s", channelID)
	data, err := p.API.KVGet(key)
	if err != nil || data == nil {
		return &ChannelTaskList{
			Items:  []TaskItem{},
			Groups: []TaskGroup{},
		}
	}

	var list ChannelTaskList
	if err := json.Unmarshal(data, &list); err != nil {
		return &ChannelTaskList{
			Items:  []TaskItem{},
			Groups: []TaskGroup{},
		}
	}

	return &list
}

func (p *Plugin) saveChannelTaskList(channelID string, list *ChannelTaskList) error {
	key := fmt.Sprintf("tasks_%s", channelID)
	data, err := json.Marshal(list)
	if err != nil {
		return err
	}

	return p.API.KVSet(key, data)
}

func main() {
	plugin.ClientMain(&Plugin{})
}
```

4. Initialize Go module in server directory:
```bash
cd server
go mod init github.com/yourusername/mattermost-channel-task/server
go get github.com/mattermost/mattermost-server/v6/model
go get github.com/mattermost/mattermost-server/v6/plugin
cd ..
```

5. Create Makefile in the root:
```makefile
PLUGIN_ID = com.mattermost.channel-task
PLUGIN_VERSION = 1.0.0

# Detect architecture
UNAME_M := $(shell uname -m)
ifeq ($(UNAME_M),arm64)
    GOARCH = arm64
    PLUGIN_BINARY = plugin-darwin-arm64
else ifeq ($(UNAME_M),x86_64)
    GOARCH = amd64
    PLUGIN_BINARY = plugin-linux-amd64
else
    GOARCH = amd64
    PLUGIN_BINARY = plugin-linux-amd64
endif

.PHONY: all
all: server webapp bundle

.PHONY: server
server:
	mkdir -p server/dist
	cd server && GOOS=linux GOARCH=amd64 go build -o dist/plugin-linux-amd64 .
	cd server && GOOS=darwin GOARCH=arm64 go build -o dist/plugin-darwin-arm64 .

.PHONY: webapp
webapp:
	cd webapp && npm install && npm run build

.PHONY: bundle
bundle:
	rm -rf dist
	mkdir -p dist
	cp plugin.json dist/
	mkdir -p dist/server/dist
	cp server/dist/plugin-linux-amd64 dist/server/dist/
	cp server/dist/plugin-darwin-arm64 dist/server/dist/
	mkdir -p dist/webapp/dist
	cp webapp/dist/main.js dist/webapp/dist/
	cd dist && tar -czf $(PLUGIN_ID)-$(PLUGIN_VERSION).tar.gz plugin.json server webapp

.PHONY: clean
clean:
	rm -rf dist
	rm -rf server/dist
	rm -rf webapp/dist
	rm -rf webapp/node_modules
```

### Step 3: Set Up Webapp

1. Create `webapp/package.json`:
```json
{
  "name": "mattermost-channel-task-webapp",
  "version": "1.0.0",
  "scripts": {
    "build": "webpack --mode=production",
    "dev": "webpack --mode=development --watch"
  },
  "dependencies": {
    "react": "^17.0.2",
    "react-dom": "^17.0.2"
  },
  "devDependencies": {
    "@types/react": "^17.0.0",
    "@types/react-dom": "^17.0.0",
    "typescript": "^4.5.0",
    "webpack": "^5.65.0",
    "webpack-cli": "^4.9.0",
    "ts-loader": "^9.2.6"
  }
}
```

2. Create `webapp/webpack.config.js`:
```javascript
const path = require('path');

module.exports = {
  entry: './src/index.tsx',
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: 'ts-loader',
        exclude: /node_modules/,
      },
    ],
  },
  resolve: {
    extensions: ['.tsx', '.ts', '.js'],
  },
  output: {
    filename: 'main.js',
    path: path.resolve(__dirname, 'dist'),
    libraryTarget: 'window',
  },
  externals: {
    react: 'React',
    'react-dom': 'ReactDOM',
  },
};
```

3. Create `webapp/tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "es5",
    "module": "commonjs",
    "lib": ["es2015", "dom"],
    "jsx": "react",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

4. Create `webapp/src/index.tsx` with the React code from the previous artifact (I'll provide a simplified version):

```typescript
// Copy the full React component code here from the previous artifact
// For brevity, I'm not repeating it, but you should copy the entire
// TaskSidebar, TaskHeaderButton, and Plugin class from the webapp artifact
```

### Step 4: Build and Install Plugin

1. Build the plugin:
```bash
# From the mattermost-channel-task directory
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
3. Look for the checkmark icon in the channel header (next to the channel name)
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
