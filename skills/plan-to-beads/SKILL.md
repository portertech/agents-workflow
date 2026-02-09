---
name: plan-to-beads
description: Create a beads epic from the current plan file, inserting the ENTIRE PLAN TEXT into the epic and creating subtasks to track all work items.
---

# Plan to Beads

Converts a plan shared in chat into a beads epic with subtasks.

## Instructions

When the user asks to create a beads epic from their plan:

1. **Ensure stealth mode is initialized**: Check if beads is already initialized for the project. If not, run:
   ```bash
   bd init --stealth
   ```
   Stealth mode uses beads locally without committing files to the main repo.

2. **Identify the plan in chat**: Look for the most recent plan shared in the conversation.

3. **Extract the plan content**: Capture the ENTIRE TEXT of the plan exactly as shared

4. **Create a beads epic**: Use `bd create -t epic` to create an epic with:
   - A descriptive title based on the plan's goal
   - The complete plan text in the description
   ```bash
   bd create -t epic --title "Epic Title" --description "$(cat <<'EOF'
   <ENTIRE PLAN TEXT HERE>
   EOF
   )"
   ```

5. **Create subtasks**: Create subtasks for each work item using `bd create`:
   ```bash
   bd create --title "Subtask Title" --parent <epic-id> --description "$(cat <<'EOF'
   <SUBTASK DESCRIPTION FROM PLAN>
   EOF
   )"
   ```
   - Include the full task description from the plan (not just the title)
   - Capture each subtask ID from the output for linking dependencies

6. **Link dependencies**: Parse `**Depends on**: Task N` markers from the plan. Use `bd dep` to create blocking relationships:

   ```bash
   # At creation time
   bd create --title "Subtask Title" --parent <epic-id> --deps <blocker-id>

   # After creation
   bd dep add <blocked-id> <blocker-id>
   ```

   **Rules for dependencies:**
   - Sequential tasks: each task depends on the one before it
   - Explicit `**Depends on**: Task N` markers always create a dependency

7. **Verify and sync**:
   ```bash
   bd graph <epic-id>   # Visual dependency graph
   bd sync              # Persist changes
   ```

8. **Copy the epic id**: Put the epic id on the clipboard
   ```bash
   echo <epic-id> | pbcopy
   ```

## Notes

- Always use stealth mode (`bd init --stealth`) for personal task tracking
- The epic description should contain the COMPLETE plan text for reference
- Use `bd graph <epic-id>` to verify the dependency structure before finishing
- If no plan is found in chat, ask the user to share their plan first
