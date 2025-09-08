# Dify Local Dev Quick Guide

A concise runbook for starting the local dev stack and initializing Dify via API.

## 1) Prerequisites

- Docker + Docker Compose v2
- Node.js 18+ and pnpm
- uv (Astral)
- Dify repository cloned. Set an absolute path: `export REPO="/absolute/path/to/dify"`

## 2) Start middleware (Postgres, Redis, etc.)

```bash
cd "$REPO/docker"
cp middleware.env.example middleware.env
docker compose -f docker-compose.middleware.yaml up -d
```

## 3) Start services

```bash
pnpm --prefix "$REPO/web" install
"$REPO/dev/start-api" > "$REPO/api.log" 2>&1 &
"$REPO/dev/start-worker" > "$REPO/worker.log" 2>&1 &
pnpm --prefix "$REPO/web" dev > "$REPO/web/web.log" 2>&1 &
```

## 4) Initialize via API (first-run)

```bash
curl -X POST http://localhost:5001/console/api/setup \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","name":"admin","password":"Password123"}'
```
Expected: `{ "result": "success" }`

Then obtain tokens:
```bash
curl -X POST http://localhost:5001/console/api/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"Password123"}'
```

## 5) Verify / troubleshoot

- Check containers: `docker compose -f "$REPO/docker/docker-compose.middleware.yaml" ps`
- Tail logs: `tail -f "$REPO/api.log" "$REPO/worker.log" "$REPO/web/web.log"`

Notes:
- If setup was already completed, skip step 4 and use the login endpoint directly.
- Backend Python commands: prefix with `uv run --project api` when needed.




# Dify Frontend API Operations Documentation

Based on frontend service layer code analysis, the following are the main backend API operations that the Dify frontend can call. All APIs requiring authentication need to include `Authorization: Bearer {access_token}` in the request headers.

## Basic Configuration

- **Console API Prefix**: `http://localhost:5001/console/api`
- **Public API Prefix**: `http://localhost:5001/api`
- **Marketplace API Prefix**: `http://localhost:5002/api` (for plugin marketplace)
- **Authentication Method**: Bearer Token (via access_token)

## 1. Authentication and Account Management

### 1.1 Account Initialization
```bash
POST /console/api/setup
Content-Type: application/json

{
    "email": "testuser@example.com",
    "name": "testuser", 
    "password": "Password123"
}
```

### 1.2 User Login
```bash
POST /console/api/login
Content-Type: application/json

{
    "email": "testuser@example.com",
    "password": "Password123"
}
```

### 1.3 Refresh Access Token
```bash
POST /console/api/refresh-token
Authorization: Bearer {refresh_token}

{
    "refresh_token": "{refresh_token}"
}
```

### 1.4 User Profile Management
```bash
# Get user profile
GET /console/api/account/profile
Authorization: Bearer {access_token}

# Update user profile
POST /console/api/account/profile
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "name": "new_name",
    "avatar": "avatar_url"
}
```

### 1.5 Password Management
```bash
# Send forgot password verification code
POST /console/api/forgot-password
Content-Type: application/json

{
    "email": "user@example.com",
    "language": "en-US"
}

# Reset password
POST /console/api/forgot-password/validity
Content-Type: application/json

{
    "email": "user@example.com",
    "code": "123456",
    "token": "reset_token",
    "new_password": "NewPassword123",
    "password_confirm": "NewPassword123"
}
```

## 2. Workspace Management

### 2.1 Workspace Information
```bash
# Get current workspace information
GET /console/api/workspaces/current
Authorization: Bearer {access_token}

# Update workspace information
POST /console/api/workspaces/current
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "name": "workspace_name",
    "description": "workspace_description"
}

# Get workspace list
GET /console/api/workspaces
Authorization: Bearer {access_token}

# Switch workspace
POST /console/api/workspaces/{workspace_id}/switch
Authorization: Bearer {access_token}
```

### 2.2 Member Management
```bash
# Get member list
GET /console/api/workspaces/current/members
Authorization: Bearer {access_token}

# Invite member
POST /console/api/workspaces/current/members/invite
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "email": "member@example.com",
    "role": "normal"
}

# Update member role
PUT /console/api/workspaces/current/members/{member_id}
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "role": "admin"
}

# Delete member
DELETE /console/api/workspaces/current/members/{member_id}
Authorization: Bearer {access_token}
```

## 3. Application Management

