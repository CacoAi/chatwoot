# Team Mentions Specification

## Overview
Extend Chatwoot's existing mention system to support team mentions, where mentioning a team (`@team_name`) notifies all members of that team.

## Current Mention System Analysis

### Backend Architecture

#### Mention Model (`app/models/mention.rb`)
- **Schema**: `user_id`, `conversation_id`, `account_id`, `mentioned_at`
- **Constraints**: Unique per user per conversation
- **Associations**: `belongs_to :user, :conversation, :account`
- **Callbacks**: `after_commit :notify_mentioned_user`

#### Mention Processing (`app/services/messages/mention_service.rb`)
- **Trigger**: Only on private messages (`message.private?`)
- **Parsing**: Extracts `(mention://user/ID/NAME)` from message content
- **Validation**: Mentioned users must be inbox members or account administrators
- **Actions**:
  - Creates notifications via `NotificationBuilder`
  - Adds users as conversation participants
  - Enqueues `UserMentionJob`

#### Job Processing (`app/jobs/conversations/user_mention_job.rb`)
- Creates/updates `Mention` records
- Updates `mentioned_at` timestamp for existing mentions

#### Notification System
- **Type**: `conversation_mention` (enum value: 4)
- **Special Handling**: Works even for blocked conversations
- **Delivery**: Push/email based on user preferences

### Frontend Architecture

#### Editor Integration (`app/javascript/dashboard/components/widgets/WootWriter/Editor.vue`)
- **Trigger**: `@` character in private notes only
- **Plugin**: ProseMirror suggestions plugin
- **Constraints**: `isAllowed: () => props.isPrivate`

#### UI Components
- **TagAgents.vue**: Agent selection dropdown with search
- **MentionBox.vue**: Generic keyboard-navigable dropdown
- **Format**: Frontend `@` � Backend `(mention://user/ID/NAME)`

## Team Mentions Requirements

### Core Features
1. **Team Selection**: `@team_name` should trigger team mention dropdown
2. **Member Notification**: All team members receive individual notifications using existing `Mention` model
3. **Permission Validation**: Only teams with inbox access can be mentioned
4. **Mention Storage**: Reuse existing `Mention` model - no new tables needed
5. **UI Integration**: Seamless integration with existing mention UI

### Technical Specifications

#### No Database Changes Required
- Reuse existing `Mention` model and `UserMentionJob`
- Team mentions will create individual `Mention` records for each team member
- No additional tables or migrations needed

#### Backend Implementation

##### Simple Enhancement to Mention Service
```ruby
# app/services/messages/mention_service.rb - minimal changes needed
class Messages::MentionService
  private

  def mentioned_ids
    # Parse both user and team mentions
    user_mentions = message.content.scan(%r{\(mention://user/(\d+)/(.+?)\)}).map(&:first)
    team_mentions = message.content.scan(%r{\(mention://team/(\d+)/(.+?)\)}).map(&:first)
    
    # Expand team mentions to individual user IDs
    expanded_user_ids = expand_team_mentions_to_users(team_mentions)
    
    # Combine user mentions with expanded team member IDs
    (user_mentions + expanded_user_ids).uniq
  end

  def expand_team_mentions_to_users(team_ids)
    return [] if team_ids.blank?
    
    # Get valid teams that have inbox access
    valid_teams = message.inbox.account.teams
                        .joins(:team_members)
                        .where(id: team_ids)
                        .where(team_members: { user_id: valid_mentionable_user_ids })
                        .distinct
    
    # Extract all user IDs from valid teams
    valid_teams.joins(:team_members).pluck('team_members.user_id').map(&:to_s)
  end

  def valid_mentionable_user_ids
    @valid_mentionable_user_ids ||= begin
      inbox = message.inbox
      inbox.account.administrators.pluck(:id) + inbox.members.pluck(:id)
    end
  end

  # No other changes needed - existing logic handles the rest!
end
```

#### Frontend Implementation

