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

Tag every pane with a custom `@agent-id` option for reliable identification. Pane titles are unreliable (the shell's prompt overwrites them), but `@agent-id` survives pane kills, respawns, and shell activity.

## pane helpers
```bash
# Create a pane and tag it with @agent-id
spawn_pane() {
  local direction=$1 target=$2 percent=$3 agent_id=$4
  local new_pane
  new_pane=$(tmux split-window $direction -t "$target" -p "$percent" -P -F '#{pane_id}')
  [ -z "$new_pane" ] && return 1
  tmux set-option -p -t "$new_pane" @agent-id "$agent_id"
  tmux set-option -p -t "$new_pane" remain-on-exit on
  echo "$new_pane"
}

# Find a pane by its @agent-id
find_pane() {
  tmux list-panes -F '#{pane_id} #{@agent-id}' | awk -v id="$1" '$2==id {print $1}'
}
```

## split percentages
To get N equal-sized parts, each successive split uses:
```
Split 1: -p 75   (4 parts) or -p 67 (3 parts) or -p 50 (2 parts)
Split 2: -p 67   (4 parts) or -p 50 (3 parts)
Split 3: -p 50   (4 parts)
```
Formula: split *i* of *N−1* → `-p $((100 * (N-i) / (N-i+1)))`

## super-manager: spawning managers
```bash
SM_PANE=$TMUX_PANE

# Manager 1: split right from super-manager's pane
MGR1_PANE=$(spawn_pane -h $SM_PANE 75 mgr-{role})
tmux send-keys -t $MGR1_PANE '$MULTI_AGENT_HOME/bin/spawn mgr-{role} --claude -f $MULTI_AGENT_HOME/prompt/manager.md' Enter

# Manager 2: split right from manager 1
MGR2_PANE=$(spawn_pane -h $MGR1_PANE 67 mgr-{role})

# Manager 3: split right from manager 2
MGR3_PANE=$(spawn_pane -h $MGR2_PANE 50 mgr-{role})
```

## manager: spawning workers
A manager splits its own pane vertically to stack workers below it:
```bash
MY_PANE=$TMUX_PANE  # set automatically by tmux in each pane's environment

# Worker 1: split below
WKR1_PANE=$(spawn_pane -v $MY_PANE 75 wkr-{name})
tmux send-keys -t $WKR1_PANE '$MULTI_AGENT_HOME/bin/spawn wkr-{name} --codex -p "..."' Enter

# Worker 2: split below worker 1
WKR2_PANE=$(spawn_pane -v $WKR1_PANE 67 wkr-{name})

# Worker 3: split below worker 2
WKR3_PANE=$(spawn_pane -v $WKR2_PANE 50 wkr-{name})
```

## navigating
```bash
# List all panes with their agent IDs
tmux list-panes -F '#{pane_id} #{@agent-id} #{pane_current_command}'

# Jump to a pane by stable ID
tmux select-pane -t %5

# Find a pane by agent name
find_pane wkr-api

# Check for crashed panes
tmux list-panes -F '#{pane_id} #{@agent-id} dead=#{pane_dead}' | grep 'dead=1'

# Respawn a crashed pane (preserves @agent-id)
tmux respawn-pane -t $(find_pane wkr-api)
```

# spawn
`$MULTI_AGENT_HOME/bin/spawn` creates a git worktree (isolated branch copy of the repo), launches an AI assistant inside it, and cleans up the worktree on exit. Each spawned agent works in its own worktree, so agents never conflict on file changes.

## manager
The super-manager splits a column for the manager in the current window (see window section) and spawns it:
```bash
$MULTI_AGENT_HOME/bin/spawn mgr-{role} --claude -f $MULTI_AGENT_HOME/prompt/manager.md
```

## worker
A manager splits its column vertically and spawns a worker:
```bash
$MULTI_AGENT_HOME/bin/spawn wkr-{name} --codex -p "$(cat $MULTI_AGENT_HOME/prompt/worker.md) Your task: ..."
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