### 3.1 Basic Application Operations
```bash
# Get application list
GET /console/api/apps?page=1&limit=20
Authorization: Bearer {access_token}

# Get explore/app market application list
GET /console/api/explore/apps
Authorization: Bearer {access_token}

# Get explore application details
GET /console/api/explore/apps/{app_id}
Authorization: Bearer {access_token}

# Get application details
GET /console/api/apps/{app_id}
Authorization: Bearer {access_token}

# Create application
POST /console/api/apps
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "name": "My App",
    "mode": "chat",
    "icon_type": "emoji",
    "icon": "🤖",
    "description": "App description"
}

# Update application information
PUT /console/api/apps/{app_id}
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "name": "Updated App Name",
    "description": "Updated description"
}

# Delete application
DELETE /console/api/apps/{app_id}
Authorization: Bearer {access_token}

# Copy application
POST /console/api/apps/{app_id}/copy
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "name": "Copied App",
    "mode": "chat"
}
```

### 3.2 Application Import/Export
```bash
# Export application configuration
GET /console/api/apps/{app_id}/export?include_secret=false
Authorization: Bearer {access_token}

# Import application/workflow from DSL (create new)
POST /console/api/apps/imports
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "mode": "yaml_content",
    "yaml_content": "yaml_content_here",
    "name": "Imported App",
    "description": "Imported from DSL",
    "icon_type": "emoji",
    "icon": "🤖"
}

# Import DSL from URL
POST /console/api/apps/imports
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "mode": "yaml_url",
    "yaml_url": "https://example.com/workflow.yml",
    "name": "Imported Workflow",
    "description": "Imported from URL"
}

# Update existing application DSL (overwrite)
POST /console/api/apps/imports
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "mode": "yaml_content",
    "yaml_content": "updated_yaml_content_here",
    "app_id": "existing_app_id"
}

# Confirm DSL import
POST /console/api/apps/imports/{import_id}/confirm
Authorization: Bearer {access_token}

# Directly update workflow draft (deprecated, recommend using above method)
POST /console/api/apps/{app_id}/workflows/draft/import
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "data": "yaml_dsl_content"
}
```

**DSL Import Instructions:**
- The `mode` parameter supports two modes:
  - `yaml_content`: Directly pass YAML content
  - `yaml_url`: Get YAML content from URL
- If `app_id` parameter is provided, it will update existing application; otherwise create new application
- Import operation is asynchronous, needs to be confirmed through `confirm` interface
- DSL import supported statuses: `pending`, `completed`, `completed-with-warnings`, `failed`

### 3.3 Application Configuration
```bash
# Update application model configuration
POST /console/api/apps/{app_id}/model-config
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "model": {
        "provider": "openai",
        "name": "gpt-3.5-turbo",
        "mode": "chat",
        "completion_params": {
            "temperature": 0.7
        }
    }
}

# Update application site status
POST /console/api/apps/{app_id}/site/enable
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "enable_site": true
}

# Update API status
POST /console/api/apps/{app_id}/api/enable
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "enable_api": true
}
```

## 4. Workflow Management

### 4.1 Workflow Operations
```bash
# Get workflow draft
GET /console/api/apps/{app_id}/workflows/draft
Authorization: Bearer {access_token}

# Sync workflow draft
POST /console/api/apps/{app_id}/workflows/draft
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "graph": {},
    "features": {},
    "environment_variables": [],
    "conversation_variables": []
}

# Get node default configuration
GET /console/api/apps/{app_id}/workflows/default-workflow-block-configs/{block_type}
Authorization: Bearer {access_token}

# Single node execution
POST /console/api/apps/{app_id}/workflows/draft/nodes/{node_id}/run
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "inputs": {},
    "query": "test query"
}

# Stop workflow execution
POST /console/api/apps/{app_id}/workflows/draft/stop
Authorization: Bearer {access_token}

# Update workflow draft from DSL (deprecated, recommend using apps/imports)
POST /console/api/apps/{app_id}/workflows/draft/import
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "data": "yaml_dsl_content_string"
}
```

### 4.2 Execution History
```bash
# Get workflow execution history
GET /console/api/apps/{app_id}/workflows/run-history?page=1&limit=20
Authorization: Bearer {access_token}

# Get chat execution history
GET /console/api/apps/{app_id}/chat-messages?conversation_id={conversation_id}
Authorization: Bearer {access_token}
```

## 5. Dataset Management

