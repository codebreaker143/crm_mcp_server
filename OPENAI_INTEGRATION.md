# OpenAI Agent Builder Integration Guide

## Overview

This CRM MCP server uses the **Model Context Protocol (MCP)** with **stdio transport**, which is designed for direct integration with MCP-compatible clients like Claude Desktop. OpenAI's Agent Builder currently does not natively support MCP servers.

## MCP Server Connection Details

When you run this MCP server, it will display detailed connection information including:

- **Protocol**: MCP over stdio (standard input/output)
- **Server Type**: Google Sheets, Calendly, or both
- **Credentials**: Google Service Account, Calendly API tokens
- **Configuration**: Full client configuration examples

### Starting the Server and Viewing Connection Details

```bash
# Build the project
npm run build

# Start the server (displays connection details)
npm start
```

The server will output comprehensive connection details including:
- Google Sheets configuration (credentials path, spreadsheet ID, service account email)
- Calendly configuration (API token masked for security, organization URI)
- MCP client configuration examples

## Integration Options for OpenAI

Since OpenAI Agent Builder requires HTTP REST APIs, here are your options:

### Option 1: Use Direct API Integration (Recommended for OpenAI)

Instead of using the MCP server, integrate directly with Google Sheets and Calendly APIs in your OpenAI custom actions:

#### Google Sheets API for OpenAI Actions

1. **Get your credentials from the MCP server logs**:
   - Service Account Email
   - Spreadsheet ID
   - Project ID

2. **Create OpenAPI schema for Google Sheets**:

```yaml
openapi: 3.0.0
info:
  title: CRM Google Sheets API
  version: 1.0.0
servers:
  - url: https://sheets.googleapis.com/v4/spreadsheets
paths:
  /{spreadsheetId}/values/{range}:
    get:
      summary: Get spreadsheet values
      parameters:
        - name: spreadsheetId
          in: path
          required: true
          schema:
            type: string
        - name: range
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Success
```

3. **Configure OpenAI Custom Action**:
   - Use OAuth 2.0 with Google Service Account
   - Add scopes: `https://www.googleapis.com/auth/spreadsheets`

#### Calendly API for OpenAI Actions

1. **Get your Calendly credentials from the MCP server logs**:
   - API Token
   - Organization URI

2. **Create OpenAPI schema for Calendly**:

```yaml
openapi: 3.0.0
info:
  title: Calendly CRM API
  version: 1.0.0
servers:
  - url: https://api.calendly.com
paths:
  /event_types:
    get:
      summary: List event types
      parameters:
        - name: organization
          in: query
          required: true
          schema:
            type: string
      security:
        - BearerAuth: []
      responses:
        '200':
          description: Success
components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
```

3. **Configure OpenAI Custom Action**:
   - Authentication: Bearer Token
   - Token: Your Calendly API token from the MCP logs

### Option 2: Create an HTTP Wrapper for MCP Server

Create a REST API wrapper that translates HTTP requests to MCP protocol calls:

#### Implementation Steps:

1. **Create an HTTP server** (Express.js example):

```javascript
// http-wrapper.js
const express = require('express');
const { Client } = require('@modelcontextprotocol/sdk/client/index.js');
const { StdioClientTransport } = require('@modelcontextprotocol/sdk/client/stdio.js');
const { spawn } = require('child_process');

const app = express();
app.use(express.json());

// Start MCP server as child process
const mcpProcess = spawn('node', ['dist/index.js'], {
  env: process.env
});

const transport = new StdioClientTransport({
  command: 'node',
  args: ['dist/index.js'],
  env: process.env
});

const client = new Client({
  name: 'http-wrapper',
  version: '1.0.0'
}, {
  capabilities: {}
});

// REST API endpoints
app.post('/api/tools/:toolName', async (req, res) => {
  try {
    const result = await client.callTool({
      name: req.params.toolName,
      arguments: req.body
    });
    res.json(result);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.get('/api/tools', async (req, res) => {
  try {
    const tools = await client.listTools();
    res.json(tools);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Initialize and start
async function start() {
  await client.connect(transport);
  app.listen(3000, () => {
    console.log('HTTP wrapper running on http://localhost:3000');
  });
}

start();
```

2. **Create OpenAPI schema for your wrapper**:

```yaml
openapi: 3.0.0
info:
  title: CRM MCP HTTP Wrapper
  version: 1.0.0
servers:
  - url: http://localhost:3000
paths:
  /api/tools/add_customer_record:
    post:
      summary: Add customer record
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                name:
                  type: string
                email:
                  type: string
                issue:
                  type: string
                status:
                  type: string
                  enum: [open, in-progress, resolved, closed]
                priority:
                  type: string
                  enum: [low, medium, high, urgent]
      responses:
        '200':
          description: Success
```

3. **Deploy the wrapper** (options):
   - Local: `http://localhost:3000`
   - Cloud: Deploy to Heroku, Vercel, AWS Lambda, etc.

4. **Configure in OpenAI**:
   - Import the OpenAPI schema
   - Set the base URL to your deployed wrapper

