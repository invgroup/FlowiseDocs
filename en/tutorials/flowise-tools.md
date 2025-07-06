# Flowise Agentflow v2: Developer Guide – Creating a New Tool

This guide helps developers implement and integrate new tools into Flowise Agentflow v2. Tools are connectors to external services that follow a common interface and can be used dynamically within flows using the `Tool_Agentflow` orchestrator.

------

## 1. Understanding the `Tool_Agentflow` Node

The `Tool_Agentflow` node is a generic orchestrator with the following responsibilities:

- **Tool discovery** via `loadMethods.listTools` (filters by category/type)
- **Schema inspection** via `loadMethods.listToolInputArgs` (converts zod to JSON Schema)
- **Runtime execution** via the `run()` method

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

------

## 2. Creating a Tool Node

A Tool Node must implement the `INode` interface and be registered under the `Tools` or `Tools (MCP)` category.

### Example

```typescript
class MyCustomTool implements INode {
  label       = 'My Custom';
  name        = 'myCustomTool';
  version     = 1.0;
  type        = 'MyCustom';
  category    = 'Tools'; // Must be 'Tools' or 'Tools (MCP)'
  description = 'Describe your tool functionality';
  baseClasses = [this.type, 'Tool'];
  icon        = 'mytool.svg';

  // Optional credentials
  credential: INodeParams = {
    label: 'API Key',
    name: 'credential',
    type: 'credential',
    credentialNames: ['myCustomApi']
  };

  // User-facing inputs
  inputs: INodeParams[] = [
    { label: 'API Endpoint', name: 'apiEndpoint', type: 'string', placeholder: 'https://...' },
    { label: 'Option A', name: 'optA', type: 'options', options: [...] }
  ];

  // Tool creation
  async init(nodeData, _, options) {
    const credData = await getCredentialData(nodeData.credential, options);
    const apiKey = getCredentialParam('apiKey', credData, nodeData);
    const endpoint = nodeData.inputs?.apiEndpoint;

    const actions = convertMultiOptionsToStringArray(nodeData.inputs?.actions);
    return createMyCustomTools({ apiKey, endpoint, actions });
  }

  // Optional: transform inputs for downstream tools
  transformNodeInputsToToolArgs(nodeData) {
    return {
      endpoint: nodeData.inputs?.apiEndpoint,
      optA:     nodeData.inputs?.optA
    };
  }
}

module.exports = { nodeClass: MyCustomTool };
```

### Notes

- Use `optional: true` and `additionalParams: true` for non-required fields
- Use `show` conditions to dynamically display inputs based on other values

------

## 3. Defining Tool Logic Using `DynamicStructuredTool`

Flowise tools typically subclass `DynamicStructuredTool`, which allows structured input schemas using `zod` and includes built-in HTTP request handling.

### Steps

1. **Define input schemas** using `z.object(...)` with `.describe()` metadata
2. **Create a base tool class** that manages common behavior (e.g. auth, headers)
3. **Create one subclass per operation**, implementing `_call()`
4. **Use a factory function** to return tool instances based on selected actions

### Example

```typescript
// schemas.ts
const FooSchema = z.object({
  fooId: z.string().describe('The ID of the Foo to fetch')
});

// core.ts
class BaseFooTool extends DynamicStructuredTool {
  // shared auth and request logic
}

class GetFooTool extends BaseFooTool {
  constructor(args) {
    super({ name: 'get_foo', description: 'Retrieve a Foo', schema: FooSchema });
  }

  async _call(args) {
    return makeRequest({ method: 'GET', url: `/foo/${args.fooId}` });
  }
}

export function createFooTools({ actions, ...rest }) {
  const tools = [];
  if (actions.includes('getFoo')) {
    tools.push(new GetFooTool(rest));
  }
  return tools;
}
```

------

## 4. Register and Test the Tool

1. **Register your node file** by placing it in `components/AgentFlow/Tools/`
2. **Restart Flowise** so the tool can be discovered
3. **Use the tool in the UI**:
   - Add a Tool node (`Tool_Agentflow`)
   - Select your new tool from the dropdown
   - Fill in tool input arguments (generated from the zod schema)
   - Connect input/output as needed
4. **Run your flow** and verify correct output and tool behavior

------

## 5. Best Practices

| Area           | Recommendation                                     |
| -------------- | -------------------------------------------------- |
| Input Schemas  | Use `zod.describe()` to add helpful labels         |
| Code Structure | Reuse logic in a shared base class                 |
| Input Mapping  | Keep `transformNodeInputsToToolArgs()` minimal     |
| Output         | Return clean, structured results                   |
| Errors         | Handle errors clearly with actionable messages     |
| Streaming      | Only stream results if your tool is the final node |



------

## Summary

To build a new tool:

1. Implement a class using the `INode` interface and define inputs
2. Use `DynamicStructuredTool` to handle schemas and HTTP actions
3. Add schema definitions and action handlers
4. Register your tool, restart Flowise, and test it from the UI

This approach ensures your tool is reusable, dynamically configurable, and seamlessly integrated into any Flowise Agentflow.