### 5.1 Basic Dataset Operations
```bash
# Get dataset list
GET /console/api/datasets?page=1&limit=20
Authorization: Bearer {access_token}

# Get dataset details
GET /console/api/datasets/{dataset_id}
Authorization: Bearer {access_token}

# Create empty dataset
POST /console/api/datasets
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "name": "My Dataset"
}

# Update dataset settings
PATCH /console/api/datasets/{dataset_id}
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "name": "Updated Dataset",
    "description": "Updated description",
    "indexing_technique": "high_quality"
}

# Delete dataset
DELETE /console/api/datasets/{dataset_id}
Authorization: Bearer {access_token}
```

### 5.2 Document Management
```bash
# Create document
POST /console/api/datasets/{dataset_id}/documents
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "name": "document.txt",
    "text": "Document content",
    "indexing_technique": "high_quality",
    "process_rule": {
        "mode": "automatic"
    }
}

# Get indexing status
GET /console/api/datasets/{dataset_id}/documents/{document_id}/indexing-status
Authorization: Bearer {access_token}

# Rename document
POST /console/api/datasets/{dataset_id}/documents/{document_id}/rename
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "name": "new_document_name"
}

# Pause document indexing
PATCH /console/api/datasets/{dataset_id}/documents/{document_id}/processing/pause
Authorization: Bearer {access_token}

# Resume document indexing
PATCH /console/api/datasets/{dataset_id}/documents/{document_id}/processing/resume
Authorization: Bearer {access_token}
```

### 5.3 Hit Testing
```bash
# Execute hit testing
POST /console/api/datasets/{dataset_id}/hit-testing
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "query": "test query",
    "retrieval_model": {
        "search_method": "semantic_search",
        "reranking_enable": false,
        "reranking_model": {
            "reranking_provider_name": "",
            "reranking_model_name": ""
        },
        "top_k": 2,
        "score_threshold_enabled": false
    }
}

# Get test records
GET /console/api/datasets/{dataset_id}/queries?page=1&limit=20
Authorization: Bearer {access_token}
```

## 6. Model Provider Management

### 6.1 Model Provider Operations
```bash
# Get model provider list
GET /console/api/workspaces/current/model-providers
Authorization: Bearer {access_token}

# Get model provider credentials
GET /console/api/workspaces/current/model-providers/{provider}/credentials
Authorization: Bearer {access_token}

# Validate model provider
POST /console/api/workspaces/current/model-providers/{provider}/test
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "credentials": {
        "api_key": "your_api_key"
    }
}

# Set model provider
POST /console/api/workspaces/current/model-providers/{provider}
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "credentials": {
        "api_key": "your_api_key"
    }
}

# Delete model provider
DELETE /console/api/workspaces/current/model-providers/{provider}
Authorization: Bearer {access_token}
```

### 6.2 Model Management
```bash
# Get model list
GET /console/api/workspaces/current/models
Authorization: Bearer {access_token}

# Get default model
GET /console/api/workspaces/current/default-model
Authorization: Bearer {access_token}

# Update default model
POST /console/api/workspaces/current/default-model
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "model": "gpt-3.5-turbo"
}
```

## 7. Tool Management

### 7.1 Tool Provider Operations
```bash
# Get tool provider list
GET /console/api/workspaces/current/tool-providers
Authorization: Bearer {access_token}

# Get built-in tool list
GET /console/api/workspaces/current/tool-provider/builtin/{provider}/tools
Authorization: Bearer {access_token}

# Get custom tool list
GET /console/api/workspaces/current/tool-provider/api/tools?provider={provider}
Authorization: Bearer {access_token}

# Get workflow tool list
GET /console/api/workspaces/current/tool-provider/workflow/tools?workflow_tool_id={app_id}
Authorization: Bearer {access_token}
```

### 7.2 Tool Credential Management
```bash
# Get built-in tool credential schema
GET /console/api/workspaces/current/tool-provider/builtin/{provider}/credentials_schema
Authorization: Bearer {access_token}

# Update built-in tool credentials
POST /console/api/workspaces/current/tool-provider/builtin/{provider}/update
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "credentials": {
        "api_key": "your_api_key"
    }
}

# Delete built-in tool credentials
POST /console/api/workspaces/current/tool-provider/builtin/{provider}/delete
Authorization: Bearer {access_token}
```

