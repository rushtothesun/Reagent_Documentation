# ReAgent Plugin Documentation

## 1. Introduction

ReAgent is a powerful automation and information display tool for Path of Exile, designed to work with the ExileCore framework. It allows users to create complex, rule-based actions that trigger automatically based on the game's state.

The core of ReAgent is its rule engine, which operates on a simple "if this, then that" principle. You define conditions (`if`), and when they are met, you specify actions (`then that`) to be executed.

## 2. Core Concepts

ReAgent's configuration is structured hierarchically:

-   **Profiles**: The top-level container. You can have multiple profiles for different characters, builds, or situations (e.g., a mapping profile, a bossing profile). You can only have one active profile at a time.
-   **Rule Groups**: Each profile consists of one or more groups. Groups are a way to organize your rules. You can enable or disable entire groups at once.
-   **Rules**: Each group contains a list of rules. A rule is a combination of a condition and an action.

## 3. Rule Engine

### 3.1. Rule Conditions

A rule's condition is a C# expression that evaluates to a boolean (`true` or `false`). If the expression evaluates to `true`, the rule's action is triggered.

You have access to a rich set of game state information within your rule conditions, exposed through the `RuleState` object.

**Example:**

```csharp
Vitals.HP.Percent < 50
```

This condition will be `true` when your character's health drops below 50%.

### 3.2. Expression Syntax (V1 vs. V2)

ReAgent supports two different syntaxes for writing rule conditions, which can be toggled in the rule editor via the "Use new syntax" checkbox.

-   **V1 (Legacy):** This is the original syntax, powered by the `System.Linq.Dynamic.Core` library. It is simpler but less powerful. It does not support multi-line expressions, variable declarations, or complex logic. It is best suited for very simple, one-line conditions.

-   **V2 (New):** This syntax is powered by Roslyn (`Microsoft.CodeAnalysis.Scripting`) and allows you to write much more complex, multi-line C# code. You can declare variables, write comments, and use more advanced language features. For any non-trivial rule, it is highly recommended to use the V2 syntax.

**Examples of V2 syntax rules:**

```csharp
// Example 1: Find the closest rare monster within 50 units
// Use this when you need to check properties of the closest monster
var closestRare = Monsters(50, MonsterRarity.Rare)
                  .OrderBy(m => m.Distance)
                  .FirstOrDefault();

// Return true only if a rare monster was found
closestRare != null
```

```csharp
// Example 2: Advanced filtering - Find rare monsters within 100 units that are also invincible
// This demonstrates chaining multiple filters before ordering
var invincibleRare = Monsters(100, MonsterRarity.Rare)
                     .Where(m => m.IsInvincible)
                     .OrderBy(m => m.Distance)
                     .FirstOrDefault();

// Only trigger if we found an invincible rare monster
invincibleRare != null
```

```csharp
// Example 3: Count-based logic - Trigger when surrounded by many enemies
// Useful for defensive skills or area-of-effect abilities
var nearbyEnemyCount = MonsterCount(30, MonsterRarity.Any);

// Return true if 5 or more enemies are within 30 units
nearbyEnemyCount >= 5
```

### 3.3. Rule Actions (`SideEffects`)

When a condition is met, ReAgent can perform various actions, called `SideEffects`. There are three types of rule actions:

1.  `Key`: Presses a specified key.
2.  `SingleSideEffect`: Executes a single, more complex action.
3.  `MultipleSideEffects`: Executes a list of actions.

## 4. The `RuleState` Object: Your Gateway to Game Data

The `RuleState` object provides a snapshot of the current game state. You can access its properties in your rule conditions. Here is a comprehensive list of available properties:

### Player Information

