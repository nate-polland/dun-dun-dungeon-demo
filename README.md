# Dun Dun Dungeon

Dun Dun Dungeon is a persistent role-playing game inside ChatGPT. You create a character, choose a dungeon, and describe what you want to do in your own words. Between rooms, you decide whether to press on or retreat. Characters, equipment, and progression carry across runs. Players can also author dungeons for others.

## Technical architecture

ChatGPT interprets player intent and narrates the outcome. A Cloudflare Worker exposes the game through MCP, assembles the context for each turn, and runs the rules engine. Supabase stores characters, authored dungeons, runs, and rewards.

The examples below follow one fictional dungeon, *The Bell Below*. They use the game's real block names and tool fields with invented content and abridged payloads.

### 1. A builder authors the adventure

The builder workflow saves the story and voice as one checkpoint and rooms as another before publishing the dungeon. The story gives the narrator a throughline across rooms. Each room provides an immediate situation and objective. A puzzle also supplies facts the narrator can use to recognize useful approaches and pace the reveal. Voice is separate so it can shape the telling without changing the plot or the rules.

**Story**

A harbor's warning bell rings from beneath a shuttered tide station. Each chamber reveals how the station once diverted floodwater. In the finale, the player must decide what to protect when the water reaches the old floodgate.

**Room**

In the Signal Chamber, three bronze plates control a locked sluice. Water is rising through the floor grates. The player needs to open the sluice before the chamber floods.

**Puzzle notes**

The plates represent the warning signals for boats at different depths. A line of old tide marks gives the order. The narrator can respond to a player studying the marks or comparing signals while leaving the answer for the player to discover.

### 2. Starting a run gives the narrator the current scene

After the player picks a character and dungeon, `start_run` opens the entryway. The game waits for the player to enter before presenting the first room. At a puzzle room, the Worker combines authored content with current character and run context. The model receives the scene, spoiler-bearing DM notes, routing instructions, and the voice for that adventure.

```text
[SCENE]
Dungeon story: A harbor's warning bell rings from beneath a shuttered tide station.
Room: Signal Chamber — puzzle
Summary: Three bronze signal plates control a locked sluice.
Setup: Water wells up through the floor grates. Old tide marks score the wall.
Room goal: Open the sluice.
Character: Mara, level 2 rogue.

[DM NOTES - DO NOT NARRATE]
Secret: The plates represent boat warning signals for different depths.
Happy path: Match the plates to the tide marks in order.

[VOICE]
DUNGEON VOICE
register: eerie
style: harbor watch
instruction: Describe the danger through physical details.
             Treat the old machinery with the gravity of a public official.
examples:
- The smallest plate rang first. Under the grate, something rang back.
- The sluice had held all winter. The tide was asking for another hour.
```

The narrator might say, *“Water pushed through the grate, one square at a time. Three bronze plates stood beside the locked sluice. The wall above them was scored with old tide marks. What do you do?”* The player hears a situation they can act on. The model has the additional facts it needs to judge how their ideas fit the scene. The DM notes are guidance about what to reveal in play.

### 3. A freeform reply becomes a rules-engine action

The player says, *“I study the markings and compare them with the symbols on the plates.”* The model can map that attempt to Perception and send the current run sequence. The engine evaluates the approach against the room and character, applies the result to the run, and returns the outcome. Progress, damage, and room clearance are separate facts.

```json
{
  "name": "take_action",
  "arguments": {
    "run_id": "00000000-0000-4000-8000-000000000001",
    "action": "choose_approach",
    "action_sequence": 7,
    "action_params": {
      "skill": "perception",
      "summary": "Compare the tide marks with the bronze plate symbols."
    }
  }
}
```

An abridged narration block for an attempt that makes progress without clearing the room:

```text
[RESULT]
outcome: progress
progress_delta: 3
damage: 0

[NARRATION TASK]
Narrate the outcome in DUNGEON VOICE.
```

*“The marks counted depths. The plates counted boats. You found where the two systems met, though one plate still refused to make sense.”* The immediate action moved the puzzle forward. The sluice is still shut, so the player gets another turn in the same room.

### 4. The tool response governs what can happen next

When the room clears, the game asks the narrator to describe the result and wait. At an eligible room boundary, the player can continue or retreat. The tool response can carry a specific follow-up operation or an explicit set of choices. During onboarding, a cleared room can route to a coaching exchange before play resumes.

An abridged response when a returning player syncs an already-cleared room:

```json
{
  "status": "room_already_cleared",
  "state": { "action_sequence": 9, "first_run": false },
  "render": { "transition": "room_cleared" },
  "next": {
    "kind": "wait_for_player",
    "resume": {
      "tool": "take_action",
      "arguments": {
        "run_id": "00000000-0000-4000-8000-000000000001",
        "action": "continue"
      }
    }
  }
}
```

The host has enough information to present the cleared room and advance on the player's reply. This recovery path reads the run without replaying the room or granting rewards again. For actions that advance the run, the database update can require the expected action sequence. That prevents two competing writes from both advancing from the same turn.

### 5. The run ends and the character persists

At a terminal outcome, the game routes to `end_run` for settlement. The settlement path tracks whether gold and XP have already been applied so a repeated call does not award them twice. The post-run flow uses the outcome and character state to decide which steps apply. It can route to `record_traits`, `mint_loot`, or a level-up choice. When another step applies, the tool response carries the next operation so the host can recover the flow from the current response if earlier conversation has been compacted.

```text
Terminal outcome
  → end_run
  → record_traits, when available
  → mint_loot, when eligible
  → level-up choice, when pending
```

The run has an end state. The character retains the applicable progression, traits, and equipment for the next adventure.

### What I test before shipping

I test engine outcomes and tool contracts in code, then play through rooms to check whether the result makes sense to a player. Two changes illustrate the difference:

- **Room recovery.** Syncing a cleared room could skip first-run coaching. The recovery path now routes based on the saved run. A regression test exercises sync and repeated choice for first-time and returning players, and checks that neither path writes run state or grants rewards again.
- **MCP transport.** Some clients passed only structured content to the model. Removing scene and voice from that response made the authored context disappear for those clients. The Worker now builds one narration packet and carries it in both response forms. Tests check that the copies match and that scene context survives room and boss transitions.

In playthroughs I look for whether the narrator explains the immediate stakes, preserves the dungeon's voice, and reveals discoveries when the player earns them. Those checks cover the experience the player sees when all the parts work together.
