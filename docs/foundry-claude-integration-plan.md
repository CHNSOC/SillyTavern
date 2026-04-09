# Azure AI Foundry Claude Integration Plan

## Overview

Integrate Claude models hosted on Microsoft Azure AI Foundry into SillyTavern as a
sub-mode of the existing Claude provider. Foundry uses the **identical Anthropic Messages
API** but routes through Azure endpoints with Azure-style authentication, so the cleanest
approach is to inject Foundry-specific URL/auth into the existing `sendClaudeRequest()`
flow — similar to how `reverse_proxy` already works, but with dedicated UI fields and a
separate API key secret.

Reference: https://platform.claude.com/docs/en/build-with-claude/claude-in-microsoft-foundry

## Foundry API Details

- **Endpoint format**: `https://{resource}.services.ai.azure.com/anthropic/v1/messages`
- **Auth header**: `api-key: <AZURE_API_KEY>` (instead of `x-api-key`)
- **Required header**: `anthropic-version: 2023-06-01` (same as standard Claude)
- **Request/response format**: Identical to Anthropic Messages API
- **No `/models` endpoint** — model name equals the deployment name configured in Azure
- **No standard rate-limit headers** — use Azure monitoring instead

## Implementation Plan

### Step 1: Add secret key for Foundry

**File**: `src/endpoints/secrets.js`

Add to `SECRET_KEYS`:
```js
AZURE_CLAUDE: 'api_key_azure_claude',
```

### Step 2: Add Foundry toggle + settings to frontend defaults

**File**: `public/scripts/openai.js`

Add new settings to `default_settings` (~line 460):
```js
claude_use_foundry: false,
claude_foundry_resource_url: '',   // e.g. "https://my-resource.services.ai.azure.com"
```

Add these to `oai_setting_names` / `settingPaths` so they persist and load correctly.

### Step 3: Add UI fields in index.html

**File**: `public/index.html`

Inside the existing Claude settings section (search for `data-source="claude"`), add a
collapsible "Azure AI Foundry" section:

```html
<!-- Azure AI Foundry Configuration (within Claude source) -->
<div data-source="claude" class="flex-container flexFlowColumn">
    <label class="checkbox_label" for="claude_use_foundry">
        <input id="claude_use_foundry" type="checkbox" data-setting="claude_use_foundry" />
        <span>Use Azure AI Foundry</span>
    </label>
    <div id="claude_foundry_settings" class="flex-container flexFlowColumn" style="display: none;">
        <label for="claude_foundry_resource_url">Foundry Resource URL</label>
        <input id="claude_foundry_resource_url"
               data-setting="claude_foundry_resource_url"
               placeholder="https://your-resource.services.ai.azure.com" />
        <small>
            Enter just the base URL. The <code>/anthropic/v1/messages</code> path is
            appended automatically.
        </small>
        <label for="api_key_azure_claude">Foundry API Key</label>
        <input id="api_key_azure_claude" type="password"
               placeholder="Azure API key from Keys and Endpoint page" />
    </div>
</div>
```

Wire up the checkbox to show/hide the settings div.

### Step 4: Frontend — Pass Foundry config to backend

**File**: `public/scripts/openai.js`

In `createGenerationParameters()` (~line 2694, the Claude-specific block):
```js
if (settings.claude_use_foundry) {
    generate_data.claude_use_foundry = true;
    generate_data.claude_foundry_resource_url = settings.claude_foundry_resource_url;
}
```

In `onConnectButtonClick()` — when `claude_use_foundry` is true, save the Foundry API key
under `SECRET_KEYS.AZURE_CLAUDE` and pass `claude_use_foundry` + URL in the status
request body.

### Step 5: Backend — Modify `sendClaudeRequest()` to support Foundry

**File**: `src/endpoints/backends/chat-completions.js`

At the top of `sendClaudeRequest()` (~line 209), replace the static URL/key resolution:

```js
// Current code:
const apiUrl = new URL(request.body.reverse_proxy || API_CLAUDE).toString();
const apiKey = request.body.reverse_proxy
    ? request.body.proxy_password
    : readSecret(request.user.directories, SECRET_KEYS.CLAUDE);

// New code:
const isFoundry = Boolean(request.body.claude_use_foundry);
let apiUrl, apiKey, authHeader;

if (isFoundry && request.body.claude_foundry_resource_url) {
    // Azure AI Foundry: endpoint is {resource_url}/anthropic/v1
    apiUrl = new URL(
        request.body.claude_foundry_resource_url.replace(/\/+$/, '') + '/anthropic/v1'
    ).toString();
    apiKey = readSecret(request.user.directories, SECRET_KEYS.AZURE_CLAUDE);
    authHeader = { 'api-key': apiKey };        // Foundry uses 'api-key' header
} else if (request.body.reverse_proxy) {
    apiUrl = new URL(request.body.reverse_proxy).toString();
    apiKey = request.body.proxy_password;
    authHeader = { 'x-api-key': apiKey };
} else {
    apiUrl = new URL(API_CLAUDE).toString();
    apiKey = readSecret(request.user.directories, SECRET_KEYS.CLAUDE);
    authHeader = { 'x-api-key': apiKey };
}
```