| Property           | Type              | Description                                                                                             |
| ------------------ | ----------------- | ------------------------------------------------------------------------------------------------------- |
| `Player`           | `MonsterInfo`     | An object representing the player character, with all the properties of a monster (see `MonsterInfo`).    |
| `Vitals`           | `VitalsInfo`      | Provides access to the player's Health, Mana, and Energy Shield.                                        |
| `Buffs`            | `BuffDictionary`  | A dictionary of all active buffs on the player.                                                         |
| `Ailments`         | `IReadOnlyCollection<string>` | A collection of custom ailment groups that are currently affecting the player (e.g., "Bleeding", "Cursed"). Defined in `CustomAilments.json`. |
| `Skills`           | `SkillDictionary` | A dictionary of the player's active skills.                                                             |
| `WeaponSwapSkills` | `SkillDictionary` | A dictionary of the player's skills on their inactive weapon set.                                       |
| `Animation`        | `AnimationE`      | The player's current animation.                                                                         |
| `AnimationId`      | `int`             | The numeric ID of the player's current animation.                                                       |
| `AnimationStage`   | `int`             | The current stage of the player's animation.                                                            |
| `IsMoving`         | `bool`            | `true` if the player is currently moving.                                                               |
| `Flasks`           | `FlasksInfo`      | Provides access to the player's flasks.                                                                 |
| `MapStats`         | `StatDictionary`  | See [`StatDictionary`](README.md:248) for usage. |
| `MousePosition`    | `Vector2`         | The current world grid position of the mouse cursor.                                                    |

### Game & Area Information

| Property                 | Type     | Description                                                              |
| ------------------------ | -------- | ------------------------------------------------------------------------ |
| `IsInHideout`            | `bool`   | `true` if the player is in their hideout.                                |
| `IsInTown`               | `bool`   | `true` if the player is in a town.                                       |
| `IsInPeacefulArea`       | `bool`   | `true` if the player is in any non-hostile area.                         |
| `IsInEscapeMenu`         | `bool`   | `true` if the escape menu is open.                                       |
| `AreaName`               | `string` | The name of the current area.                                            |
| `IsChatOpen`             | `bool`   | `true` if the chat panel is open.                                        |
| `IsLeftPanelOpen`        | `bool`   | `true` if the inventory or character panel is open.                      |
| `IsRightPanelOpen`       | `bool`   | `true` if the world map or atlas panel is open.                          |
| `IsAnyFullscreenPanelOpen` | `bool`   | `true` if any fullscreen UI is open (e.g., Atlas, Passive Tree).       |
| `IsAnyLargePanelOpen`    | `bool`   | `true` if any large UI panel is open (e.g., Stash, Vendor).               |

### Entity & Monster Information

| Method/Property        | Return Type                 | Description                                                                                                                            |
| ---------------------- | --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `Monsters(range, rarity)` | `IEnumerable<MonsterInfo>`  | Returns a collection of hostile monsters within the specified `range` and matching the given `rarity` (`MonsterRarity.Normal`, `MonsterRarity.Magic`, `MonsterRarity.Rare`, `MonsterRarity.Unique`, `MonsterRarity.Any`). |
| `MonsterCount(range, rarity)` | `int`                       | Returns the number of hostile monsters matching the criteria.                                                                          |
| `AllMonsters`          | `IEnumerable<MonsterInfo>`  | Returns all valid monsters in the area, regardless of distance.                                                                        |
| `HiddenMonsters`       | `IEnumerable<MonsterInfo>`  | Returns monsters that are present but hidden (e.g., Abyss monsters before they emerge).                                                |
| `Corpses`              | `IEnumerable<MonsterInfo>`  | Returns all monster corpses in the area.                                                                                               |
| `FriendlyMonsters`     | `IEnumerable<MonsterInfo>`  | Returns all friendly monsters (minions, allies).                                                                                       |
| `AllPlayers`           | `IEnumerable<MonsterInfo>`  | Returns all players in the area, including yourself.                                                                                   |
| `PlayerByName(name)`   | `MonsterInfo`               | Returns a specific player by their character name.                                                                                     |
| `PartyLeader`          | `MonsterInfo`               | Returns the current party leader.                                                                                                      |
| `MiscellaneousObjects` | `IEnumerable<EntityInfo>`   | Returns miscellaneous objects like breach hands, ritual altars, etc.                                                                  |
| `NoneEntities`         | `IEnumerable<EntityInfo>`   | Returns entities with an `EntityType` of `None`. This can sometimes include interactable objects or invisible triggers.                |
| `IngameIcons`          | `IEnumerable<EntityInfo>`   | Returns entities that are displayed on the minimap, such as quest markers or master icons.                                             |
| `MiniMonoliths`        | `IEnumerable<EntityInfo>`   | Returns Legion monolith entities.                                                                                                      |
| `Effects`              | `IEnumerable<EntityInfo>`   | Returns visual effects entities.                                                                                                       |
| `Portals`              | `IEnumerable<EntityInfo>`   | Returns portal entities.                                                                                                               |
| `PortalExists(distance)` | `bool`                      | Checks if a portal exists within the given distance.                                                                                   |

