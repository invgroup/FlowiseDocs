# Flowise Agentflow v2: Tool Development  Guide

This guide aims to help developers implement and integrate new tools into Flowise  for use with Agentflow v2 flows. 


## Overview

Flowise tools act as connectors to external services, available to Agentflow v2 via the `Tool_Agentflow` orchestrator node. Each tool encapsulates one or more operations, exposes structured inputs (via Zod schemas), and returns normalised outputs.

Tools allow modular, reusable functionality in flows, such as:

- Fetching data from REST or GraphQL APIs
- Calling internal microservices
- Performing database queries
- Executing custom business logic

Existing tool implementations can be found in the Flowise repository: https://github.com/FlowiseAI/Flowise/tree/main/packages/components/nodes/tools


## Getting setup

- Node.js & pnpm installed
- Install Git and Clone the flowise repository see [README.md](https://github.com/FlowiseAI/Flowise/blob/main/README.md) guide.


## Understanding Custom Tools

All custom tool nodes implement the `INode` interface and register under the **Tools** category,  the following is a breakdown of what each of these property means:

| Property    | Description                                      |
| ----------- | ------------------------------------------------ |
| label       | The name of the tool node that appears on the UI |
| name        | The name that is used by code.                   |
| version     | Version of the node tool                         |
| type        | Tools                                            |
| icon        | Icon of the node tool                            |
| author      | Creator of the tool                              |
| description | Tool description                                 |
| baseClasses | The base classes from the node tool              |

The (`Tool_Agentflow`) [https://github.com/FlowiseAI/Flowise/blob/main/packages/components/nodes/agentflow/Tool/Tool.ts] handles discovery, schema extraction, and invoking tool instances at runtime.

```typescript
class Tool_Agentflow implements INode {
  loadMethods = {
    async listTools(...) { /* discover tools */ },
    async listToolInputArgs(...) { /* extract zod-based inputs */ },
    async listRuntimeStateKeys(...) { /* extract state from startAgentflow */ }
  };

  async run(nodeData, input, options) {
    // 1. Resolve selected tool
    // 2. Instantiate tool(s)
    // 3. Merge user inputs
    // 4. Invoke tool.call(...) or call() for each tool
    // 5. Handle special output markers
    // 6. Return normalized output
  }
}
```

## Implementing a Custom Tool

Each tool node should live in `components/agentflow/Tools/`. Filename convention: `<toolName>.ts`, exporting `{ nodeClass: YourToolClass }`.

### Class Definition

```
export class MyCustomTool implements INode {
  label       = 'My Custom';      // UI label
  name        = 'myCustomTool';   // code identifier (camelCase)
  version     = 1.0;              // tool version
  type        = 'Tool';
  category    = 'Tools';          // must be 'Tools'
  description = 'Short description';
  baseClasses = ['Tool', 'MyCustom'];
  icon        = 'my-custom.svg';  // icon file in assets
```

### Credentials & Inputs

Use `INodeParams` to declare credentials or user inputs.

```
  // Optional credential field
  credential: INodeParams = {
    label: 'API Key',
    name: 'credential',
    type: 'credential',
    credentialNames: ['myCustomApi']
  };

  // User inputs
  inputs: INodeParams[] = [
    {
      label: 'Endpoint',
      name: 'apiEndpoint',
      type: 'string',
      placeholder: 'https://api.example.com'
    },
    {
      label: 'Mode',
      name: 'mode',
      type: 'options',
      options: [
        { label: 'Read', value: 'read' },
        { label: 'Write', value: 'write' }
      ]
    }
  ];
```

- Mark non-required fields with `optional: true`
- Allow arbitrary extras with `additionalParams: true`
- Use `show` conditions to display inputs dynamically

###  Initialisation (`init`)

The `init()` hook returns one or multiple tool instances. Fetch credentials and configure your API client here.

```
  async init(nodeData, _, options) {
    const creds = await getCredentialData(nodeData.credential, options);
    const apiKey = getCredentialParam('apiKey', creds, nodeData);
    const endpoint = nodeData.inputs?.apiEndpoint;

    // Factory returns instances of DynamicStructuredTool
    return createMyCustomTools({ apiKey, endpoint });
  }
```

### Input Transformation

Override `transformNodeInputsToToolArgs()` to map node inputs to the structured tool arguments:

```
  transformNodeInputsToToolArgs(nodeData) {
    return {
      endpoint: nodeData.inputs?.apiEndpoint,
      mode:     nodeData.inputs?.mode
    };
  }
}

module.exports = { nodeClass: MyCustomTool };
```

## DynamicStructuredTool Pattern

For HTTP-based services and structured inputs, extend `DynamicStructuredTool`:

1. **Define Zod schema** in `schemas.ts` with `.describe()` for metadata.
2. **Create a base class** for shared logic (auth, headers).
3. **Subclass per operation**, implementing `_call(args)`.
4. **Use factory** to assemble tools for available actions.

```
// schemas.ts
export const GetItemSchema = z.object({
  itemId: z.string().describe('ID of item')
});

// core.ts
class BaseItemTool extends DynamicStructuredTool { /* common behavior */ }

export class GetItemTool extends BaseItemTool {
  constructor() {
    super({
      name: 'get_item',
      description: 'Fetch an item by ID',
      schema: GetItemSchema
    });
  }

  async _call(args) {
    return this.request({
      method: 'GET',
      url: `/items/${args.itemId}`
    });
  }
}

export function createItemTools({ actions, ...opts }) {
  const tools = [];
  if (actions.includes('get')) tools.push(new GetItemTool(opts));
  return tools;
}
```



## Registering & Discovering Tools

1. **Register your node file** by placing it in `components/AgentFlow/Tools/`
2. **Restart Flowise** so the tool can be discovered
3. **Use the tool in the UI**:
   - Add a Tool node (`Tool_Agentflow`)
   - Select your new tool from the dropdown
   - Fill in tool input arguments (generated from the zod schema)
   - Connect input/output as needed
4. **Run your flow** and verify correct output and tool behavior



## Best Practices

| Area           | Recommendation                                 |
| -------------- | ---------------------------------------------- |
| Input Schemas  | Use `zod.describe()` to add helpful labels     |
| Code Structure | Reuse logic in a shared base class             |
| Input Mapping  | Keep `transformNodeInputsToToolArgs()` minimal |
| Output         | Return clean, structured results               |
| Errors         | Handle errors clearly with actionable message  |

