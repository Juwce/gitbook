# Key Learnings (C++)

**Unreal's physics system basics (C++)**

* Collision channels
* Collision filtering (ignore/overlap/block)
* Binding functions to physics events (overlap, hit, etc.)

**C++ Gameplay Interface**

* `GameplayInterface` with an `Interact()` method

**Debugging tools**

* C++
  * Logging
  * Breakpoint (& disabling compiler optimizations)
  * Console vars
  * Asserts
* Unreal:
  * Printing on screen messages
  * Size Map (view hard references of any asset and how much memory they consume)
  * Reference Viewer (view parent and child references of any asset)
  * 'Stat' commands (call count / timing / etc. info, neatly accessible

**C++ Attribute Framework**

* `AttributeComponent` class
* Give any actor attributes (health, stamina, etc.)
* Delegates for broadcasting attribute change events

**Dynamic Materials (Shader Graph & C++)**

* Passed gameplay parameters to materials (shaders) during gameplay
* Implemented 'Hit Flash' effect that triggers on player damaged

**AI (C++ and Blueprints)**

* Implemented AICharacter and AIController
* Behavior Trees (C++ and Blueprints)
  * Implemented AI character with roaming, targeting, attacking, and hiding behaviors
    * Defined behaviors and services in C++
      * Service: find attack target within range
      * Service: check health
      * Task: Heal self
      * Task: Perform ranged attack
    * Gained an understanding of the following Behavior Tree and AI basics:
      * Tasks
      * Services
      * Decorators
      * Blackboard (Unreal's behavior tree "database" for quick access to properties from the BT)
      * Environmental Query System
      * PawnSensing
      * NavMeshes

**UI**

* Manage UI lifecycle in C++
* Pass data from application to UI from C++
* In-world widgets (floating damage text, health bars, etc.)
* Player HUD widgets (health bar, game clock, etc.)

**Unreal's Gameplay Framework**

* Built understanding of lifetime of Unreal's core framework - GameMode, Controller, Character, GameState, PlayerState, etc.
* Managing UI, etc. on player death and respawn

**Action System (C++)**

* Key goals:
  * Code separation
    * Avoid monolithic Character class containing all ability code and VFX
    * Easier to maintain single-purpose classes
    * Simplify network replication
  * Also...
    * Avoid loading entire game into memory at once via 'hard-references'
    * Manage large amount of state and labels via GameplayTags
    * Actions can be performed by any actor in the world, not just the player

Action System Classes:

* `Action` class
  * `StartAction()`
  * `StopAction()`
* `ActionComponent` class
  * `Array<Action> Actions`
  * `AddAction()`
  * `StartAction()`
  * `StopAction()`
* `ActionEffect` class (extends `Action`)
  * `ExecutePeriodicEffect()`
  * `duration`
  * `period`

**Networked Game Logic (C++)**

* Used Unreal's Network Replication system (C++)
  * Property replication
  * RPCs (reliable & unreliable)
    * Server RPC
    * Client RPC
    * Multicast RPC (sends to all sessions)
* Replicated the following gameplay features (with C++):
  * Player, AI, and projectile spawning and movement (handled by Unreal by default)
  * Player attributes (health, rage, credits)
  * Powerups (potions, coins, etc.)
  * Interaction (open chests, pickup powerups, etc.)
  * Action system (all abilities including attacks, sprinting, buffs, 'burning' effect, etc.)
* Enforced server authority (C++)
  * Only the server can:
    * Modify player attributes
    * Interact with interactables
    * Affect AI decisions (e.g. picking attack target)
    * Start and stop actions
  * Client limited to cosmetic actions

**Misc:**

* Save Game State: Iterate world and game state and serialize into a save file
* Menus: Implemented main & pause menu with functionality to host and join a game
* Blend Spaces: Implemented blend space for sprinting animation (via Unreal's UI) based on player speed
* Data Tables and Assets: Moved monster spawning configuration into data tables and assets, separating configuration data from assets and code
* Async loading of assets: Created soft references to several larger assets and added C++ logic to asynchronously load those assets at runtime, only when needed
* Packaging game