### Internal State & Functions

| Method/Property        | Return Type | Description                                                                                             |
| ---------------------- | ----------- | ------------------------------------------------------------------------------------------------------- |
| `IsKeyPressed(key)`    | `bool`      | Checks if a specific key is currently held down (e.g., `IsKeyPressed(Keys.LeftShift)`).                 |
| `SinceLastActivation(seconds)` | `bool`      | `true` if the current rule has not been activated for at least the specified number of `seconds`. Useful for cooldowns. |
| `IsFlagSet(name)`      | `bool`      | Checks if a custom flag with the given `name` is set for the current rule group.                        |
| `GetNumberValue(name)` | `float`     | Gets the value of a custom number with the given `name` for the current rule group.                     |
| `GetTimerValue(name)`  | `float`     | Gets the elapsed time in seconds of a custom timer with the given `name`.                               |
| `IsTimerRunning(name)` | `bool`      | Checks if a custom timer with the given `name` is currently running.                                    |
| `Random(min, max)`     | `int`     | Returns a random integer between `min` (inclusive) and `max` (exclusive).                               |

## 5. Data Types Deep Dive

### `VitalsInfo`

| Property | Type    | Description                               |
| -------- | ------- | ----------------------------------------- |
| `HP`     | `Vital` | Player's Health (Current, Max, Percent).    |
| `Mana`   | `Vital` | Player's Mana (Current, Max, Percent).      |
| `ES`     | `Vital` | Player's Energy Shield (Current, Max, Percent). |

### `FlasksInfo`

The `Flasks` property on the main `RuleState` object gives you access to your flasks. You can access them by index or by their specific property name.

| Property    | Type        | Description                               |
| ----------- | ----------- | ----------------------------------------- |
| `this[int]` | `FlaskInfo` | Access a flask by its index (0-4).        |
| `Flask1`    | `FlaskInfo` | Access the flask in slot 1.               |
| `Flask2`    | `FlaskInfo` | Access the flask in slot 2.               |
| `Flask3`    | `FlaskInfo` | Access the flask in slot 3.               |
| `Flask4`    | `FlaskInfo` | Access the flask in slot 4.               |
| `Flask5`    | `FlaskInfo` | Access the flask in slot 5.               |

**Example:** `Flasks[0].CanBeUsed` or `Flasks.Flask1.CanBeUsed`

### `FlaskInfo`

| Property      | Type     | Description                                                              |
| ------------- | -------- | ------------------------------------------------------------------------ |
| `Active`      | `bool`   | `true` if the flask effect is currently active.                          |
| `CanBeUsed`   | `bool`   | `true` if the flask has enough charges and is not on cooldown.           |
| `Charges`     | `int`    | Current number of charges.                                               |
| `MaxCharges`  | `int`    | Maximum number of charges.                                               |
| `ChargesPerUse` | `int`  | The number of charges consumed on use.                                   |
| `CanBeUsedIn` | `float`  | Seconds until a Tincture can be used again. Returns 0 for normal flasks. |
| `Name`        | `string` | The unique name if available, otherwise the base name of the flask.      |
| `BaseName`    | `string` | The base name of the flask (e.g., "Granite Flask").                      |
| `ClassName`   | `string` | The internal class name of the flask item.                               |

### `BuffDictionary`