### 7.3 Custom Tool Operations
```bash
# Create custom tool
POST /console/api/workspaces/current/tool-provider/api/add
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "provider": "custom_provider",
    "openapi_schema": "openapi_schema_content",
    "credentials": {}
}

# Update custom tool
POST /console/api/workspaces/current/tool-provider/api/update
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "provider": "custom_provider",
    "openapi_schema": "updated_schema"
}

# Delete custom tool
POST /console/api/workspaces/current/tool-provider/api/delete
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "provider": "custom_provider"
}
```

## 8. Plugin Management

### 8.1 Basic Plugin Operations
```bash
# Upload plugin file
POST /console/api/workspaces/current/plugin/upload/pkg
Authorization: Bearer {access_token}
Content-Type: multipart/form-data

# Upload plugin from GitHub
POST /console/api/workspaces/current/plugin/upload/github
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "repo": "github_repo_url",
    "version": "v1.0.0",
    "package": "package_name"
}

# Get plugin manifest
GET /console/api/workspaces/current/plugin/fetch-manifest?plugin_unique_identifier={identifier}
Authorization: Bearer {access_token}

# Uninstall plugin
POST /console/api/workspaces/current/plugin/uninstall
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "plugin_installation_id": "plugin_id"
}
```

### 8.2 Marketplace Plugin Installation

```bash
# Get marketplace collections
GET {MARKETPLACE_API_PREFIX}/collections?page=1&page_size=100
# No authorization required (public marketplace API)
# Returns collections of plugins available in marketplace

# Get plugins by collection
GET {MARKETPLACE_API_PREFIX}/collections/{collection_name}/plugins?page=1&page_size=100
# Returns plugins within a specific collection

# Get plugin information from marketplace
GET {MARKETPLACE_API_PREFIX}/plugins/{org}/{name}
# Returns detailed plugin information

# Get plugin manifest from marketplace
GET {MARKETPLACE_API_PREFIX}/plugins/identifier?unique_identifier={unique_identifier}
# Returns plugin manifest and version information

# Search plugins in marketplace
POST {MARKETPLACE_API_PREFIX}/plugins/search/advanced
Content-Type: application/json

{
    "query": "search_term",
    "page": 1,
    "page_size": 20,
    "sort_by": "created_at",
    "sort_order": "desc",
    "category": "category_name",
    "tags": ["tag1", "tag2"]
}

# Install plugin from marketplace
POST /console/api/workspaces/current/plugin/install/marketplace
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "plugin_unique_identifiers": ["plugin_unique_identifier"]
}

# Upgrade plugin from marketplace
POST /console/api/workspaces/current/plugin/upgrade/marketplace
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "plugin_unique_identifier": "plugin_unique_identifier",
    "version": "new_version"
}

# Get plugin declaration from marketplace (for installed plugins)
GET /console/api/workspaces/current/plugin/marketplace/pkg?plugin_unique_identifier={unique_identifier}
Authorization: Bearer {access_token}
```

### 8.3 Complete Marketplace Plugin Installation Workflow

When users install plugins from the marketplace, the following API sequence is involved:

```bash
# Step 1: Browse marketplace collections
GET {MARKETPLACE_API_PREFIX}/collections?page=1&page_size=100
# User browses different plugin collections

# Step 2: View plugins in a collection or search plugins
GET {MARKETPLACE_API_PREFIX}/collections/{collection_name}/plugins
# or
POST {MARKETPLACE_API_PREFIX}/plugins/search/advanced
# User finds desired plugin

# Step 3: Get detailed plugin information
GET {MARKETPLACE_API_PREFIX}/plugins/{org}/{name}
# Get plugin details including versions, dependencies, etc.

# Step 4: Install plugin
POST /console/api/workspaces/current/plugin/install/marketplace
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "plugin_unique_identifiers": ["selected_plugin_unique_identifier"]
}
# Install plugin to workspace

# Step 5: Monitor installation status
GET /console/api/workspaces/current/plugin/tasks?page=1&page_size=255
Authorization: Bearer {access_token}
# Check installation progress

# Step 6: Verify installation
GET /console/api/workspaces/current/plugin/fetch-manifest?plugin_unique_identifier={identifier}
Authorization: Bearer {access_token}
# Verify plugin is properly installed
```

**Marketplace Installation Workflow Description:**
1. Users browse marketplace collections or search for specific plugins
2. View plugin details including description, version, and dependencies
3. Click install button to add plugin to workspace
4. System downloads and installs plugin from marketplace
5. Monitor installation progress through task management
6. Plugin becomes available in workspace once installation completes

