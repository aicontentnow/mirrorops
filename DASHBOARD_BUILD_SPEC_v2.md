# MIRROR OPS DASHBOARD -- Build Spec v2
## Authored by: master-strategist
## Date: 2026-04-10
## Priority: High -- this is the primary interface between Bodhi and the agent fleet
## Build in: Cowork session

---

## THE VISION

The ops dashboard is the command center. Everything the agents produce, flag, or need surfaces here. Bodhi comes here first. Cowork sessions are for major lifts only. The dashboard handles routine approval, feedback, and monitoring.

The feedback loop must be airtight:
Bodhi sees flag -> clicks it -> leaves feedback inline -> it writes to agent_triggers -> poller picks up within 10 minutes -> agent redrafts -> surfaces back in dashboard.

No more going to Supabase to understand what's happening. No more mystery about which agents are running.

---

## FEATURE 1 -- CLICKABLE AGENT NOTIFICATIONS IN BELL DROPDOWN

### Current state
agent_notifications are fetched from Supabase but NOT rendered in the bell dropdown. The dropdown only shows transmission-status events (review, image ready, etc.). When bell is opened, all agent_notifications are immediately marked acknowledged and disappear.

### What it needs to do

**Bell badge:** shows count of unread items from BOTH sources:
- Transmission-status events (existing)
- agent_notifications with severity='action_needed' or severity='warning' (NEW)

**Bell dropdown:** two sections, visually separated:

Section 1 -- NEEDS YOU (action_needed severity)
- Each item shows: severity dot (magenta=action_needed, amber=warning), agent name, title, timestamp
- If notification body contains a transmission_id in payload: clicking opens that transmission card with feedback field pre-focused
- If notification is a general action item: clicking expands the body inline in the dropdown
- Each item has a dismiss button (marks acknowledged=true in agent_notifications)
- Each item has a "Send to Agent" button if there's a relevant agent_name -- writes feedback to agent_triggers

Section 2 -- INFO (info severity, collapsible)
- Shows agent activity confirmations (image batch complete, publish run, etc.)
- Collapsed by default, expand to see

**Acknowledge behavior:** Items are NOT auto-acknowledged on bell open. They persist until Bodhi explicitly dismisses them.

### Inline feedback flow
When Bodhi clicks an action_needed notification tied to a transmission:
1. Transmission card opens (or a modal appears) with the full body, cultural_hook, image preview
2. A feedback text field appears at the bottom
3. Submit button writes to agent_triggers:
   ```sql
   INSERT INTO agent_triggers (agent_name, reason, payload, source)
   VALUES ('content-engine', 'feedback_ready', 
     jsonb_build_object('transmission_id', :id, 'feedback', :feedback_text),
     'dashboard_notification');
   ```
4. Notification is marked acknowledged
5. A "queued for agent" confirmation appears

---

## FEATURE 2 -- AGENT JOURNEY MAP

### Placement
Immediately below the stats bar row, above the tabs. Full width. Always visible regardless of which tab is active.

### Design
A horizontal strip of agent nodes. Each node is a rounded card. Bodhi mentioned he'd share a design screenshot -- use that as reference. Default spec below.

### Agent nodes to show (with new names per renaming spec)

| Agent ID | Display Name | Role Label |
|---|---|---|
| mirror-track-a-agent | Track A | TikTok Gap |
| mirror-track-b-agent | Track B | World News |
| mirror-approval-watcher | Image Gen | Approval Watcher |
| mirror-website-publisher | Publisher | Blog Publisher |
| mirror-chief-of-staff | Chief of Staff | Ops Coordinator |
| mirror-core-term-agent | Core Term | Quality Gate |
| mirror-trigger-poller | Poller | Queue Runner |

### Node states and colors
- RUNNING (cyan) -- agent_triggers has a row with status='running' for this agent
- DONE / OK (green) -- last activity in agent_notifications < 2 hours ago, no failures
- WAITING (gray) -- no activity in last 6 hours, no pending triggers
- PENDING (amber) -- agent_triggers has pending row for this agent
- BLOCKED (magenta) -- last notification severity='critical' or 'warning' in last 6h, OR agent_triggers row status='failed'
- NEEDS ACTION (magenta, pulsing) -- agent_notifications has unacknowledged action_needed from this agent