-   **`Has(string buffName)`**: Checks if a buff is active.
-   **`this[string buffName]`**: Accesses a `StatusEffect` object for a buff by its name.
-   **`AllBuffs`**: A `List<StatusEffect>` that you can iterate over (e.g., with a `.Where(...)` clause) to find buffs with specific properties.

### `StatusEffect`

| Property        | Type     | Description                                                              |
| --------------- | -------- | ------------------------------------------------------------------------ |
| `Name`          | `string` | The internal name of the buff.                                           |
| `DisplayName`   | `string` | The user-visible name of the buff.                                       |
| `Exists`        | `bool`   | `true` if the buff is active.                                            |
| `TimeLeft`      | `double` | Remaining duration of the buff in seconds.                               |
| `TotalTime`     | `double` | Total duration of the buff in seconds.                                   |
| `PercentTimeLeft` | `double` | Remaining duration as a percentage.                                    |
| `Charges`       | `int`    | Number of stacks or charges the buff has.                                |
| `Skill`         | `SkillInfo`| The skill that applied this buff.                                        |

### `SkillDictionary`

-   **`Has(string skillName)`**: Checks if you have a skill.
-   **`this[string skillName]`**: Accesses a `SkillInfo` object by its name.
-   **`Current`**: Returns the `SkillInfo` for the skill currently being used.
-   **`AllSkills`**: A `List<SkillInfo>` that you can iterate over to find skills with specific properties.
-   **`BySlotIndex(int slotIndex)`**: Returns the `SkillInfo` for the skill in the specified slot index. Returns an empty `SkillInfo` if no skill is found in that slot.

**Example Usage:**

```csharp
// Streamlined mana flask logic checking if skill in slot 3 can be used
SinceLastActivation(1) &&
!Buffs.Has("grace_period") &&
(Flasks[4].Name.Contains("Mana Flask") && !Skills.BySlotIndex(3).CanBeUsed) &&
Flasks[0].CanBeUsed
```

### `SkillInfo`

| Property        | Type                  | Description                                                              |
| --------------- | --------------------- | ------------------------------------------------------------------------ |
| `Name`          | `string`              | The name of the skill.                                                   |
| `Exists`        | `bool`                | `true` if the skill is known.                                            |
| `CanBeUsed`     | `bool`                | `true` if the skill is usable (not on cooldown, enough resources).       |
| `IsUsing`       | `bool`                | `true` if the skill is currently being used.                             |
| `UseStage`      | `int`                 | The current stage of the skill usage.                                    |
| `ManaCost`      | `int`                 | The mana cost of the skill.                                              |
| `LifeCost`      | `int`                 | The life cost of the skill.                                              |
| `EsCost`        | `int`                 | The energy shield cost of the skill.                                     |
| `MaxUses`       | `int`                 | The maximum number of times the skill can be used (charges).             |
| `RemainingUses` | `int`                 | Number of charges remaining.                                             |
| `MaxCooldown`   | `float`               | The maximum cooldown time in seconds.                                    |
| `Cooldowns`     | `List<float>`         | A list of remaining cooldowns for skills with multiple charges.          |
| `CastTime`      | `float`               | The cast time of the skill in seconds.                                   |
| `DeployedEntities`| `List<MonsterInfo>` | A list of entities created by this skill (e.g., totems, minions, brands). |
| `Stats`         | `StatDictionary`      | A dictionary of the skill's stats. See [`StatDictionary`](README.md:249) for usage. |

### `MonsterInfo` / `EntityInfo`