### 8.4 Plugin Task Management
```bash
# Get plugin tasks
GET /console/api/workspaces/current/plugin/tasks?page=1&page_size=255
Authorization: Bearer {access_token}

# Check task status
GET /console/api/workspaces/current/plugin/tasks/{task_id}
Authorization: Bearer {access_token}

# Update plugin permissions
POST /console/api/workspaces/current/plugin/permission/change
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "permissions": {
        "tool": {
            "enabled": true
        }
    }
}
```

## 9. Message and Conversation Management

### 9.1 Chat Messages
```bash
# Send chat message (streaming)
POST /console/api/apps/{app_id}/chat-messages
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "query": "Hello",
    "conversation_id": "",
    "inputs": {},
    "response_mode": "streaming",
    "user": "user_id"
}

# Stop message response
POST /console/api/apps/{app_id}/chat-messages/{task_id}/stop
Authorization: Bearer {access_token}

# Get suggested questions
GET /console/api/apps/{app_id}/chat-messages/{message_id}/suggested-questions
Authorization: Bearer {access_token}
```

### 9.2 Completion Messages
```bash
# Send completion message
POST /console/api/apps/{app_id}/completion-messages
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "inputs": {
        "query": "Complete this text"
    },
    "response_mode": "streaming",
    "user": "user_id"
}
```

## 10. Annotation Management

### 10.1 Annotation Operations
```bash
# Get annotation configuration
GET /console/api/apps/{app_id}/annotation-setting
Authorization: Bearer {access_token}

# Update annotation status
POST /console/api/apps/{app_id}/annotation-reply/{action}
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "score_threshold": 0.9,
    "embedding_model": {
        "embedding_provider_name": "openai",
        "embedding_model_name": "text-embedding-ada-002"
    }
}

# Get annotation list
GET /console/api/apps/{app_id}/annotations?page=1&limit=20
Authorization: Bearer {access_token}

# Add annotation
POST /console/api/apps/{app_id}/annotations
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "question": "What is AI?",
    "answer": "AI stands for Artificial Intelligence"
}

# Delete annotation
DELETE /console/api/apps/{app_id}/annotations/{annotation_id}
Authorization: Bearer {access_token}
```

## 11. File Management

### 11.1 File Upload
```bash
# Upload file
POST /console/api/files/upload
Authorization: Bearer {access_token}
Content-Type: multipart/form-data

# Get file preview
GET /console/api/files/{file_id}/preview
Authorization: Bearer {access_token}

# Remote file upload
POST /console/api/remote-files/upload
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "url": "https://example.com/file.pdf"
}
```

## 12. Application Statistics and Monitoring

### 12.1 Application Statistics
```bash
# Get application daily message statistics
GET /console/api/apps/{app_id}/statistics/daily-messages?start={start_date}&end={end_date}
Authorization: Bearer {access_token}

# Get application daily conversation statistics
GET /console/api/apps/{app_id}/statistics/daily-conversations?start={start_date}&end={end_date}
Authorization: Bearer {access_token}

# Get application daily user statistics
GET /console/api/apps/{app_id}/statistics/daily-end-users?start={start_date}&end={end_date}
Authorization: Bearer {access_token}

# Get application token consumption statistics
GET /console/api/apps/{app_id}/statistics/token-costs?start={start_date}&end={end_date}
Authorization: Bearer {access_token}
```

### 12.2 Tracing Configuration
```bash
# Get tracing status
GET /console/api/apps/{app_id}/trace
Authorization: Bearer {access_token}

# Update tracing status
POST /console/api/apps/{app_id}/trace
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "enabled": true
}

# Add tracing configuration
POST /console/api/apps/{app_id}/trace-config
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "tracing_provider": "langfuse",
    "tracing_config": {
        "api_key": "your_api_key"
    }
}
```

## 13. API Key Management

### 13.1 API Key Operations
```bash
# Get API key list
GET /console/api/apps/{app_id}/api-keys?page=1&limit=20
Authorization: Bearer {access_token}

# Create API key
POST /console/api/apps/{app_id}/api-keys
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "name": "My API Key"
}

# Delete API key
DELETE /console/api/apps/{app_id}/api-keys/{api_key_id}
Authorization: Bearer {access_token}
```

## 14. App Market and Exploration

