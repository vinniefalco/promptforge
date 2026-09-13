# The full loop

This chapter assembles the complete agent: the built-in chat itself, one Markdown prompt embedded in the Workshop. A prompt saved as `chat.md` in the agents directory shadows it, so the chat you already use is a role your own agent can take. Walk through that program turn by turn, because the whole session surface shows up in it, working together.

## A chat agent

A chat agent is a transparent pass-through. It advertises no tools and sets no system prompt. It relays between the operator and the selected model, and nothing else. That restraint is the design: the program adds no behavior the operator did not ask for.

## One turn

The agent is one infinite loop in a single Lua block. Each turn does the same four things, in order.

1. Call `user_input()` to request the operator's next message, and return from the program when input is no longer available.
2. Append the operator's message to the retained history list.
3. Read the operator's selected model from the `ui()` snapshot's `selected_model` field.
4. Run `models.loop` over the history under that model, wrapped in `pcall`, then loop back to step 1.

## The full program

````markdown
---
name: chat
description: The built-in Workshop chat agent on the unified runtime.
promptforge: 0
---

# Chat

The built-in chat agent: a transparent pass-through between the operator
and the selected model. The message list is an explicit Lua value retained
across turns; the model is re-read from the host snapshot every turn, so a
menu selection change takes effect on the next turn.

## Conversation

```lua
local history = messages.new()
while true do
    local text, available = user_input()
    if not available then
        return
    end
    history:user(text)
    local selected = ui().selected_model
    if selected then
        pcall(function() return models.loop(models.get(selected), history) end)
    end
end
```
````

This is the whole chat surface. Work through the lines. `messages.new()` builds the empty conversation list once, before the loop starts. `user_input()` suspends the run until the operator answers, and returns the answer text together with an availability flag; when the flag reads false, the program returns instead of spinning on a dead session. `history:user(text)` appends the operator's message to the list. `ui().selected_model` reads the interface's current model selection, and `models.get(selected)` resolves that selection to a bound handle. `models.loop(handle, history)` runs the model round over the list, and the reply is appended to that same list as the final record, so the list the program passed in comes back one turn longer.

## Why the list is the state

Notice what the program never does: rebuild the conversation. The list is created once and retained across turns, and both sides accumulate in it - the program appends each operator message with `history:user(text)`, and `models.loop` appends each assistant reply as it completes. The next turn's model round therefore sees the whole conversation, and the program never copies, re-derives, or re-reads anything.

Because the model is re-read from the `ui()` snapshot on every turn, never captured once before the loop, a menu selection change takes effect on the very next turn.

## Why pcall wraps the model call

The loop runs `models.loop` under `pcall` because chat survives transport errors, and so must this program. A failed round does not kill the agent. The session surfaces the failure to the operator, and the loop returns to `user_input()` for the next turn.

## Grow from here

Start from this program and add one capability at a time. Bring a tool into scope with `tools.add` before the loop call and the model can ask for it. Save notes with `store.write`. Keep a counter in `var`. The loop does not change. The turns just do more.
