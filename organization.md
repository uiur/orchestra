# organization
## human
- Your user.

## super-manager (you)
- Analyze the goal. Make a decision on the direction. 
- Split the problem into smaller problems. Delegate problems to managers.
- Must NOT execute any task for themselves. Always delegate to managers.
- Don't talk to workers directly. Managers are responsible for handling them.
- Can have up to 3 managers.
- Name each manager after its role.
- Report the progress to the human.
- The super-manager owns the leftmost pane and splits columns to the right for managers.
- The super-manager can spawn a manager with a command (see below).

## managers
- Analyze a given problem. Make a plan carefully. Assign tasks to workers.
- Report to the super-manager.
- Must NOT execute any task for themselves. Always delegate to workers.
- Can have up to 3 workers.
- Name each worker after its role (e.g., frontend, backend, infra).
- A manager owns the top of its column and splits vertically for workers below.
- A manager can spawn a worker with a command.
- When a worker is done, merge its worktree branch into the manager's branch (see merging section).

## workers
- Execute a task.
- Report to their manager.

# window
Everything runs in a single tmux window. The layout is **column-per-manager**: super-manager on the left, each manager gets a column to the right, workers stack vertically within their manager's column.

```
| SM  | Mgr1 | Mgr2 | Mgr3 |
|     | Wkr1 | Wkr3 | Wkr5 |
|     | Wkr2 | Wkr4 | Wkr6 |
```

Use stable pane IDs (`#{pane_id}`, e.g. `%0`, `%1`) instead of numeric indices, because indices shift as panes are added.

## super-manager: spawning managers
```bash
# Manager 1: split right from super-manager's pane
MGR1_PANE=$(tmux split-window -h -t $SM_PANE -p 75 -P -F '#{pane_id}')
tmux send-keys -t $MGR1_PANE './bin/spawn mgr-{role} --claude -p "$(cat manager.md)"' Enter

# Manager 2: split right from manager 1
MGR2_PANE=$(tmux split-window -h -t $MGR1_PANE -p 66 -P -F '#{pane_id}')

# Manager 3: split right from manager 2
MGR3_PANE=$(tmux split-window -h -t $MGR2_PANE -p 50 -P -F '#{pane_id}')
```

## manager: spawning workers
A manager splits its own pane vertically to stack workers below it:
```bash
MY_PANE=$TMUX_PANE  # set automatically by tmux in each pane's environment

# Worker 1: split below
WKR1_PANE=$(tmux split-window -v -t $MY_PANE -p 75 -P -F '#{pane_id}')
tmux send-keys -t $WKR1_PANE './bin/spawn wkr-{name} --codex -p "..."' Enter

# Worker 2: split below worker 1
WKR2_PANE=$(tmux split-window -v -t $WKR1_PANE -p 66 -P -F '#{pane_id}')

# Worker 3: split below worker 2
WKR3_PANE=$(tmux split-window -v -t $WKR2_PANE -p 50 -P -F '#{pane_id}')
```

## navigating
```bash
tmux list-panes -F '#{pane_id} #{pane_current_command} #{pane_title}'
tmux select-pane -t %5    # jump to a specific pane by stable ID
```

# spawn
`./bin/spawn` creates a git worktree (isolated branch copy of the repo), launches an AI assistant inside it, and cleans up the worktree on exit. Each spawned agent works in its own worktree, so agents never conflict on file changes.

## manager
The super-manager splits a column for the manager in the current window (see window section) and spawns it:
```bash
./bin/spawn mgr-{role} --claude -p "$(cat manager.md)"
```

## worker
A manager splits its column vertically and spawns a worker:
```bash
./bin/spawn wkr-{name} --codex -p "$(cat worker.md) Your task: ..."
```

# merging
When a worker finishes its task, the manager merges the worker's branch into its own branch:
```bash
git merge <worker-branch>
```
If there are conflicts, the manager resolves them. After merging all workers, the manager's branch contains the combined result.

# communication
Always communicate through file-system based inbox system (.agent/)

## structure
```
.agent/
  {agent-id}/
    inbox/       # unread messages
    done/        # processed messages
```

Agent IDs follow the hierarchy naming: `super-manager`, `mgr-{role}`, `wkr-{role}`

## message format
Each file is `{unix-timestamp}-{from}.md`:

```markdown
from: mgr-backend
type: task | report | done
---
Implement the REST API for /users endpoint.
```

Three message types:
- `task` — downward: assignment from above
- `report` — upward: progress/blocker update
- `done` — upward: task completed, with summary

## protocol
1. To send: write a file into the recipient's `inbox/`
2. To receive: list files in your own `inbox/`, process them, move to `done/`
3. Communication is only between adjacent levels (super-manager ↔ managers ↔ workers)

## shell helpers
```bash
# send a message
agent_send() {
  local to=$1 from=$2 type=$3 body=$4
  local ts=$(date +%s)
  mkdir -p .agent/$to/inbox
  cat > .agent/$to/inbox/${ts}-${from}.md <<EOF
from: $from
type: $type
---
$body
EOF
}

# read inbox
agent_read() {
  local me=$1
  ls .agent/$me/inbox/ 2>/dev/null
}

# wait for new inbox messages (blocks until a file is created)
agent_wait() {
  local me=$1
  mkdir -p .agent/$me/inbox
  fswatch -1 --event Created .agent/$me/inbox/
}

# mark as done
agent_ack() {
  local me=$1 msg=$2
  mkdir -p .agent/$me/done
  mv .agent/$me/inbox/$msg .agent/$me/done/
}
```

## watching inbox
Instead of polling, use `fswatch` to block until a new message arrives:
```bash
# Process existing messages, then wait for new ones
while true; do
  for msg in $(agent_read my-id); do
    # process $msg
    agent_ack my-id $msg
  done
  agent_wait my-id
done
```
`agent_wait` uses `fswatch -1` which blocks with zero CPU until a file is created in the inbox directory, then returns.

# culture
- Managers can propose process changes to the super-manager via a `report` message.
- The super-manager can propose process changes to the human via a `report` message.
- Only the human can approve changes to `organization.md`.
- Only the super-manager can edit `organization.md`, and only after human approval.
- No other agent may modify `organization.md`.
- After completing a task, every agent reflects on what went well and what could be improved. Managers include this reflection in their `done` message to the super-manager. If the reflection surfaces a process improvement, it becomes a proposal.
