# Dun Dun Dungeon

Dun Dun Dungeon is a persistent role-playing game inside ChatGPT. You create a character, choose a dungeon, and describe what you want to do in your own words. Between rooms, you decide whether to press on or retreat. Characters, equipment, and progression carry across runs. Players can also author dungeons for others.

## Technical architecture

ChatGPT interprets player intent and narrates the outcome. A Cloudflare Worker exposes the game through MCP, assembles the context for each turn, and runs the rules engine. Supabase stores characters, authored dungeons, runs, and rewards.

The walkthrough uses *The Dame in the Rain*, a published noir dungeon. The story, opening puzzle, and voice are drawn from the authored dungeon. Payloads are abridged, and the sample player turns and outcomes illustrate how the system can respond. The walkthrough reveals the first room's clue.

### 1. A builder authors the adventure

The builder workflow saves the story and voice as one checkpoint and rooms as another before publishing the dungeon. The story gives the narrator a throughline across rooms. Each room provides an immediate situation and objective. A puzzle also supplies facts the narrator can use to recognize useful approaches and pace the reveal. Voice is separate so it can shape the telling without changing the plot or the rules.

**Story**

A private eye takes a missing-husband case. As the investigation progresses, the detective begins to suspect the husband is hiding from the woman who hired them.

**Room**

In **The Cold-Water Flat**, the detective searches the missing husband's apartment. Someone has already been through it. The goal is to find out where he went.

**Puzzle notes**

The magazines beside the well-worn chair include one mailed to Room 312 at the Halcyon Arms, near the Blue Saint Lounge. That address is the clue the narrator can reveal as the player investigates.

### 2. Starting a run gives the narrator the current scene

After the player picks a character and dungeon, `start_run` opens the entryway. The game waits for the player to enter before presenting the first room. At a puzzle room, the Worker combines authored content with current character and run context. The model receives the scene, spoiler-bearing DM notes, routing instructions, and the voice for that adventure.

```text
[SCENE]
Dungeon story: A private eye takes a missing-husband case.
Room: The Cold-Water Flat — puzzle
Summary: The husband's apartment was left in a hurry and has already been searched.
Setup: Drawers hang open. By the radiator, a stack of magazines and unpaid bills has been disturbed.
Room goal: Figure out where the husband went.

[DM NOTES - DO NOT NARRATE]
Secret: A magazine points to a second address near the Blue Saint Lounge.
Happy path: Inspect the worn chair and look through the magazine stack for the mailing address.

[VOICE]
DUNGEON VOICE
register: wry
style: hardboiled noir
instruction: Use a world-weary first-person narrator. Open on a specific detail.
             Keep sentences lean and let a concrete fact end the scene.
examples:
- I took her money. It smelled wrong, like flowers somebody bought to cover something, and I took it anyway.
```

The narrator might say, *“I pushed the door open. Three drawers stood out from the dresser. Somebody had cut the mattress. Beside the radiator, the papers had been moved. What do you look at first?”* The player hears a situation they can act on. The model has the additional facts it needs to judge how their ideas fit the scene. The DM notes are guidance about what to reveal in play.

### 3. A freeform reply becomes a rules-engine action

The player says, *“I check the chair and look through the papers beside it for a name or an address.”* The model can map that attempt to Perception and send the current run sequence. The engine evaluates the approach against the room and character, applies the result to the run, and returns the outcome. Progress, damage, and room clearance are separate facts.

```json
{
  "name": "take_action",
  "arguments": {
    "run_id": "00000000-0000-4000-8000-000000000001",
    "action": "choose_approach",
    "action_sequence": 7,
    "action_params": {
      "skill": "perception",
      "summary": "Search the chair and nearby papers for a name or address."
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

*“I picked the bills out of the pile. Same name, same address. The magazines underneath had been thumbed through until the spines split.”* The immediate action moved the puzzle forward. The detective has narrowed the search, and the player gets another turn to examine the magazines.

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

In this dungeon, continuing from the flat takes the detective to The Rain-Slick Block, where they notice someone following them. The host has enough information to present the cleared room and advance on the player's reply. This recovery path reads the run without replaying the room or granting rewards again. For actions that advance the run, the database update can require the expected action sequence. That prevents two competing writes from both advancing from the same turn.

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