| Property           | Type               | Description                                                              |
| ------------------ | ------------------ | ------------------------------------------------------------------------ |
| `Id`               | `uint`             | The entity's unique identifier.                                          |
| `Path`             | `string`           | The entity's path (e.g., "Metadata/Monsters/Zombies/ZombieStandard").    |
| `Metadata`         | `string`           | The entity's metadata path.                                              |
| `IsAlive`          | `bool`             | `true` if the entity is alive.                                           |
| `Distance`         | `float`            | Distance from the player.                                                |
| `Position`         | `Vector3`          | The entity's 3D world position.                                          |
| `GridPosition`     | `Vector2`          | The entity's 2D grid position.                                           |
| `Position2D`       | `Vector2`          | The entity's 2D world position (X and Y).                                |
| `VectorToPlayer`   | `Vector2`          | The vector from the entity to the player.                                |
| `Rarity`           | `MonsterRarity`    | The rarity of the monster.                                               |
| `Vitals`           | `VitalsInfo`       | The monster's health.                                                    |
| `Buffs`            | `BuffDictionary`   | The monster's buffs.                                                     |
| `Stats`            | `StatDictionary`   | A dictionary of the monster's stats. See `StatDictionary` for usage.     |
| `Actor`            | `ActorInfo`        | Provides information about the entity's animations and actions.          |
| `IsTargetable`     | `bool`             | `true` if the entity can be targeted.                                    |
| `IsTargeted`       | `bool`             | `true` if the entity is currently targeted by the player.                |
| `IsUsingAbility`   | `bool`             | `true` if the entity is currently using an ability.                      |
| `IsInvincible`     | `bool`             | `true` if the monster has the "Cannot be Damaged" stat.                  |
| `PlayerName`       | `string`           | The character name if the entity is a player.                            |
| `DistanceToCursor` | `float`            | The distance from the entity to the world position of the mouse cursor.  |

### `ActorInfo`

| Property              | Type     | Description                                               |
| --------------------- | -------- | --------------------------------------------------------- |
| `Action`              | `string` | The name of the entity's current action (e.g., "Attack").   |
| `Animation`           | `string` | The name of the entity's current animation.               |
| `CurrentAnimationId`    | `int`    | The numeric ID of the current animation.                  |
| `CurrentAnimationStage` | `int`    | The stage of the current animation.                       |
| `AnimationProgress`   | `float`  | The progress of the current animation (from 0.0 to 1.0).  |

### `StatDictionary` and `Stat`

The `Stats` property on an entity allows you to check for the presence and value of specific game stats.

-   **`Stats[GameStat stat]`**: Accesses a `Stat` object. The `GameStat` enum contains all possible stats.
-   **`Stat.Exists`**: A boolean indicating if the entity has the stat.
-   **`Stat.Value`**: The integer value of the stat.

**Example:**

```csharp
// Check if a monster cannot be damaged
Monsters(50).Any(m => m.Stats[GameStat.CannotBeDamaged].Exists)

// Check if the player has the "CannotBeFrozen" stat
Player.Stats[GameStat.CannotBeFrozen].Exists
```

### `MonsterRarity` Enum

This is a flag enum, which means you can combine values using the `|` operator (e.g., `MonsterRarity.Rare | MonsterRarity.Unique`).

| Value         | Description                               |
| ------------- | ----------------------------------------- |
| `Normal`      | Normal (white) monsters.                  |
| `Magic`       | Magic (blue) monsters.                    |
| `Rare`        | Rare (yellow) monsters.                   |
| `Unique`      | Unique (brown) monsters.                  |
| `Any`         | Any monster rarity.                       |
| `AtLeastRare` | Combines `Rare` and `Unique`.             |

## 6. Available Actions (`SideEffects`)

These are the actions you can perform when a rule's condition is met. You create them using `new ActionName(...)`.

### Key Presses

-   **`PressKeySideEffect(HotkeyNodeValue Key)`**: Presses a key. Also has a constructor `PressKeySideEffect(Keys key)`.
    -   Example: `new PressKeySideEffect(Keys.D1)`
-   **`StartKeyHoldSideEffect(HotkeyNodeValue Key)`**: Starts holding a key down. Also has a constructor `StartKeyHoldSideEffect(Keys key)`.
-   **`ReleaseKeyHoldSideEffect(HotkeyNodeValue Key)`**: Releases a held key. Also has a constructor `ReleaseKeyHoldSideEffect(Keys key)`.

### State Management (Timers, Flags, Numbers)

These are useful for creating complex, stateful logic within a rule group.

