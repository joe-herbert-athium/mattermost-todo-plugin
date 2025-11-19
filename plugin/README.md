# Mattermost Channel Todo List Plugin

A Mattermost plugin that adds a comprehensive todo list feature to each channel with grouping and assignment capabilities.

## Features

- **Channel-Specific Todo Lists**: Each channel has its own independent todo list
- **Header Button**: Quick access button in the channel header showing the count of incomplete todos
- **Right-Hand Sidebar**: Full-featured todo list interface that opens in the right sidebar
- **Todo Management**: Add, complete, and delete todo items
- **Grouping**: Organize todos into custom groups
- **Assignment**: Assign todos to channel members
- **Persistent Storage**: All todos are stored in Mattermost's key-value store

## Project Structure

```
mattermost-channel-todo/
├── plugin.json              # Plugin manifest
├── server/
│   └── plugin.go           # Backend Go code
└── webapp/
    └── index.tsx           # Frontend React components
```

## Installation

1. Build the plugin:
   ```bash
   make
   ```

2. Upload the generated `.tar.gz` file to your Mattermost server:
    - Go to **System Console** > **Plugins** > **Management**
    - Click **Upload Plugin**
    - Select the plugin file

3. Enable the plugin:
    - Click **Enable** next to the plugin name

## Usage

### Accessing the Todo List

1. In any channel, look for the checkmark (✓) icon with a number in the channel header
2. Click the icon to open the todo sidebar on the right

### Adding Todos

1. Type your todo text in the input field
2. Optionally select a group from the dropdown
3. Click "Add Todo" or press Enter

### Managing Todos

- **Complete/Uncomplete**: Click the checkbox next to any todo
- **Assign**: Select a channel member from the dropdown next to each todo
- **Delete**: Click the × button to remove a todo

### Creating Groups

1. Click "Add Group" button
2. Enter a group name
3. Click "Create Group"
4. When adding new todos, select the group from the dropdown

### Deleting Groups

- Click "Delete Group" button next to the group name
- All todos in that group will be moved to "Ungrouped"

## Development

### Prerequisites

- Go 1.16+
- Node.js 14+
- Mattermost Server 5.20+

### Building

```bash
# Install dependencies
cd webapp
npm install

# Build the plugin
cd ..
make
```

### API Endpoints

The plugin exposes the following REST API endpoints:

#### Get Todos
```
GET /plugins/com.mattermost.channel-todo/api/v1/todos?channel_id={channelId}
```

#### Create Todo
```
POST /plugins/com.mattermost.channel-todo/api/v1/todos?channel_id={channelId}
Body: {
  "text": "Todo text",
  "group_id": "optional_group_id"
}
```

#### Update Todo
```
PUT /plugins/com.mattermost.channel-todo/api/v1/todos?channel_id={channelId}
Body: {
  "id": "todo_id",
  "text": "Updated text",
  "completed": true,
  "assignee_id": "user_id",
  "group_id": "group_id"
}
```

#### Delete Todo
```
DELETE /plugins/com.mattermost.channel-todo/api/v1/todos?channel_id={channelId}&id={todoId}
```

#### Create Group
```
POST /plugins/com.mattermost.channel-todo/api/v1/groups?channel_id={channelId}
Body: {
  "name": "Group name"
}
```

#### Delete Group
```
DELETE /plugins/com.mattermost.channel-todo/api/v1/groups?channel_id={channelId}&id={groupId}
```

## Data Structure

### TodoItem
```typescript
{
  id: string;
  text: string;
  completed: boolean;
  assignee_id?: string;
  group_id?: string;
  created_at: string;
  completed_at?: string;
}
```

### TodoGroup
```typescript
{
  id: string;
  name: string;
}
```

## Storage

All data is stored in Mattermost's built-in key-value store with keys in the format:
```
todos_{channelId}
```

Each key contains a JSON object with the complete todo list and groups for that channel.

## Contributing

Contributions are welcome! Please feel free to submit issues or pull requests.

## License

MIT License