### Option 3: Use MCP-to-HTTP Bridge Tools

Use existing tools that can bridge MCP to HTTP:

- **MCP Proxy Server**: Community projects that expose MCP servers over HTTP
- **API Gateway**: Use services like Kong or Tyk with custom plugins

## Credentials Reference

### From MCP Server Logs

When you run `npm start`, the server displays:

#### Google Sheets
- **Credentials Path**: Path to your service account JSON file
- **Spreadsheet ID**: Your Google Sheets spreadsheet ID
- **Spreadsheet URL**: Direct link to your spreadsheet
- **Service Account Email**: Email to share spreadsheet with
- **Project ID**: Google Cloud project ID

#### Calendly
- **API Token**: Your Calendly personal access token (partially masked)
- **Organization URI**: Your Calendly organization URI
- **API Base URL**: `https://api.calendly.com`

### Using These Credentials

1. **For Direct API Integration**: Use these credentials directly in OpenAI custom actions
2. **For HTTP Wrapper**: Pass as environment variables to your wrapper service
3. **For Testing**: Use with tools like Postman or curl

## Example: Complete OpenAI Custom Action Setup

### Step 1: Run MCP Server to Get Credentials

```bash
npm start
```

Copy the displayed credentials.

### Step 2: Create Custom Action in OpenAI

1. Go to OpenAI Platform → Actions
2. Click "Create new action"
3. Choose "Import from URL" or "Define from scratch"

### Step 3: Add Google Sheets Action

```yaml
openapi: 3.0.0
info:
  title: CRM Google Sheets
  version: 1.0.0
servers:
  - url: https://sheets.googleapis.com/v4
paths:
  /spreadsheets/{spreadsheetId}/values/{range}:append:
    post:
      summary: Add customer record
      operationId: addCustomerRecord
      parameters:
        - name: spreadsheetId
          in: path
          required: true
          schema:
            type: string
            default: YOUR_SPREADSHEET_ID_FROM_LOGS
        - name: range
          in: path
          required: true
          schema:
            type: string
            default: CustomerRecords!A:J
        - name: valueInputOption
          in: query
          required: true
          schema:
            type: string
            default: RAW
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                values:
                  type: array
                  items:
                    type: array
                    items:
                      type: string
      responses:
        '200':
          description: Success
      security:
        - OAuth2: []
components:
  securitySchemes:
    OAuth2:
      type: oauth2
      flows:
        authorizationCode:
          authorizationUrl: https://accounts.google.com/o/oauth2/auth
          tokenUrl: https://oauth2.googleapis.com/token
          scopes:
            https://www.googleapis.com/auth/spreadsheets: Full access to Google Sheets
```

### Step 4: Add Calendly Action

```yaml
openapi: 3.0.0
info:
  title: CRM Calendly
  version: 1.0.0
servers:
  - url: https://api.calendly.com
paths:
  /event_types:
    get:
      summary: List event types
      operationId: listEventTypes
      parameters:
        - name: organization
          in: query
          required: true
          schema:
            type: string
            default: YOUR_ORG_URI_FROM_LOGS
      responses:
        '200':
          description: Success
      security:
        - BearerAuth: []
components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: token
```

### Step 5: Configure Authentication

1. **Google Sheets**: Set up OAuth 2.0 with your service account
2. **Calendly**: Add Bearer token from MCP logs

## Testing Your Integration

### Test Google Sheets API

```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
  "https://sheets.googleapis.com/v4/spreadsheets/YOUR_SPREADSHEET_ID/values/CustomerRecords!A1:J1"
```

### Test Calendly API

```bash
curl -H "Authorization: Bearer YOUR_CALENDLY_TOKEN" \
  "https://api.calendly.com/event_types?organization=YOUR_ORG_URI"
```

## Security Considerations

1. **Never commit credentials** to version control
2. **Use environment variables** for all sensitive data
3. **Rotate API tokens** regularly
4. **Use HTTPS** for all HTTP wrapper deployments
5. **Implement rate limiting** in HTTP wrappers
6. **Validate inputs** to prevent injection attacks

## Troubleshooting

### MCP Server Not Showing Credentials

- Ensure `.env` file is configured correctly
- Check that credentials files exist at specified paths
- Run with `npm start` not `node dist/index.js` directly

### OpenAI Can't Connect to APIs

- Verify API tokens are current and valid
- Check firewall/network settings if using HTTP wrapper
- Ensure OAuth scopes are correctly configured
- Test APIs directly with curl first

### HTTP Wrapper Issues

- Ensure MCP server can start successfully
- Check that ports are not blocked
- Verify environment variables are passed correctly
- Monitor logs for connection errors

## Additional Resources

- [MCP Specification](https://modelcontextprotocol.io/)
- [Google Sheets API Documentation](https://developers.google.com/sheets/api)
- [Calendly API Documentation](https://developer.calendly.com/)
- [OpenAI Custom Actions Guide](https://platform.openai.com/docs/actions)

## Support

For issues specific to this CRM MCP server, please open an issue on GitHub.
For OpenAI integration questions, consult OpenAI's documentation.