-   **`StartTimerSideEffect(string Id)`**: Starts a timer with the given name.
-   **`StopTimerSideEffect(string Id)`**: Stops a running timer.
-   **`RestartTimerSideEffect(string Id)`**: Resets a timer and starts it again.
-   **`ResetTimerSideEffect(string Id)`**: Deletes a timer.
-   **`SetFlagSideEffect(string Id)`**: Sets a boolean flag to `true`.
-   **`ResetFlagSideEffect(string Id)`**: Deletes a flag.
-   **`SetNumberSideEffect(string Id, float value)`**: Sets a named number to a specific value.
-   **`ResetNumberSideEffect(string Id)`**: Deletes a number.

### Display

-   **`DisplayTextSideEffect(string Text, Vector2 Position, string Color)`**: Displays text on the screen.
    -   `position` is a screen coordinate. `color` is a string like "White" or "Red".
-   **`DisplayGraphicSideEffect(string GraphicFilePath, Vector2 Position, Vector2 Size, string ColorTint)`**: Draws an image from the `textures/ReAgent` folder.
-   **`ProgressBarSideEffect(string Text, Vector2 Position, Vector2 Size, float Fraction, string Color, string BackgroundColor, string TextColor)`**: Displays a progress bar.

### Advanced

-   **`DelayedSideEffect(double Delay, Func<IReadOnlyList<ISideEffect>> SideEffects)`**: Executes one or more actions after a `delay` in seconds.
    -   Example: `new DelayedSideEffect(0.5, () => new[] { new PressKeySideEffect(Keys.D2) })`
-   **`DisconnectSideEffect()`**: Disconnects from the server (TCP disconnect).
-   **`PluginBridgeSideEffect<T>(string MethodName, Action<T> InvokeFunctionAction)`**: Calls a method in another plugin. (where `T` is a `Delegate`)

## 7. `CustomAilments.json`

This file allows you to group multiple buff names into a single, custom ailment category. You can then check for this category in your rules using `Ailments.Contains("MyAilmentName")`.

**Example:**

```json
{
  "MyCurseGroup": [
    "curse_temporal_chains",
    "curse_enfeeble"
  ]
}
```

In a rule, you could then use: `Ailments.Contains("MyCurseGroup")` to check if the player is affected by either Temporal Chains or Enfeeble.

### Pre-defined Ailment Groups

The default `CustomAilments.json` file comes with a number of pre-defined groups for common ailments:

| Group Name             | Description                                                              |
| ---------------------- | ------------------------------------------------------------------------ |
| `Bleeding`             | Groups standard bleeding and puncture effects.                           |
| `Bleeding Or Corruption` | A broader category that includes both bleeding and Corrupted Blood.      |
| `Burning`              | Includes various sources of fire damage over time (e.g., ignite, RF).    |
| `Chilled`              | Groups different sources of the chill effect.                            |
| `Corruption`           | Specifically for Corrupted Blood debuffs.                                |
| `Cursed`               | A large group containing all player-affecting curses.                    |
| `Exposed`              | For elemental exposure debuffs that lower resistances.                   |
| `Frozen`               | For freeze effects.                                                      |
| `Frozen Or Chilled`    | Combines all sources of both freeze and chill.                           |
| `Poisoned`             | Groups various poison and caustic ground effects.                        |
| `Shocked`              | For shock effects from different sources.                                |
| `Unable To Recover`    | For dangerous debuffs that prevent life/ES recovery (e.g., from Maven).  |

## 8. For Developers: `SideEffectApplicationResult`

When a `SideEffect` is applied, it returns a `SideEffectApplicationResult`. While you cannot use this in your rules, it may appear in debug logs and is useful for understanding how the rule engine is processing your actions.

| Value              | Description                                                                                             |
| ------------------ | ------------------------------------------------------------------------------------------------------- |
| `UnableToApply`    | The side effect could not be applied. For example, trying to press a key when another key press is already queued. |
| `AppliedUnique`    | The side effect was applied successfully and was the first of its kind (e.g., starting a timer that was not already running). |
| `AppliedDuplicate` | The side effect was already in the desired state (e.g., trying to start a timer that is already running), so no action was taken. |