##### Enhanced TagAgents Component (Reuse Existing)
```vue
<!-- app/javascript/dashboard/components/widgets/conversation/TagAgents.vue -->
<!-- Extend existing component to show both users and teams -->
<script setup>
// ... existing imports
const getters = useStoreGetters();
const agents = computed(() => getters['agents/getVerifiedAgents'].value);
const teams = computed(() => getters['teams/getTeams'].value);

// Combine agents and teams in a single list
const items = computed(() => {
  const agentItems = agents.value.map(agent => ({
    ...agent,
    type: 'user',
    displayName: agent.name,
    displayInfo: agent.email
  }));
  
  const teamItems = teams.value.map(team => ({
    ...team,
    type: 'team', 
    displayName: team.name,
    displayInfo: `${team.team_members?.length || 0} members`
  }));
  
  const allItems = [...agentItems, ...teamItems];
  
  if (!props.searchKey) return allItems;
  
  return allItems.filter(item =>
    item.displayName.toLowerCase().includes(props.searchKey.toLowerCase())
  );
});

const onSelect = () => {
  const selectedItem = items.value[selectedIndex.value];
  emit('selectAgent', selectedItem);
};
</script>

<template>
  <!-- Extend existing template to handle both users and teams -->
  <div>
    <ul v-if="items.length" class="mention--box">
      <li v-for="(item, index) in items" :key="`${item.type}-${item.id}`">
        <div class="flex items-center">
          <!-- User avatar or team icon -->
          <div class="mr-2">
            <Avatar 
              v-if="item.type === 'user'" 
              :src="item.thumbnail" 
              :name="item.displayName" 
              rounded-full 
            />
            <div 
              v-else 
              class="w-8 h-8 rounded-full bg-gray-200 flex items-center justify-center"
            >
              👥
            </div>
          </div>
          <div>
            <h5>{{ item.displayName }}</h5>
            <div class="text-xs">{{ item.displayInfo }}</div>
          </div>
        </div>
      </li>
    </ul>
  </div>
</template>
```

##### Editor Handler for Mention Selection
```javascript
// app/javascript/dashboard/components/widgets/conversation/ReplyBox.vue
// Add handler for team mentions in existing mention selection logic

const onSelectAgent = (selectedItem) => {
  const mentionType = selectedItem.type || 'user'; // 'user' or 'team'
  const mentionText = `(mention://${mentionType}/${selectedItem.id}/${selectedItem.displayName})`;
  
  // Insert mention using existing editor logic
  // ... rest of existing selection logic
};
```

#### UI/UX Considerations

##### Mention Format
- **User mentions**: `@john_doe` � `(mention://user/123/John Doe)`
- **Team mentions**: `@team_support` � `(mention://team/456/Support Team)`

##### Visual Differentiation
- User mentions: Avatar + name
- Team mentions: Team icon (=e) + team name + member count

##### Search Behavior
- Combined dropdown showing both users and teams
- Teams appear with `team:` prefix or special icon
- Filter by typing team name

##### Notification Experience
- Individual notifications for each team member
- Notification shows "mentioned you via @team_name"
- Team mention tracked separately for analytics

### Implementation Phases

#### Phase 1: Backend Changes (Minimal)
1. Extend `Messages::MentionService.mentioned_ids` method to parse team mentions
2. Add `expand_team_mentions_to_users` method
3. Test team mention parsing and user expansion

#### Phase 2: Frontend Changes (Reuse Existing)
4. Extend `TagAgents.vue` to show both users and teams
5. Update mention selection handler to generate team mention format
6. Add visual differentiation (team icon vs user avatar)

#### Phase 3: Testing & Polish
7. Test team mention flow end-to-end
8. Add unit tests for team expansion logic
9. Update documentation

### Edge Cases & Considerations

1. **Empty Teams**: Automatically handled - no users to expand to
2. **Permission Validation**: Only teams with inbox access are expanded (handled in `expand_team_mentions_to_users`)
3. **Large Teams**: No rate limiting needed - existing notification system handles this
4. **Duplicate Notifications**: Handled by `uniq` in `mentioned_ids` and existing mention logic
5. **Team Deletion**: Gracefully handled - deleted teams won't be found in database query
6. **Cross-Account**: Team expansion respects account boundaries via existing team queries

### Testing Requirements

#### Backend Tests (Minimal)
- Test `expand_team_mentions_to_users` method with various team configurations
- Test team mention parsing in `mentioned_ids`
- Test permission validation for team mentions

#### Frontend Tests (Extend Existing)
- Test combined user/team dropdown functionality
- Test mention format generation for teams
- Test visual differentiation between users and teams

### Benefits of This Simple Approach

1. **No Database Changes**: Reuse existing `Mention` model and infrastructure
2. **Minimal Backend Changes**: Only extend the mention parsing logic
3. **Reuse Frontend Components**: Extend existing `TagAgents` component
4. **Same Permissions**: Team mentions follow same rules as user mentions
5. **Same Notifications**: Individual team members get standard mention notifications
6. **Easy Rollback**: Minimal changes make it easy to revert if needed

### Future Enhancements (Optional)

1. **Team Mention Analytics**: Track which teams are mentioned most
2. **Team Mention Display**: Show team name in mention display instead of individual names
3. **Smart Team Suggestions**: Suggest relevant teams based on conversation context

## Questions for Clarification

1. Should team mentions work the same way as user mentions (private notes only)?
2. How should we handle very large teams (e.g., 50+ members)?
3. Should there be any special permissions for who can mention teams?
4. Do we want to show a preview of team members when hovering over team mentions?
5. Should we track team mentions separately for reporting/analytics?