### Node content
Each node shows:
- Agent display name (large)
- Role label (small, muted)
- Status badge with state label
- Last run: "2h ago" / "just now" / "14h ago"
- If BLOCKED or NEEDS ACTION: red dot + short reason text from notification title

### Data source
Query on load + real-time subscription:
```sql
-- Last activity per agent
SELECT agent_name, MAX(created_at) as last_active, 
  COUNT(*) FILTER (WHERE severity='action_needed' AND acknowledged=false) as pending_actions,
  COUNT(*) FILTER (WHERE severity='critical' AND acknowledged=false) as critical_count
FROM agent_notifications
WHERE created_at > now() - interval '24 hours'
GROUP BY agent_name;

-- Pending triggers
SELECT agent_name, status, COUNT(*) as count
FROM agent_triggers
WHERE status IN ('pending', 'running', 'failed')
GROUP BY agent_name, status;
```

### Interaction
Clicking a node opens a side drawer showing:
- Last 5 notifications from that agent (title + body + timestamp)
- Last 3 agent_triggers rows (reason + status + result_summary)
- A "Kick Agent" button that writes a manual_kick trigger row

---

## FEATURE 3 -- CHIEF OF STAFF MORNING REPORT PANEL

### Current state
MorningReport component exists but only shows the Opus Queue. Chief of staff report is buried.

### What it needs to show
A pinned panel at the top of the Overview tab, always expanded by default, collapsible:

- Header: "CHIEF OF STAFF // [date] // [time]"
- Pipeline state: color-coded counts (draft / review / approved / published / expired)
- Agent health row: mini status indicators for each agent (green/amber/red dots)
- Attention items: each item from the morning report's attention_items array, rendered as a clickable card with severity badge. Clicking an attention item that references a transmission opens that transmission card.
- Opus Queue: transmissions ready for manual Opus workflow, with one-click copy of narration script

---

## FEATURE 4 -- FEEDBACK LOOP HARDENING

### Current problem
Some feedback paths write to agent_triggers correctly. Others don't. The Approve/Reject/Revision buttons work. But:
- Master-strategist action items don't create feedback paths
- Image revision requests sometimes go to a non-existent image-engine agent
- There's no generic "send note to agent" UI

### What it needs

**Universal feedback widget:** Appears on every transmission card (in Morning Queue, Approvals tab, anywhere). A small "Send to Agent" link that expands a text field + dropdown to pick which agent receives it. Writes to agent_triggers with the appropriate agent_name.

**Image revision:** Fix the dispatch. image-engine does NOT exist. Image revision requests must go to approval-watcher, not image-engine:
```sql
INSERT INTO agent_triggers (agent_name, reason, payload, source)
VALUES ('approval-watcher', 'image_revision', 
  jsonb_build_object('transmission_id', :id, 'feedback', :text),
  'dashboard_image_revision');
```

**Auto-acknowledge on action:** When Bodhi clicks Approve, Reject, or submits feedback on a transmission, any agent_notifications related to that transmission should be auto-acknowledged.

---

## FEATURE 5 -- AGENT RENAMING (requires parallel scheduled task update)

Current names -> new names:
- mirror-content-engine -> mirror-track-a-agent
- mirror-track-b-engine -> mirror-track-b-agent

The dashboard should display new names. But the rename also requires:
1. Updating the scheduled task registrations in the scheduled tasks system
2. Updating agent_name references in all prompt files
3. The agent_triggers and agent_notifications tables use agent_name as a string -- old rows keep old names, new rows use new names. Dashboard should handle both during transition.

---

## BUILD ORDER

1. Feature 3 (Chief of Staff panel improvement) -- lowest risk, biggest morning impact
2. Feature 1 (Clickable notifications) -- unblocks the feedback loop
3. Feature 4 (Feedback loop hardening) -- fixes the image revision bug
4. Feature 2 (Agent journey map) -- awaiting Bodhi's design screenshot for reference
5. Feature 5 (Agent renaming) -- coordinate with scheduled task rename

---

## NOTES FOR BUILD SESSION

- Bodhi will share a design screenshot for the agent journey map. Use it as the primary reference for Feature 2.
- Keep the dark aesthetic. All new UI uses existing CSS variables.
- Real-time subscriptions already work for transmissions. Extend to agent_notifications and agent_triggers using the same Supabase subscription pattern.
- Bell badge must count BOTH transmission events AND unacknowledged action_needed agent_notifications.
- Do not auto-acknowledge agent_notifications on bell open. Persist until explicit dismiss.