### 14.1 App Market Operations
```bash
# Get app market application list
GET /console/api/explore/apps
Authorization: Bearer {access_token}
# Return data structure:
# {
#   "categories": ["Writing", "Translate", "HR", "Programming", "Assistant", "Agent", "Recommended", "Workflow"],
#   "recommended_apps": [
#     {
#       "app": {
#         "id": "app_id",
#         "name": "App Name",
#         "description": "App Description",
#         "mode": "chat|completion|workflow",
#         "icon_type": "emoji|image",
#         "icon": "🤖",
#         "icon_background": "#D5F5F6"
#       },
#       "app_id": "app_id",
#       "category": "Assistant",
#       "install_count": 100,
#       "installed": false,
#       "editable": false,
#       "is_agent": false
#     }
#   ]
# }

# Get explore application details
GET /console/api/explore/apps/{app_id}
Authorization: Bearer {access_token}

# Get installed application list
GET /console/api/installed-apps
Authorization: Bearer {access_token}

# Get specific installed application information
GET /console/api/installed-apps?app_id={app_id}
Authorization: Bearer {access_token}

# Install application
POST /console/api/installed-apps
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "app_id": "app_id_to_install"
}

# Uninstall application
DELETE /console/api/installed-apps/{installed_app_id}
Authorization: Bearer {access_token}

# Update application pin status
PATCH /console/api/installed-apps/{installed_app_id}
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "is_pinned": true
}

# Get application access mode (enterprise feature)
GET /console/api/enterprise/webapp/app/access-mode?appId={app_id}
Authorization: Bearer {access_token}
```

### 14.2 Complete Workflow for Installing Recommended Apps to Workspace

When users install recommended applications from the app market to their workspace, the following API call sequence is involved:

```bash
# Step 1: Get recommended application list
GET /console/api/explore/apps
Authorization: Bearer {access_token}
# Returns recommended application list for users to browse and select

# Step 2: Get detailed information and export data for specific application
GET /console/api/explore/apps/{app_id}
Authorization: Bearer {access_token}
# Returns application details, including export_data field (DSL configuration)

# Step 3: Create application copy through DSL import
POST /console/api/apps/imports
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "mode": "yaml_content",
    "yaml_content": "export_data obtained from step 2",
    "name": "My Custom App Name",
    "icon_type": "emoji",
    "icon": "🤖", 
    "icon_background": "#D5F5F6",
    "description": "My custom description"
}
# Returns import status, which may be pending/completed/failed

# Step 4: If import status is pending, need to confirm import
POST /console/api/apps/imports/{import_id}/confirm
Authorization: Bearer {access_token}
# Complete application import to workspace

# Step 5: Update local installed application list (optional)
GET /console/api/installed-apps
Authorization: Bearer {access_token}
# Get latest installed application list
```

**Operation Workflow Description:**
1. Users browse recommended applications in the app market
2. Click "Add to Workspace" button for a specific application
3. System retrieves complete configuration data (DSL) for the application
4. Create application copy in user workspace through DSL import API
5. Users can customize application name, icon, and description
6. After confirming import, the application is added to user's workspace

## 15. System Features

### 15.1 System Information
```bash
# Get system features
GET /console/api/system-features
Authorization: Bearer {access_token}

# Get supported retrieval methods
GET /console/api/datasets/retrieval-setting
Authorization: Bearer {access_token}

# Get file upload configuration
GET /console/api/files/upload
Authorization: Bearer {access_token}
```

## Summary

The above lists the main backend API operations that the Dify frontend can call, covering:

1. **Authentication Management**: Login, registration, password reset
2. **Workspace Management**: Workspace and member management
3. **Application Management**: Application CRUD, configuration, import/export
4. **Workflow Management**: Workflow editing, execution, history
5. **Dataset Management**: Dataset and document management, hit testing
6. **Model Management**: Model provider and model configuration
7. **Tool Management**: Built-in tools, custom tools, workflow tools
8. **Plugin Management**: Plugin installation, uninstallation, configuration
9. **Message Management**: Chat and completion messages
10. **Annotation Management**: Annotation CRUD operations
11. **File Management**: File upload and preview
12. **Statistics and Monitoring**: Application usage statistics and tracing
13. **API Key Management**: API key creation and management
14. **App Market and Exploration**: App market browsing, installation, management
15. **System Features**: System configuration and feature queries

All these APIs require valid access_token for authenticated access.
