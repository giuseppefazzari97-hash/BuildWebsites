# Monday.com PMO — Complete Reverse Engineering Specification

> **Purpose**: 1:1 functional rebuild blueprint. Every feature, data model, API operation, widget, column type, automation, and view documented from live API introspection + platform research.
>
> **Source**: Live GraphQL schema introspection, widget schema dump, column type schemas, and exhaustive web research (Oct 2026).

---

## Table of Contents

1. [Platform Architecture & Hierarchy](#1-platform-architecture--hierarchy)
2. [Data Model — Core Entities](#2-data-model--core-entities)
3. [Column Types — Complete Registry](#3-column-types--complete-registry)
4. [Updates & Activity System](#4-updates--activity-system)
5. [Views — All Types](#5-views--all-types)
6. [Dashboard Widgets — Complete Registry](#6-dashboard-widgets--complete-registry)
7. [Automations Engine](#7-automations-engine)
8. [Dependencies & Critical Path](#8-dependencies--critical-path)
9. [Portfolio & Project Management](#9-portfolio--project-management)
10. [WorkDocs & Knowledge Base](#10-workdocs--knowledge-base)
11. [Forms (WorkForms)](#11-forms-workforms)
12. [Time Tracking](#12-time-tracking)
13. [Resource Management & Workload](#13-resource-management--workload)
14. [Notifications System](#14-notifications-system)
15. [Permissions & Access Control](#15-permissions--access-control)
16. [Integrations Framework](#16-integrations-framework)
17. [AI Features](#17-ai-features)
18. [API — Full GraphQL Reference](#18-api--full-graphql-reference)
19. [Overlapping Tools Matrix](#19-overlapping-tools-matrix)
20. [Implementation Priority Map](#20-implementation-priority-map)

---

## 1. Platform Architecture & Hierarchy

### Structural Hierarchy (top → bottom)

```
Account
├── Workspaces (organizational containers)
│   ├── Folders (group boards/docs/dashboards)
│   │   ├── Boards (the core work surface)
│   │   │   ├── Groups (row sections / categories)
│   │   │   │   ├── Items (individual rows / tasks)
│   │   │   │   │   ├── Subitems (up to 4 levels deep)
│   │   │   │   │   ├── Updates (comments/activity thread)
│   │   │   │   │   └── Column Values (cell data)
│   │   │   │   └── ...more items
│   │   │   ├── Columns (field definitions)
│   │   │   └── Views (Gantt, Kanban, Calendar, etc.)
│   │   ├── Dashboards (multi-board widgets)
│   │   ├── WorkDocs (rich documents)
│   │   └── Sub-folders
│   └── ...more folders
├── Teams (user groupings)
├── Departments (org structure)
└── Users (members, guests, viewers)
```

### Key Concepts

| Concept | Description | Rebuild Equivalent |
|---|---|---|
| **Account** | Top-level tenant. Has plan, products, settings. | Tenant / Organization |
| **Workspace** | Isolated container with its own members. Types: `open`, `closed`. | Project Space / Namespace |
| **Folder** | Organizational grouping within a workspace. Has color, icon, font weight. | Folder / Directory |
| **Board** | The primary work surface. A table of items × columns. Types: `main`, `shareable`, `private`. | Project Board / Table |
| **Group** | A section within a board that groups items. Has color, position, collapsed state. | Section / Category |
| **Item** | A single row on a board. The fundamental work unit. | Task / Record / Row |
| **Subitem** | A child item nested under a parent item. Up to 4 levels of nesting. | Subtask |
| **Column** | A field definition on a board. Defines the data type and settings for a vertical slice. | Field / Column Definition |
| **Column Value** | The actual data in a cell (intersection of item × column). | Cell Value |
| **Update** | A threaded comment/activity entry on an item. | Comment / Activity Entry |
| **View** | A visual presentation of board data (Gantt, Kanban, Calendar, etc.). | View / Perspective |
| **Dashboard** | A reporting surface that aggregates widgets from multiple boards. | Dashboard / Report |
| **Widget** | A visual component on a dashboard (chart, battery, number, etc.). | Widget / Card |

### Board Types

| API Value | Name | Description |
|---|---|---|
| `public` | Main Board | Visible to all workspace members |
| `private` | Private Board | Visible only to subscribers |
| `share` | Shareable Board | Can be shared with guests outside the account |

### Board Access Levels

```graphql
enum BoardAccessLevel {
  collaborators  # Only invited collaborators
  everyone       # All workspace members  
}
```

### Board Edit Permissions

```graphql
enum BoardEditPermissions {
  everyone       # All board members can edit
  owners_only    # Only board owners can edit
}
```

---

## 2. Data Model — Core Entities

### 2.1 Board

```graphql
type Board {
  id: ID!
  name: String!
  description: String
  state: State!              # active, archived, deleted, all
  board_kind: BoardKind!     # public, private, share
  board_folder_id: ID
  workspace_id: ID
  permissions: String
  columns: [Column!]!
  groups: [Group!]!
  items_count: Int
  items_page: ItemsResponse
  owners: [User!]!
  subscribers: [User!]!
  tags: [Tag!]!
  views: [BoardView!]!
  activity_logs: [ActivityLogType!]
  updates: [Update!]
  top_group: Group
  creator: User
  communication: JSON
  item_terminology: String
  type: ObjectType
  updated_at: ISO8601DateTime
}
```

**Key Operations:**
- `create_board(board_name, board_kind, workspace_id, template_id)` → Board
- `duplicate_board(board_id, duplicate_type)` → BoardDuplication
- `archive_board(board_id)` → Board
- `delete_board(board_id)` → Board
- `update_board(board_id, board_attribute, new_value)` → JSON

### 2.2 Item

```graphql
type Item {
  id: ID!
  name: String!
  state: State
  board: Board
  group: Group
  column_values: [ColumnValue!]!
  subitems: [Item!]
  parent_item: Item
  updates: [Update!]
  assets: [Asset!]
  subscribers: [User!]
  creator: User
  creator_id: String
  created_at: ISO8601DateTime
  updated_at: ISO8601DateTime
  email: String
  relative_link: String
  linked_items: [Item!]
}
```

**Key Operations:**
- `create_item(board_id, group_id, item_name, column_values)` → Item
- `create_subitem(parent_item_id, item_name, column_values)` → Item
- `change_column_value(board_id, item_id, column_id, value)` → Item
- `change_multiple_column_values(board_id, item_id, column_values)` → Item
- `move_item_to_group(item_id, group_id)` → Item
- `move_item_to_board(item_id, board_id, group_id, columns_mapping)` → Item
- `archive_item(item_id)` → Item
- `delete_item(item_id)` → Item
- `duplicate_item(board_id, item_id)` → Item
- `change_item_position(item_id, relative_to, position_relative_method)` → Item

### 2.3 Group

```graphql
type Group {
  id: String!
  title: String!
  color: String!
  position: String!
  archived: Boolean
  deleted: Boolean
  items_page: ItemsResponse
}
```

**Key Operations:**
- `create_group(board_id, group_name, group_color, position, relative_to)` → Group
- `update_group(board_id, group_id, group_attribute, new_value)` → JSON
- `duplicate_group(board_id, group_id)` → Group
- `archive_group(board_id, group_id)` → Group
- `delete_group(board_id, group_id)` → Group

### 2.4 Column

```graphql
type Column {
  id: String!
  title: String!
  type: ColumnType!
  description: String
  settings_str: String       # JSON string of column settings
  width: Int
  archived: Boolean
}
```

### 2.5 User

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  title: String
  phone: String
  photo_original: String
  photo_small: String
  photo_thumb: String
  photo_thumb_small: String
  photo_tiny: String
  birthday: Date
  country_code: String
  location: String
  time_zone_identifier: String
  is_admin: Boolean
  is_guest: Boolean
  is_pending: Boolean
  is_view_only: Boolean
  is_verified: Boolean
  enabled: Boolean
  created_at: ISO8601DateTime
  current_language: String
  account: Account!
  teams: [Team!]
  out_of_office: OutOfOffice
  sign_up_product_kind: String
  last_activity: ISO8601DateTime
}
```

**Key Operations:**
- `users(ids, kind, limit, page)` → [User]
- `me` → User
- `invite_users(emails, role, team_ids)` → InviteUsersResult
- `activate_users(user_ids)` → ActivateUsersResult
- `deactivate_users(user_ids)` → DeactivateUsersResult
- `update_users_role(user_ids, role)` → UpdateUsersRoleResult

### 2.6 Workspace

```graphql
type Workspace {
  id: ID!
  name: String!
  description: String
  kind: WorkspaceKind!       # open, closed
  state: State
  created_at: ISO8601DateTime
  settings: WorkspaceSettings
  owners_subscribers: [User!]
  teams_subscribers: [Team!]
  account_product: AccountProduct
  is_default_workspace: Boolean
}
```

---

## 3. Column Types — Complete Registry

### Complete ColumnType Enum

```graphql
enum ColumnType {
  auto_number
  board_relation        # Connect Boards
  button
  checkbox
  color_picker
  country
  creation_log
  date
  dependency
  doc
  dropdown
  email
  file
  formula
  group
  hour
  integration
  item_id
  last_updated
  link
  location
  long_text
  mirror                # Mirror (reflects connected board data)
  name                  # Item Name (mandatory first column)
  numbers
  people                # Person column
  phone
  progress
  rating
  status                # Status (colored labels)
  subtasks              # Subitems
  tags
  team
  text
  timeline              # Timeline (date range bar)
  time_tracking
  unsupported
  vote
  week
  world_clock
  custom_field          # Custom fields
}
```

### 3.1 Status Column

The most-used column type. Displays colored labels representing workflow states.

**Schema:**
```json
{
  "settings": {
    "labels": [
      {
        "id": 0,           // 0-19, 101-110, 151-160 (max 40 labels)
        "label": "Done",
        "color": "done_green",  // or numeric color ID
        "index": 0,
        "description": "Task is complete",
        "is_deactivated": false,
        "is_done": true    // marks completion state
      }
    ]
  }
}
```

**Available Colors (40 total):**
```
working_orange, done_green, stuck_red, dark_blue, purple, explosive,
grass_green, bright_blue, saladish, egg_yolk, blackish, dark_red,
sofia_pink, lipstick, dark_purple, bright_green, chili_blue,
american_gray, brown, dark_orange, sunset, bubble, peach, berry,
winter, river, navy, aquamarine, indigo, dark_indigo, pecan,
lavender, royal, steel, orchid, lilac, tan, sky, coffee, teal
```

**Value Format (write):**
```json
{ "label": "Done" }
// or by index:
{ "index": 1 }
```

**Value Format (read) — StatusValue:**
```graphql
type StatusValue {
  index: Int
  label: String
  label_style: StatusLabelStyle  # { color, border, var_name }
  update_id: ID
  is_done: Boolean
}
```

### 3.2 Timeline Column

Date range displayed as a horizontal bar. Core to Gantt views.

**Schema:**
```json
{
  "settings": {
    "hide_footer": false,
    "show_set_as_milestone": true,
    "show_weekends": true,
    "show_week_number": false
  }
}
```

**Value Format (write):**
```json
{ "from": "2026-01-01", "to": "2026-03-31" }
```

**Value Format (read) — TimelineValue:**
```graphql
type TimelineValue {
  from: Date
  to: Date
  visualization_type: String  # "milestone" or null
  updated_at: ISO8601DateTime
}
```

### 3.3 Dependency Column

Links items with dependency relationships (FS, SS, FF, SF).

**Schema:**
```json
{
  "settings": {
    "boardIds": [123456],
    "boardId": 123456,
    "dependencyNewInfra": true,
    "allowMultipleItems": true
  }
}
```

**Dependency Relations:**
```graphql
enum DependencyRelation {
  finish_to_start   # FS — predecessor must finish before successor starts
  start_to_start    # SS — both start together
  finish_to_finish  # FF — both finish together
  start_to_finish   # SF — predecessor starts before successor finishes
}
```

**Value Format (write):**
```json
{
  "linkedPulseIds": [
    { "linkedPulseId": 12345, "relation": "finish_to_start" }
  ]
}
```

**Dependency Response Modes:**
- **Flexible**: Dependent items adjust but allow manual override
- **Strict**: Dependent items are locked to the dependency chain
- **No action**: Dependencies are informational only

**Batch Update (up to 50 items):**
```graphql
mutation {
  batch_update_dependency_column(
    board_id: 123
    items: [{ item_id: 1, value: {...} }, ...]
  )
}
```

### 3.4 People Column

Assigns users/teams to items.

**Schema:**
```json
{
  "settings": {
    "hide_footer": false,
    "max_people_allowed": "0"   // "0" = unlimited
  }
}
```

**Value Format (write):**
```json
{
  "personsAndTeams": [
    { "id": 12345, "kind": "person" },
    { "id": 67890, "kind": "team" }
  ]
}
```

**Value Format (read) — PeopleValue:**
```graphql
type PeopleValue {
  persons_and_teams: [PeopleEntity!]
  updated_at: ISO8601DateTime
}
type PeopleEntity {
  id: ID!
  kind: String!    # "person" or "team"
}
```

### 3.5 Numbers Column

Numeric values with optional unit/currency display.

**Schema:**
```json
{
  "settings": {
    "unit": {
      "symbol": "$",           // or "custom"
      "custom_unit": "USD",
      "direction": "left",     // "left" or "right"
      "precision": 2           // -1=auto, 0-5 decimal places
    },
    "hide_footer": false
  }
}
```

**Value Format (write):** `"42.5"` or `42.5`

**Summary Row Functions:** SUM, AVG, MIN, MAX, MEDIAN, COUNT

### 3.6 Date Column

Single date (optionally with time).

**Schema:**
```json
{
  "settings": {
    "show_current_year": true,
    "show_week_day": false,
    "show_week_number": false,
    "show_time_by_default": false,
    "hide_footer": false,
    "use_numeric_only_format": false,
    "show_add_icon": true,
    "show_weekends": true,
    "calcType": "earliest",     // earliest, latest, earliestToLatest
    "date_format": "YYYY-MM-DD",
    "time_format": "HH:mm"
  }
}
```

**Value Format (write):**
```json
{ "date": "2026-10-15", "time": "14:30:00" }
```

### 3.7 Formula Column

Calculated values from other columns on the same item (horizontal only).

**Schema:**
```json
{
  "settings": {
    "formula": "IF({Status}=\"Done\", {Numbers} * 1.1, {Numbers})",
    "hide_footer": false
  }
}
```

**Compatible Source Columns:** Check, Country, Creation Log, Date, Dependency, Dropdown, Email, Formula, Hour, Item ID, Last Updated, Connect Boards, Mirror, Long Text, Numbers, Person, Phone, Rating, Status, Text, Timeline, Time Tracking, Vote

**Available Function Categories:**
- **Math**: ADD, SUBTRACT, MULTIPLY, DIVIDE, POWER, SQRT, ABS, ROUND, ROUNDUP, ROUNDDOWN, MOD, MIN, MAX, LOG, LN
- **Text**: CONCATENATE, LEFT, RIGHT, MID, LEN, LOWER, UPPER, TRIM, SUBSTITUTE, REPLACE, SEARCH, REPT, TEXT
- **Logic**: IF, AND, OR, NOT, XOR, SWITCH, EXACT
- **Date**: DAYS, HOURS_DIFF, MINUTES_DIFF, WORKDAYS, TODAY, FORMAT_DATE, HOUR, MINUTE, MONTH, YEAR, DAY, WEEKDAY, WEEKNUM
- **Lookup**: VALUE (no VLOOKUP — formulas are horizontal-only)

**Limitations:**
- Cannot reference other items (no cross-row calculation)
- Cannot reference column summaries
- Cannot reference other boards
- Read-only (computed, not editable)

### 3.8 Connect Boards (Board Relation)

Links items across boards (one-to-many or many-to-many).

**Schema:**
```json
{
  "settings": {
    "allowCreateReflectionColumn": true,
    "allowMultipleItems": true,
    "boardIds": [123456, 789012],
    "boardId": 123456
  }
}
```

**Value Format (write):**
```json
{ "linkedPulseIds": [{ "linkedPulseId": 12345 }] }
```

### 3.9 Mirror Column

Reflects data from a connected board's columns. Read-only.

**Schema:**
```json
{
  "settings": {
    "relation_column": { "board_relation_mkttvdt2": true },
    "displayed_linked_columns": [
      { "board_id": "1234567890", "column_ids": ["name", "status"] }
    ],
    "sumType": "doneOnly",      // or "allStatuses"
    "calcType": "earliestToLatest"  // earliest, latest, earliestToLatest
  }
}
```

### 3.10 Time Tracking Column

Start/stop timer with manual entry support.

**Schema:**
```json
{
  "settings": {
    "showSeconds": false
  }
}
```

**Value Format (read) — TimeTrackingValue:**
```graphql
type TimeTrackingValue {
  duration: Int              # total seconds
  started: Boolean
  start_date: ISO8601DateTime
  running: Boolean
  history: [TimeTrackingHistoryItem!]
}
type TimeTrackingHistoryItem {
  started_at: ISO8601DateTime
  ended_at: ISO8601DateTime
  manually_entered_start_date: String
  manually_entered_end_date: String
  manually_entered_start_time: String
  manually_entered_end_time: String
  created_at: ISO8601DateTime
  updated_at: ISO8601DateTime
  status: String
}
```

### 3.11 Dropdown Column

Multi-select from predefined options.

**Schema:**
```json
{
  "settings": {
    "labels": [
      { "id": 1, "label": "Option A", "is_deactivated": false },
      { "id": 2, "label": "Option B", "is_deactivated": false }
    ],
    "limit_select": false,
    "label_limit_count": 5
  }
}
```

### 3.12 Remaining Column Types (Quick Reference)

| Column Type | Value Type | Write Format | Notes |
|---|---|---|---|
| `auto_number` | ItemIdValue | Read-only | Auto-incrementing ID |
| `button` | ButtonValue | N/A | Triggers automation |
| `checkbox` | CheckboxValue | `{ "checked": "true" }` | Boolean toggle |
| `color_picker` | ColorPickerValue | `{ "color": "#FF0000" }` | Hex color |
| `country` | CountryValue | `{ "countryCode": "AU", "countryName": "Australia" }` | ISO country |
| `creation_log` | CreationLogValue | Read-only | Auto: creator + timestamp |
| `email` | EmailValue | `{ "email": "x@y.com", "text": "label" }` | Clickable email |
| `file` | FileValue | Upload via `add_file_to_column` | Attachments |
| `hour` | HourValue | `{ "hour": 14, "minute": 30 }` | Time of day |
| `item_id` | ItemIdValue | Read-only | Item's unique ID |
| `last_updated` | LastUpdatedValue | Read-only | Auto: last editor + timestamp |
| `link` | LinkValue | `{ "url": "https://...", "text": "label" }` | Clickable URL |
| `location` | LocationValue | `{ "lat": -33.8, "lng": 151.2, "address": "..." }` | Map pin |
| `long_text` | LongTextValue | `{ "text": "..." }` | Multi-line text (markdown) |
| `phone` | PhoneValue | `{ "phone": "+61...", "countryShortName": "AU" }` | Phone number |
| `progress` | ProgressValue | Read-only | Auto-calculated from subitems |
| `rating` | RatingValue | `{ "rating": 4 }` | 1-5 star rating |
| `tags` | TagsValue | `{ "tag_ids": [123, 456] }` | Shared tags across boards |
| `team` | TeamValue | `{ "team_id": 123 }` | Team assignment |
| `text` | TextValue | `"plain text value"` | Single-line text |
| `vote` | VoteValue | Read-only | Upvote counter |
| `week` | WeekValue | `{ "week": { "startDate": "2026-10-12", "endDate": "2026-10-18" } }` | Calendar week |
| `world_clock` | WorldClockValue | `{ "timezone": "Australia/Sydney" }` | Timezone clock |

---

## 4. Updates & Activity System

### 4.1 Updates (Comments / Conversations)

The Updates section is an item-level threaded conversation system — social-media style.

```graphql
type Update {
  id: ID!
  body: String!              # HTML-formatted content
  text_body: String          # Plain text version
  creator_id: ID
  creator: User
  item_id: ID
  assets: [Asset!]           # Attached files
  replies: [Reply!]          # Threaded replies
  likes: [Like!]             # Reactions
  pinned_to_top: [UpdatePin!]
  created_at: ISO8601DateTime
  updated_at: ISO8601DateTime
}

type Reply {
  id: ID!
  body: String!
  text_body: String
  creator_id: ID
  creator: User
  created_at: ISO8601DateTime
  updated_at: ISO8601DateTime
}
```

**Key Operations:**
```graphql
# Create an update (comment) on an item
create_update(
  item_id: ID!
  body: String!               # Supports HTML: <b>, <i>, <a>, @mentions
  parent_id: ID               # If replying to existing update
  mentions: [UpdateMention]   # @mention users/teams
) → Update

# Edit an existing update
edit_update(id: ID!, body: String!) → Update

# Delete an update
delete_update(id: ID!) → Update

# Like / Unlike
like_update(update_id: ID!) → Update
unlike_update(update_id: ID!) → Update

# Pin / Unpin to top
pin_to_top(update_id: ID!) → Update
unpin_from_top(update_id: ID!) → Update
```

**Features to Rebuild:**
1. **@Mentions**: `@mention` a user or team → triggers notification
2. **Rich Text**: Bold, italic, links, inline code, lists
3. **File Attachments**: Images, documents, videos attached to updates
4. **Reactions**: Emoji reactions on updates and replies
5. **Seen Tracking**: Eye icon shows who has viewed the update
6. **GIF Support**: Inline GIF picker
7. **Threaded Replies**: Nested replies under each update
8. **Pin to Top**: Pin important updates to the top of the thread
9. **AI Summary**: AI-generated summary of the conversation thread

### 4.2 Activity Log

Tracks every change made to an item or board.

```graphql
type ActivityLogType {
  id: String!
  account_id: String!
  user_id: String!
  entity: String!            # "board", "pulse" (item), etc.
  event: String!             # "create_pulse", "update_column_value", etc.
  data: String!              # JSON payload of what changed
  created_at: String!
}
```

**Query:**
```graphql
query {
  boards(ids: [123]) {
    activity_logs(
      from: "2026-01-01T00:00:00Z"
      to: "2026-10-15T23:59:59Z"
      column_ids: ["status", "person"]
      group_ids: ["group_1"]
      item_ids: [456]
      user_ids: [789]
      limit: 100
      page: 1
    ) {
      id
      event
      data
      created_at
      user_id
    }
  }
}
```

**Filterable By:** Person, Time Range, Group, Column, Item

### 4.3 Audit Logs (Enterprise)

Account-level security audit trail.

```graphql
query {
  audit_logs(
    user_id: "123"
    events: ["login", "logout", "board_create", "permission_change"]
    ip_address: "1.2.3.4"
    start_time: "2026-01-01T00:00:00Z"
    end_time: "2026-10-15T23:59:59Z"
    limit: 100
    page: 1
  ) {
    logs {
      timestamp
      event
      ip_address
      user { id, name, email }
      activity_metadata
    }
    pagination { page, page_size, has_more_pages, next_page_number }
  }
}

# Get all possible audit event types:
query { audit_event_catalogue { name, description, metadata_details } }
```

### 4.4 Custom Activity (Timeline)

Custom activity entries on an item's timeline.

```graphql
type CustomActivity {
  id: ID!
  color: CustomActivityColor  # blue, purple, green, yellow, red, dark_blue
  icon: CustomActivityIcon
  content: String
  created_at: ISO8601DateTime
  user: User
}

type TimelineItem {
  id: ID!
  custom_activity: CustomActivity
  likes: [Like!]
  replies: [Reply!]
  created_at: ISO8601DateTime
}
```

---

## 5. Views — All Types

### 5.1 View System

```graphql
type BoardView {
  id: ID!
  name: String!
  type: String!              # "GanttView", "KanbanView", etc.
  settings_str: String       # JSON configuration
  view_specific_data_str: String
}

enum ViewKind {
  BoardDefaultLayoutView
  BoardCalendarView
  BoardCardView              # Kanban
  BoardChartView
  BoardFilesGalleryView
  BoardFormView
  BoardGanttView
  BoardKanbanView
  BoardTableView
  BoardTimelineView
  BoardWorkloadView
  BoardBlankView
}
```

**Operations:**
```graphql
create_view(board_id, view_kind, settings) → BoardView
update_view(view_id, settings) → BoardView
delete_view(view_id) → BoardView
```

### 5.2 Table View (Default)

The standard spreadsheet-like board view.

```graphql
input TableViewSettingsInput {
  order_columns: [String!]        # Column display order
  show_hide_columns: [String!]    # Hidden column IDs
  item_height: String             # "single", "double", "triple"
  wrapped_column_ids: [String!]   # Text-wrapped columns
  subitems_display_mode: String   # nested, like_items, with_parents
}
```

### 5.3 Gantt View

Timeline visualization with dependencies, milestones, and critical path.

**Settings Schema** (from live widget dump):
```json
{
  "columnIdsByBoardId": { "board_id": ["timeline_col_id"] },
  "groupByIdPerBoardId": { "board_id": ["none"|"board"|"group"|"parent"] },
  "colorByIdPerBoardId": { "board_id": ["group"|"status_col_id"|"board"|"parent"|"none"] },
  "label_by_id_per_board": { "board_id": ["none"|"name"] },
  "color_by_type": "group",     // none, board, group, color, parent
  "label_by_type": "name",      // none, name
  "group_by_type": "none",      // none, board, group, color, parent
  "show_today_line": true,
  "show_weekends": true,
  "is_docked": true
}
```

**Gantt-Specific Features:**
- **Milestones**: Diamond markers for key dates (set via `visualization_type: "milestone"` on timeline)
- **Critical Path**: Highlights the longest chain of dependent tasks (Pro/Enterprise)
- **Baseline**: Compare planned vs actual timelines
- **Dependencies**: Visual arrows between linked items
- **Drag-to-Resize**: Adjust timeline bars by dragging edges
- **Today Line**: Vertical indicator for current date
- **Zoom Levels**: Day, Week, Month, Quarter, Year

### 5.4 Kanban View

Card-based columns organized by status.

**Features:**
- Cards grouped by any status/dropdown column
- Drag-and-drop between columns to change status
- Card preview showing key column values
- WIP (work-in-progress) limits per column
- Swimlanes for secondary grouping
- Color-coded cards

### 5.5 Calendar View

Date/timeline data rendered as calendar events.

**Settings Schema** (from live widget dump):
```json
{
  "columns": { "board_id": ["date", "timeline", "creation_log6"] },
  "colorByIdPerBoardId": { "board_id": ["group"|"status"|"board"] },
  "color_by_type": "group",
  "show_time_slots": true,
  "hide_weekends": false,
  "event_duration": 30,         // 30, 45, 60, 90, 120, 180, 240 minutes
  "view_mode": "month",         // month, week, day
  "label_by_type": "board",     // board, group, parent, name, text, status, etc.
  "label_by_id": { "board_id": "column_id" },
  "groups_data": { "group_ids_per_board": { "board_id": ["group_1"] } },
  "exclude_board_ids": [0]
}
```

**Supported Column Types for Calendar:** date, timeline, week, creation_log, last_updated, lookup, formula

### 5.6 Workload View

Resource capacity visualization per person/team.

**Features:**
- Shows effort per person across a time range
- Capacity bar per person (green → yellow → red)
- Effort based on numbers column (story points, hours, etc.)
- Daily/weekly/monthly granularity
- Drag to rebalance work
- Available on Pro/Enterprise plans

### 5.7 Chart View

Interactive charts from board data (also available as dashboard widget).

### 5.8 Files Gallery View

Visual gallery of all file attachments on the board.

### 5.9 Form View

Public-facing form that creates items when submitted (see Section 11).

---

## 6. Dashboard Widgets — Complete Registry

### Widget Type Registry (from live schema dump)

| Widget Type | API Key | Description |
|---|---|---|
| **Battery** | `BATTERY` | Progress bar showing completion % based on status columns |
| **Calendar** | `CALENDAR` | Calendar display of date/timeline data |
| **Chart** | `CHART` | Bar, pie, line, area, stacked, donut, bubble charts |
| **Number** | `NUMBER` | Single aggregate number (sum, avg, min, max, median, count) |
| **Gantt/Timeline** | `GANTT` | Multi-board Gantt chart widget |
| **List View** | `LISTVIEW` | Filtered item list from multiple boards |
| **App Feature** | `APP_FEATURE` | Marketplace app widget |

### 6.1 Battery Widget

Shows completion percentage across boards like a battery indicator.

```json
{
  "done_text": "Done",
  "battery_data": {
    "status_column_ids_per_board": {
      "board_id": ["status", "status2"]
    },
    "numeric_column_ids_per_board": {
      "board_id": ["numbers", "story_points"]
    }
  },
  "groups_data": {
    "group_ids_per_board": {
      "board_id": ["planned", "in_progress"]
    }
  }
}
```

**How it works:**
1. Counts items where status label matches `done_text` = completed
2. Total items vs completed items = percentage
3. Optional numeric weighting (story points) for weighted completion
4. Can filter to specific groups per board

### 6.2 Chart Widget

Full charting engine with 16+ chart types.

**Chart Types Available:**
```
pie, donut, column, bar, area, line, smooth_line, bubbles,
stacked_bar, stacked_bar_percent, stacked_column, stacked_column_percent,
stacked_area, stacked_area_percent, stacked_line, stacked_line_percent
```

**Configuration:**
```json
{
  "graph_type": "column",
  "x_axis_columns": { "board_id": ["status"] },
  "y_axis_columns": { "board_id": ["default-label-count"] },
  "z_axis_columns": { "board_id": ["priority"] },      // for stacked charts
  "x_axis_group_by": "color",    // color, group, multiple-person, board, name, date
  "y_axis_group_by": "default-label-count",  // default-label-count, numeric, numbers
  "calc_function_type": "sum",   // sum, average, min, max, count
  "sort_by": "Y_AXIS_DESCENDING",
  "group_by": "month"            // month, week, day, quarter, year (when date axis)
}
```

### 6.3 Number Widget

Single KPI/metric display.

```json
{
  "counter_data": {
    "calculation_type": "columns",  // "columns" or "count"
    "column_ids_per_board": { "board_id": ["numbers"] },
    "counter_type": "sum",          // sum, average, median, min, max
    "counter_unit": {
      "symbol": "$",
      "direction": "left",
      "custom_unit": "AUD"
    },
    "should_count_items": false,
    "should_count_subitems": false
  },
  "prefix": "Total Revenue",
  "suffix": "this quarter",
  "number_format": "currency"       // number, percentage, currency
}
```

### 6.4 Dashboard Operations

```graphql
# Create dashboard
create_dashboard(
  name: String!
  workspace_id: ID
  kind: DashboardKind        # public, private
) → Dashboard

# Add widget to dashboard
create_widget(
  dashboard_id: ID!
  widget_type: String!       # "battery", "chart", "number", etc.
  title: String!
  board_ids: [ID!]!
  settings: JSON
) → Widget

# Delete widget
delete_widget(widget_id: ID!) → Widget
```

### 6.5 Aggregate Query (Advanced Analytics)

Powerful server-side aggregation engine for dashboards.

```graphql
query {
  aggregate(
    input: {
      from: { board_ids: [123], element_type: ITEMS }
      select: [{
        element: { type: COLUMN, column_id: "numbers" }
        function: { name: SUM }
      }]
      group_by: [{
        element: { type: COLUMN, column_id: "status" }
      }]
    }
  ) {
    ... on AggregateGroupByResult {
      groups {
        group_key
        aggregations { value }
      }
    }
  }
}
```

**Aggregate Functions:** `SUM`, `AVERAGE`, `MIN`, `MAX`, `COUNT`, `COUNT_DISTINCT`, `MEDIAN`

---

## 7. Automations Engine

### 7.1 Architecture

Every automation = **Trigger** + **Condition** (optional) + **Action(s)**

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐
│  TRIGGER  │ ──► │  CONDITION   │ ──► │   ACTION(S)  │
│           │     │  (optional)  │     │              │
│ Event     │     │ Filter/Gate  │     │ One or more  │
└──────────┘     └──────────────┘     └──────────────┘
```

### 7.2 Trigger Types

| Category | Trigger | Description |
|---|---|---|
| **Status** | Status changes | When a status column changes to a specific value |
| **Status** | All statuses change | When all items in a group reach a status |
| **Date** | Date arrives | When a date column reaches today |
| **Date** | Date passed | When a date column is in the past |
| **Date** | X days before/after date | Relative date trigger |
| **Item** | Item created | When a new item is added to the board |
| **Item** | Item moved to group | When an item is moved between groups |
| **Item** | Item archived/deleted | When an item is archived or deleted |
| **Column** | Column value changes | When any column value on an item changes |
| **Column** | Person assigned | When someone is assigned via people column |
| **Recurring** | Every time period | Daily, weekly, monthly, yearly |
| **Recurring** | Every X days/weeks | Interval-based |
| **Subitem** | Subitem created | When a subitem is added |
| **Subitem** | Subitem status changes | When subitem status changes |
| **Form** | Form submitted | When a WorkForm submission is received |
| **Integration** | Email received | When an email arrives (via integration) |
| **Custom** | Webhook received | External webhook trigger |

### 7.3 Action Types

| Category | Action | Description |
|---|---|---|
| **Notify** | Send notification | In-app notification to specific users |
| **Notify** | Send email | Email to specified addresses |
| **Status** | Change status | Set a status column to a value |
| **Assignment** | Assign person | Set the people column |
| **Assignment** | Clear person | Remove all assignees |
| **Date** | Set date | Set a date column value |
| **Date** | Set date to today | Current date |
| **Date** | Clear date | Remove date value |
| **Move** | Move item to group | Move within same board |
| **Move** | Move item to board | Move to different board |
| **Create** | Create item | Create a new item on any board |
| **Create** | Create subitem | Create subitem under trigger item |
| **Create** | Create update | Post a comment on the item |
| **Create** | Duplicate item | Copy the triggering item |
| **Archive** | Archive item | Archive the triggering item |
| **Delete** | Delete item | Permanently delete |
| **Column** | Set column value | Change any column to a value |
| **Column** | Clear column | Remove column value |
| **External** | Send webhook | POST to external URL |
| **External** | Create in integration | Push to Slack, Teams, Jira, etc. |
| **Time** | Start timer | Begin time tracking |
| **Time** | Stop timer | End time tracking |
| **Dependency** | Set dependency | Create dependency link |

### 7.4 Automation API

```graphql
# Automation statistics
query {
  account_trigger_statistics(
    filters: { board_ids: [123] }
  ) {
    total_runs
    successful_runs
    failed_runs
  }
}

# Trigger events
query {
  trigger_events(
    filters: {
      board_id: 123
      state: COMPLETED    # COMPLETED, FAILED, RUNNING
    }
  ) {
    uuid
    trigger_id
    state
    created_at
  }
}
```

### 7.5 Plan Limits

| Plan | Monthly Actions |
|---|---|
| Free | 250 |
| Basic | N/A (not available) |
| Standard | 250 |
| Pro | 25,000 |
| Enterprise | 250,000 |

---

## 8. Dependencies & Critical Path

### 8.1 Dependency System

**Dependency Types (all four schedule relationship types):**

| Type | Code | Meaning |
|---|---|---|
| Finish-to-Start | `FS` | B can't start until A finishes (most common) |
| Start-to-Start | `SS` | B can't start until A starts |
| Finish-to-Finish | `FF` | B can't finish until A finishes |
| Start-to-Finish | `SF` | B can't finish until A starts |

**Response Modes:**

| Mode | Behavior |
|---|---|
| **Flexible** | Dependent dates shift but can be manually overridden |
| **Strict** | Dependent dates are locked; cannot violate dependency |
| **No action** | Dependencies are visual/informational only |

### 8.2 Dependency Graph Export

```graphql
query {
  export_graph(board_id: 123) {
    nodes { id, name, start_date, end_date }
    edges { from_id, to_id, relation_type }
  }
}
```

### 8.3 Critical Path

- Available in Gantt view on Pro/Enterprise plans
- Highlights the longest chain of dependent tasks
- Any delay on the critical path delays the entire project
- Displayed as a distinct visual style on Gantt bars
- Recalculated automatically when dependencies or dates change

### 8.4 Milestones

- Set on timeline column values with `visualization_type: "milestone"`
- Displayed as diamond shapes on the Gantt chart
- Represent key deliverables or phase completions
- Zero-duration markers (single date, not a range)

---

## 9. Portfolio & Project Management

### 9.1 Project Object

Monday.com has a first-class Project entity beyond regular boards.

```graphql
# Create project from scratch
create_project(
  name: String!
  privacy: PrivacyKind          # public, private
  workspace_id: ID
  folder_id: ID
  template_id: ID
  companion_ids: [ID]           # Resource Planner, etc.
  callback_url: String          # Async — notified when complete
) → CreateProjectResult { process_id }

# Convert existing board to project
convert_board_to_project(
  board_id: ID!
  columns_mapping: ColumnsMappingInput
  callback_url: String
) → ConvertBoardToProjectResult { process_id }

# Connect project to portfolio
connect_project_to_portfolio(
  project_id: ID!
  portfolio_id: ID!
) → ConnectProjectResult
```

### 9.2 Portfolio

Aggregates multiple projects for executive visibility.

```graphql
create_portfolio(
  name: String!
  workspace_id: ID
  folder_id: ID
) → CreatePortfolioResult
```

**Portfolio Dashboard Features:**
- Roll-up status across all projects
- Budget tracking (aggregated numbers columns)
- Timeline overview (all project timelines on one Gantt)
- Resource allocation across projects
- RAG (Red/Amber/Green) health indicators via battery widget
- Milestone tracking across the portfolio

### 9.3 Sprint Management (monday dev)

```graphql
type Sprint {
  id: ID!
  name: String!
  state: SprintState!         # active, closed, future
  timeline: SprintTimeline    # start_date, end_date
  board: Board
  snapshots: [SprintSnapshot!]
}

type SprintSnapshot {
  kind: SprintSnapshotKind    # start, end, daily
  items: [SprintSnapshotItem!]
  column_metadata: [SprintSnapshotColumnMetadata!]
  created_at: ISO8601DateTime
}
```

**Sprint Operations:**
- `sprints(board_ids)` → [Sprint]
- `get_sprint_summary(sprint_id)` → summary data
- `get_sprints_metadata(board_id)` → sprint config

---

## 10. WorkDocs & Knowledge Base

### 10.1 Document System

```graphql
type Document {
  id: ID!
  name: String
  doc_kind: DocKind           # doc, knowledge_base_article
  workspace_id: ID
  board: Board
  blocks: [DocumentBlock!]
  created_at: ISO8601DateTime
  updated_at: ISO8601DateTime
  created_by: User
  object_id: ID
  settings: JSON
  url: String
  relative_url: String
}

type DocumentBlock {
  id: String!
  type: String!               # paragraph, header, list, image, table, etc.
  content: JSON
  parent_block_id: String
  position: Float
  created_at: ISO8601DateTime
  updated_at: ISO8601DateTime
}
```

**Document Operations:**
```graphql
create_doc(name, board, workspace, kind) → Document
delete_doc(doc_id) → Document
duplicate_doc(doc_id) → Document
update_doc_name(doc_id, name) → Document
create_doc_block(doc_id, type, content, after_block_id, parent_block_id) → DocumentBlock
update_doc_block(block_id, content) → DocumentBlock
delete_doc_block(block_id) → DocumentBlock
add_content_to_doc_from_markdown(doc_id, markdown, after_block_id) → DocBlocksFromMarkdownResult
export_markdown_from_doc(doc_id, block_ids) → ExportMarkdownResult
import_doc_from_html(html, name, workspace_id) → ImportDocFromHtmlResult
```

### 10.2 Version History

```graphql
doc_version_history(doc_id, since, until) → DocVersionHistory {
  restoring_points: [DocRestoringPoint {
    date: ISO8601DateTime
    users: [User]
  }]
}

doc_version_diff(doc_id, from_date, to_date) → DocVersionDiff {
  blocks: [DiffBlock {
    id, type, change_type, content
  }]
}
```

### 10.3 Knowledge Base

```graphql
# Articles (published docs)
create_article(workspace_id, name, folder_id) → Article
publish_article(object_id, privacy, folder_id, subscriber_ids, team_ids) → Article
update_article_block(block_id, content) → ArticleBlock
delete_article(object_id) → Boolean

# AI-powered search
knowledge_base_search(query, workspace_id) → [SnippetSearchResult]
```

### 10.4 Item Description

Each item has an optional rich-text description (separate from updates).

```graphql
set_item_description_content(
  item_id: ID!
  markdown: String!
) → [DocumentBlockIdOnly]
```

---

## 11. Forms (WorkForms)

### 11.1 Form System

Forms create board items from public submissions.

```graphql
type ResponseForm {
  token: String!              # Unique form identifier
  board: Board
  name: String
  description: String
  status: String              # active, inactive
  questions: [FormQuestion!]
  settings: FormSettings
  appearance: FormAppearance
  features: FormFeatures
  shortened_link: FormShortenedLink
}

type FormQuestion {
  id: ID!
  title: String!
  type: FormQuestionType!
  description: String
  required: Boolean
  hidden: Boolean
  settings: FormQuestionSettings
  options: [FormQuestionOption!]
  column_id: String           # Maps to board column
  position: Float
}
```

**Question Types:**
```graphql
enum FormQuestionType {
  short_text
  long_text
  number
  email
  phone
  url
  date
  date_time
  dropdown
  multi_select
  single_select
  status
  rating
  file_upload
  section_header
  divider
  country
  location
  checkbox
}
```

### 11.2 Form Operations

```graphql
create_form(board_id, name, description) → ResponseForm
update_form(token, input: UpdateFormInput) → ResponseForm
update_form_settings(token, input: UpdateFormSettingsInput) → ResponseForm
activate_form(token) → ResponseForm
deactivate_form(token) → ResponseForm
create_form_question(token, input: CreateQuestionInput) → FormQuestion
update_form_question(token, question_id, input: UpdateQuestionInput) → FormQuestion
delete_question(token, question_id) → Boolean
shorten_form_url(token) → FormShortenedLink
set_form_password(token, input: SetFormPasswordInput) → ResponseForm
create_form_tag(token, input) → FormTag    # UTM tracking tags
```

### 11.3 Form Customization

**Appearance:**
```json
{
  "background": { "type": "color", "value": "#FFFFFF" },
  "font_size": "medium",         // small, medium, large
  "direction": "ltr",            // ltr, rtl
  "alignment": "left",           // left, center, right
  "format": "classic",           // classic, card, conversational
  "layout": { "type": "split" },
  "logo": { "position": "top", "size": "medium" }
}
```

**Features:**
```json
{
  "close_date": { "enabled": true, "date": "2026-12-31" },
  "response_limit": { "enabled": true, "limit": 100 },
  "require_login": { "enabled": false },
  "password": { "enabled": false },
  "draft_submission": { "enabled": true },
  "redirect_after_submission": { "enabled": true, "url": "https://..." },
  "pre_submission_view": { "title": "Welcome", "description": "..." },
  "after_submission_view": { "title": "Thank you!", "description": "..." }
}
```

---

## 12. Time Tracking

### 12.1 How It Works

1. **Start Timer**: Click play button on time tracking cell → records `started_at` timestamp
2. **Stop Timer**: Click pause → records `ended_at`, calculates duration
3. **Resume**: Click play again → new history entry
4. **Manual Entry**: Add session manually with start/end times
5. **Delete Entry**: Hover over log entry → click X to remove

### 12.2 Data Model

```graphql
type TimeTrackingValue {
  column: Column!
  duration: Int!                    # Total seconds tracked
  started: Boolean!                 # Has any tracking started
  running: Boolean!                 # Is timer currently running
  start_date: ISO8601DateTime       # When current session started
  history: [TimeTrackingHistoryItem!]!
}

type TimeTrackingHistoryItem {
  started_at: ISO8601DateTime!
  ended_at: ISO8601DateTime
  manually_entered_start_date: String
  manually_entered_end_date: String
  manually_entered_start_time: String
  manually_entered_end_time: String
  created_at: ISO8601DateTime!
  updated_at: ISO8601DateTime
  status: String                    # "active", "completed"
}
```

### 12.3 Time Tracking Widget (Dashboard)

Aggregates time tracking data across boards for reporting.

### 12.4 Automation Integration

- **When status changes to X → Start timer**
- **When status changes to Y → Stop timer**
- **When timer reaches X hours → Notify manager**

---

## 13. Resource Management & Workload

### 13.1 Workload View

**Purpose:** See team capacity at a glance and rebalance work.

**How it works:**
1. Select a **People column** (who is assigned)
2. Select an **Effort column** (numbers column: hours, story points, etc.)
3. Set **capacity per person** (e.g., 40 hours/week)
4. View shows effort vs. capacity as color-coded bars:
   - Green: Under capacity
   - Yellow: Near capacity
   - Red: Over capacity
5. Drag items between people to rebalance

### 13.2 Workload Widget (Dashboard)

Shows workload across multiple boards.

**Configuration:** Uses the Gantt schema with people-based grouping.

### 13.3 Resource Directory

```graphql
query {
  get_directory_resources(
    attributes: [JOB_ROLE, SKILLS, LOCATION]
  ) {
    resources {
      user { id, name }
      attributes { job_role, skills, location }
    }
  }
}

mutation {
  update_directory_resources_attributes(
    user_ids: [123]
    attributes: { job_role: "Developer", skills: ["React", "Node"] }
  )
}
```

**Directory Resource Attributes:**
```graphql
enum DirectoryResourceAttribute {
  JOB_ROLE
  SKILLS
  LOCATION
}
```

---

## 14. Notifications System

### 14.1 In-App Notifications

```graphql
type Notification {
  id: ID!
  text: String
  user_id: ID
  target_type: NotificationTargetType  # Project, Post
}

# Create notification
create_notification(
  text: String!
  user_id: ID!
  target_id: ID!
  target_type: NotificationTargetType!
) → Notification
```

### 14.2 Notification Settings

```graphql
type NotificationSetting {
  channel: String           # email, push, in_app, desktop
  scope: ScopeType         # "account", "board"
  enabled: Boolean
}

# Update settings
update_notification_setting(
  channel_type: ChannelType!
  scope_type: ScopeType!
  enabled: Boolean!
) → NotificationSetting
```

### 14.3 Board Mute Settings

```graphql
enum BoardMuteState {
  muted_for_all          # Silenced for everyone
  muted_for_me           # Silenced for current user only
  mentions_only          # Only @mentions and assignments
  unmuted                # All notifications active
}

mutation {
  update_mute_board_settings(
    board_id: ID!
    mute_state: BoardMuteState!
  ) → BoardMuteSettings
}
```

### 14.4 Notification Triggers

Notifications fire on:
- @mentions in updates
- Item assignment (people column change)
- Status changes (configurable)
- Due date approaching / passed
- Subitem creation
- Reply to your update
- Automation-triggered notifications
- Form submission

---

## 15. Permissions & Access Control

### 15.1 User Roles

```graphql
enum UserRole {
  admin           # Full account control
  member          # Standard user
  viewer          # Read-only access
  guest           # External limited access
}

enum UserKind {
  all
  non_guests
  guests
  non_pending
}
```

### 15.2 Board-Level Permissions

```graphql
enum BoardBasicRoleName {
  owner           # Full control, can delete
  editor          # Can edit items and columns
  commenter       # Can add updates but not edit
  viewer          # Read-only
}

# Set board permission
set_board_permission(
  board_id: ID!
  role: BoardBasicRoleName!
) → SetBoardPermissionResponse
```

### 15.3 Workspace Permissions

```graphql
enum WorkspaceKind {
  open            # Any account member can join
  closed          # Invitation-only
}

enum WorkspaceMembershipKind {
  owner
  subscriber
}

# Add/remove users
add_users_to_workspace(workspace_id, user_ids, kind)
delete_users_from_workspace(workspace_id, user_ids)
add_teams_to_workspace(workspace_id, team_ids)
delete_teams_from_workspace(workspace_id, team_ids)
```

### 15.4 Account-Level Permissions

Admins can control per-role:
- Create boards and docs
- Invite guests
- Export to Excel
- Generate API tokens
- Create automations/integrations
- Create workspaces
- Manage teams
- Upload/download files

---

## 16. Integrations Framework

### 16.1 Native Integrations (200+)

| Category | Tools |
|---|---|
| **Communication** | Slack, Microsoft Teams, Gmail, Outlook, Zoom, Google Calendar |
| **CRM** | Salesforce, HubSpot, Pipedrive |
| **Development** | GitHub, GitLab, Jira, Bitbucket |
| **Storage** | Google Drive, Dropbox, OneDrive, Box |
| **Finance** | QuickBooks, Xero |
| **Marketing** | Mailchimp, HubSpot Marketing, Facebook Ads |
| **Design** | Figma, Adobe Creative Cloud |
| **Support** | Zendesk, Freshdesk, Intercom |
| **Automation** | Zapier, Make (Integromat), Power Automate |

### 16.2 Webhooks

```graphql
type Webhook {
  id: ID!
  board_id: ID!
  event: WebhookEventType!
  config: String             # JSON config
}

enum WebhookEventType {
  change_column_value
  change_specific_column_value
  change_status_column_value
  change_subitem_column_value
  change_name
  create_item
  item_archived
  item_deleted
  item_moved_to_any_group
  item_moved_to_specific_group
  create_subitem
  change_subitem_name
  subitem_archived
  subitem_deleted
  create_update
  edit_update
  delete_update
  create_subitem_update
}

# Create webhook
create_webhook(
  board_id: ID!
  url: String!
  event: WebhookEventType!
  config: JSON
) → Webhook
```

### 16.3 Integration Execution

```graphql
mutation {
  execute_integration_block(
    block_id: ID!
    field_values: JSON!
  ) → IntegrationExecutionResult
}
```

---

## 17. AI Features

### 17.1 Monday AI Capabilities

| Feature | What It Does |
|---|---|
| **monday magic** | Instantly builds complete workflows from natural language |
| **monday vibe** | No-code app builder using plain English prompts |
| **monday sidekick** | Conversational AI assistant with full business context |
| **monday agents** | Autonomous AI specialists that execute end-to-end tasks |
| **AI Blocks** | Modular AI building blocks for automations |
| **AI Summarize** | Summarizes update threads on items |
| **AI Compose** | Writes updates, emails, content from prompts |
| **AI Categorize** | Auto-categorizes items by analyzing content |
| **AI Formulas** | Generates formula column expressions from natural language |
| **AI Workflows** | AI-powered automation recipes |

### 17.2 AI in Updates

- **Summarize Thread**: Generates a concise summary of all updates on an item
- **Compose Reply**: AI-assisted response drafting
- **Extract Action Items**: Pulls tasks from conversation threads
- **Translate**: Translates updates to different languages

### 17.3 Notetaker (Meeting Intelligence)

```graphql
type Meeting {
  id: ID!
  title: String
  access: MeetingAccess          # owner, shared
  participants: [Participant!]
  topics: [Topic!]
  talking_points: [TalkingPoint!]
  action_items: [ActionItem!]
  transcript: [TranscriptEntry!]
  created_at: ISO8601DateTime
}

query {
  notetaker {
    meetings(
      filters: { access: OWNER }
      first: 10
    ) {
      nodes { id, title, topics { ... }, action_items { ... } }
    }
  }
}
```

---

## 18. API — Full GraphQL Reference

### 18.1 Endpoint

```
POST https://api.monday.com/v2
Authorization: Bearer <API_TOKEN>
Content-Type: application/json
API-Version: 2024-10
```

### 18.2 All Query Fields

| Query | Description |
|---|---|
| `me` | Current authenticated user |
| `users` | List users |
| `teams` | List teams |
| `account` | Account info |
| `boards` | List/get boards |
| `items` | Get items by IDs |
| `items_page_by_column_values` | Search items by column values |
| `next_items_page` | Paginate board items |
| `tags` | List tags |
| `updates` | List updates |
| `replies` | Get replies |
| `docs` | List documents |
| `folders` | List folders |
| `workspaces` | List workspaces |
| `webhooks` | List webhooks |
| `assets` | Get assets by IDs |
| `complexity` | Query complexity data |
| `version` / `versions` | API version info |
| `audit_logs` | Security audit trail |
| `audit_event_catalogue` | Audit event types |
| `sprints` | Sprint data |
| `aggregate` | Server-side aggregation |
| `notifications` | User notifications |
| `form` | Get form by token |
| `objects` | Objects Platform query |
| `object_types_unique_keys` | Available object types |
| `article_blocks` | Article content |
| `articles` | Published articles |
| `doc_version_history` | Document versions |
| `doc_version_diff` | Document diffs |
| `export_markdown_from_doc` | Export doc as markdown |
| `knowledge_base_search` | AI-powered doc search |
| `export_graph` | Dependency graph |
| `dependency_column_config` | Dependency settings |
| `connections` / `connection` | Integration connections |
| `trigger_events` / `block_events` / `tool_events` | Automation events |
| `all_widgets_schema` | Widget configuration schemas |
| `get_column_type_schema` | Column type schemas |
| `get_view_schema_by_type` | View type schemas |
| `notetaker.meetings` | Meeting notes |
| `platform_api` | API analytics |
| `account_roles` | Role definitions |
| `favorites` | User favorites |

### 18.3 All Mutation Fields

| Mutation | Description |
|---|---|
| `create_board` / `duplicate_board` / `archive_board` / `delete_board` / `update_board` | Board CRUD |
| `create_item` / `create_subitem` / `duplicate_item` / `archive_item` / `delete_item` | Item CRUD |
| `move_item_to_group` / `move_item_to_board` / `change_item_position` | Item movement |
| `change_column_value` / `change_multiple_column_values` / `change_simple_column_value` | Value updates |
| `create_column` / `update_column` / `delete_column` / `change_column_title` / `change_column_metadata` | Column management |
| `create_status_column` / `update_status_column` / `create_dropdown_column` / `update_dropdown_column` | Typed column ops |
| `create_group` / `update_group` / `duplicate_group` / `archive_group` / `delete_group` | Group CRUD |
| `create_update` / `edit_update` / `delete_update` / `clear_item_updates` | Update CRUD |
| `like_update` / `unlike_update` / `pin_to_top` / `unpin_from_top` | Update actions |
| `create_doc` / `delete_doc` / `duplicate_doc` / `update_doc_name` | Document CRUD |
| `create_doc_block` / `update_doc_block` / `delete_doc_block` | Doc block CRUD |
| `add_content_to_doc_from_markdown` / `import_doc_from_html` / `export_markdown_from_doc` | Doc conversion |
| `create_workspace` / `update_workspace` / `delete_workspace` | Workspace CRUD |
| `create_folder` / `update_folder` / `delete_folder` | Folder CRUD |
| `create_dashboard` / `update_dashboard` / `delete_dashboard` | Dashboard CRUD |
| `create_widget` / `delete_widget` | Widget CRUD |
| `create_view` / `update_view` / `delete_view` | View CRUD |
| `create_notification` | Send notification |
| `create_webhook` / `delete_webhook` | Webhook management |
| `add_file_to_column` / `add_file_to_update` | File uploads |
| `add_users_to_board` / `delete_subscribers_from_board` / `add_teams_to_board` / `delete_teams_from_board` | Board members |
| `add_users_to_workspace` / `delete_users_from_workspace` / `add_teams_to_workspace` / `delete_teams_from_workspace` | Workspace members |
| `add_users_to_team` / `remove_users_from_team` / `create_team` / `delete_team` | Team management |
| `invite_users` / `activate_users` / `deactivate_users` / `update_users_role` | User management |
| `create_project` / `convert_board_to_project` / `create_portfolio` / `connect_project_to_portfolio` | Project/Portfolio |
| `create_form` / `update_form` / `update_form_settings` / `activate_form` / `deactivate_form` | Form management |
| `create_form_question` / `update_form_question` / `delete_question` | Form questions |
| `create_object` / `update_object` / `delete_object` / `archive_object` | Objects Platform |
| `set_board_permission` | Permission management |
| `update_dependency_column` / `batch_update_dependency_column` | Dependency management |
| `create_article` / `publish_article` / `delete_article` / `update_article_block` | Knowledge base |
| `set_item_description_content` | Item description |
| `update_notification_setting` / `update_mute_board_settings` | Notification config |
| `create_department` / `update_department` / `delete_department` | Department management |
| `enroll_items_to_sequence` | CRM sequences |
| `update_directory_resources_attributes` | Resource directory |

---

## 19. Overlapping Tools Matrix

These features overlap significantly and need unified implementation:

| Function | Where It Appears | Overlap |
|---|---|---|
| **Status Tracking** | Status column, Battery widget, Kanban view, Automation triggers | Same data drives 4+ surfaces |
| **Timeline/Dates** | Timeline column, Date column, Gantt view, Calendar view, Workload view, Timeline widget | Same date data → 6 views |
| **People/Assignment** | People column, Workload view, Calendar labels, Chart x-axis, Automation assignment | Person data → 5+ contexts |
| **Dependencies** | Dependency column, Gantt arrows, Critical path, Automation triggers | Graph data → 4 consumers |
| **Progress** | Battery widget, Progress column, Status rollup, Sprint burndown | Completion → 4 calculations |
| **Grouping** | Groups (board), Kanban columns, Chart x-axis, Gantt swimlanes, Workload rows | Categorical grouping → 5 views |
| **Filtering** | Board filters, View filters, Dashboard filters, Automation conditions, API queries | Same filter engine everywhere |
| **Notifications** | In-app bell, Email, Push, Desktop, Automation actions, @mentions | Same event → 6 channels |
| **File Management** | File column, Update attachments, WorkDoc embeds, Form uploads | Files → 4 attachment points |
| **Comments/Updates** | Item updates, Subitem updates, Doc comments, Form responses | Threaded content → 4 surfaces |
| **Charts/Reporting** | Chart view, Chart widget, Battery widget, Number widget, Workload | Analytics → 5 chart surfaces |
| **Search** | Board search, Cross-board search, Knowledge base search, API search | Text search → 4 scopes |
| **Templates** | Board templates, Project templates, Doc templates, Form templates, Automation templates | Templatization → 5 entity types |

### Unified Component Recommendations

To avoid rebuilding overlapping logic:

1. **Unified Filter Engine**: One filter/query engine that powers board filters, view filters, dashboard queries, automation conditions, and API searches
2. **Unified Renderer**: One column value renderer that works in table cells, Kanban cards, Gantt labels, Calendar events, and form fields
3. **Unified Notification Bus**: One event system that routes to in-app, email, push, desktop, and webhook channels
4. **Unified Timeline Engine**: One date/timeline engine that powers Gantt, Calendar, Workload, and Timeline views
5. **Unified Chart Engine**: One charting library that powers Chart view, Chart widget, Battery widget, and Number widget

---

## 20. Implementation Priority Map

### Phase 1 — Core Data Model (Foundation)
1. Account / Workspace / Folder hierarchy
2. Board CRUD with Groups
3. Item CRUD with Subitems (4 levels)
4. Column type system (all 30+ types)
5. Column value read/write for all types
6. User / Team / Permission model

### Phase 2 — Views & Visualization
7. Table View (default board view)
8. Kanban View (card-based)
9. Gantt View (timeline bars, milestones)
10. Calendar View (date visualization)
11. Chart View (16+ chart types)

### Phase 3 — PMO Features
12. Dependencies (FS/SS/FF/SF + response modes)
13. Critical Path calculation
14. Workload / Resource Management
15. Portfolio & Project management
16. Dashboard system with all 6 widget types

### Phase 4 — Collaboration
17. Updates system (threaded comments, @mentions, reactions)
18. Activity Log & Audit Trail
19. Notifications (in-app, email, push)
20. WorkDocs & Knowledge Base
21. Forms (WorkForms)

### Phase 5 — Automation & Integration
22. Automation engine (triggers, conditions, actions)
23. Webhook system (inbound + outbound)
24. Integration framework
25. Time Tracking

### Phase 6 — Intelligence
26. AI Summarize / Compose
27. AI Workflows / Blocks
28. Search (cross-board, knowledge base)
29. Formula engine
30. Aggregate query engine

---

## Sources

- [Monday.com PMO Software](https://monday.com/work-management/pmo)
- [Monday.com Column Types](https://support.monday.com/hc/en-us/articles/115005310285-Available-column-types-on-monday-com)
- [Monday.com Updates Section](https://support.monday.com/hc/en-us/articles/115005900249-The-Updates-Section)
- [Monday.com Activity Log](https://support.monday.com/hc/en-us/articles/115005310745-The-Activity-Log)
- [Monday.com Board Views](https://support.monday.com/hc/en-us/articles/360001267945-The-board-views)
- [Monday.com Gantt Chart](https://support.monday.com/hc/en-us/articles/360015643840-The-Gantt-Chart-View-and-Widget)
- [Monday.com Automations](https://support.monday.com/hc/en-us/articles/360001222900-Get-started-with-monday-automations)
- [Monday.com Dependencies](https://support.monday.com/hc/en-us/articles/360007402599-Dependencies-on-monday-com)
- [Monday.com Structural Hierarchy](https://support.monday.com/hc/en-us/articles/7278527605906-Understanding-monday-com-s-structural-hierarchy)
- [Monday.com Subitems](https://support.monday.com/hc/en-us/articles/360011905480-All-about-subitems)
- [Monday.com Formula Column](https://support.monday.com/hc/en-us/articles/360001235445-The-Formula-Column)
- [Monday.com Time Tracking](https://support.monday.com/hc/en-us/articles/360001143809-The-Time-Tracking-Column)
- [Monday.com Workload](https://support.monday.com/hc/en-us/articles/360010166559-Resource-management-with-Workload)
- [Monday.com WorkDocs](https://support.monday.com/hc/en-us/articles/360021702939-Get-started-with-monday-workdocs)
- [Monday.com Permissions](https://support.monday.com/hc/en-us/articles/360019222479-Permissions-on-monday-com)
- [Monday.com AI Feature Catalog](https://support.monday.com/hc/en-us/articles/24047211522194-AI-Feature-Catalog)
- [Monday.com API Reference](https://developer.monday.com/api-reference/reference/about-the-api-reference)
- [Monday.com GraphQL Overview](https://developer.monday.com/api-reference/docs/introduction-to-graphql)
- [Monday.com Column Types Reference](https://developer.monday.com/api-reference/reference/column-types-reference)
- [The 19 Types of Views on Monday.com](https://www.simonsezit.com/article/types-of-views-on-monday-com/)
