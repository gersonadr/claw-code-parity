# Key Code Snippet: Assembling the API Request

`rust/crates/tools/src/lib.rs:3428-3437`

```rust
let message_request = MessageRequest {
    model: self.model.clone(),
    max_tokens: max_tokens_for_model(&self.model),
    messages: convert_messages(&request.messages),
    system: (!request.system_prompt.is_empty())
        .then(|| request.system_prompt.join("\n\n")),
    tools: (!tools.is_empty()).then_some(tools),
    tool_choice: (!self.allowed_tools.is_empty())
        .then_some(ToolChoice::Auto),
    stream: true,
};
```
