run_turn(user_input):
  append user_input to session history
  
  loop:
    iterations++
    if iterations > max_iterations → error
    
    → call Claude API with (system_prompt + full session history)
    ← get back events: [TextDelta, ToolUse, Usage, MessageStop]
    
    build assistant_message from events
    append assistant_message to session
    
    if no tool_use blocks → BREAK (Claude is done)
    
    for each tool_use (id, name, input):
      1. run PreToolUse hook  → may modify input, cancel, or deny
      2. check permission_policy → Allow or Deny
      
      if Allow:
        run tool_executor.execute(name, input)
        run PostToolUse hook → may append feedback to output
        build tool_result message
      
      if Deny:
        build tool_result with error = true, reason = denial message
      
      append tool_result to session
  
  maybe_auto_compact()  ← trim history if tokens > threshold
  return TurnSummary