Then in the `fetch()` call (~line 361), replace the hardcoded `'x-api-key': apiKey` with
`...authHeader`:

```js
const generateResponse = await fetch(apiUrl + '/messages', {
    method: 'POST',
    signal: controller.signal,
    body: JSON.stringify(requestBody),
    headers: {
        'Content-Type': 'application/json',
        'anthropic-version': '2023-06-01',
        ...authHeader,           // <-- was: 'x-api-key': apiKey
        ...additionalHeaders,
    },
});
```

### Step 6: Backend — Status check for Foundry mode

**File**: `src/endpoints/backends/chat-completions.js`

In the `/status` route, the Claude source currently falls through to the generic
`/models` fetch (which won't work for Foundry since there's no `/models` endpoint).

Add an early return when Foundry mode is detected:

```js
// Inside the status route, before the generic models fetch:
if (request.body.chat_completion_source === CHAT_COMPLETION_SOURCES.CLAUDE
    && request.body.claude_use_foundry) {
    const foundryUrl = request.body.claude_foundry_resource_url;
    const foundryKey = readSecret(request.user.directories, SECRET_KEYS.AZURE_CLAUDE);

    if (!foundryUrl || !foundryKey) {
        return statusResponse.status(400).send({
            error: true,
            message: 'Foundry configuration incomplete. Provide Resource URL and API Key.',
        });
    }

    // Foundry has no /models endpoint. Do a minimal probe to validate connectivity.
    try {
        const probeUrl = foundryUrl.replace(/\/+$/, '') + '/anthropic/v1/messages';
        const probeResponse = await fetch(probeUrl, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'api-key': foundryKey,
                'anthropic-version': '2023-06-01',
            },
            body: JSON.stringify({
                model: 'claude-sonnet-4-6',
                max_tokens: 1,
                messages: [{ role: 'user', content: 'Hi' }],
            }),
        });

        // Even a 400 (bad request) proves the endpoint + key are valid.
        // Only network errors or 401/403 indicate real config problems.
        if (probeResponse.status === 401 || probeResponse.status === 403) {
            return statusResponse.status(401).send({
                error: true,
                message: 'Invalid Foundry API key or insufficient permissions.',
            });
        }

        // Return empty model list — user picks model from the Claude model selector
        return statusResponse.send({ data: [] });
    } catch (error) {
        return statusResponse.status(500).send({
            error: true,
            message: 'Failed to connect to Foundry endpoint.',
        });
    }
}
```

### Step 7: Frontend — Handle Foundry in status request

**File**: `public/scripts/openai.js`

In `getStatusOpen()` or wherever the status request body is built for Claude, add:

```js
if (oai_settings.claude_use_foundry) {
    body.claude_use_foundry = true;
    body.claude_foundry_resource_url = oai_settings.claude_foundry_resource_url;
}
```

### Step 8: Handle `anthropic-beta` headers for Foundry

Some beta headers may not be supported on Foundry. Add a check:

```js
// In sendClaudeRequest(), before setting beta headers:
if (isFoundry) {
    // Foundry may not support all beta features; filter to known-supported ones
    // For now, keep the same headers — Foundry ignores unknown beta headers gracefully
}
```

This can be refined based on testing.

## Files Modified (Summary)

| File | Change |
|------|--------|
| `src/endpoints/secrets.js` | Add `AZURE_CLAUDE` secret key |
| `src/constants.js` | No changes needed (reuses `CLAUDE` source) |
| `src/endpoints/backends/chat-completions.js` | Modify `sendClaudeRequest()` URL/auth logic; add Foundry status check |
| `public/scripts/openai.js` | Add settings, pass Foundry config in requests, handle connect/status |
| `public/index.html` | Add Foundry checkbox + settings fields in Claude section |

## Testing Plan

1. **Without Foundry** — Verify standard Claude API still works unchanged (regression)
2. **Foundry connect** — Toggle checkbox, enter Resource URL + API key, click Connect
3. **Foundry generation** — Send a message, verify response streams correctly
4. **Foundry features** — Test thinking/extended thinking, tool use, system prompts
5. **Error handling** — Invalid URL, wrong API key, missing fields
6. **Settings persistence** — Reload page, verify Foundry toggle and URL are remembered

## Limitations & Future Work

- **No model listing**: Foundry has no `/models` endpoint, so the user must select the
  model from the existing Claude model dropdown. The deployment name in Azure must match
  the model ID (e.g., `claude-opus-4-6`).
- **Entra ID auth**: This plan only covers API key auth. Microsoft Entra ID (bearer token)
  auth could be added later if needed.
- **Beta feature compatibility**: Some Anthropic beta headers may behave differently on
  Foundry. Needs testing.
- **Prompt caching**: May or may not be available on Foundry — needs verification.
