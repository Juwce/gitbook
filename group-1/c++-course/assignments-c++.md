# Assignments (C++)

#### Assignment 1 - Project Setup

* Github setup
* Character jump setup with input bindings
* Explosive barrel C++ (using Unreal's gameplay statics library)

#### Assignment 2 - Projectile Abilities

* Magic projectile targeting (line trace to player reticle)
  * Spawn location (hand position)
  * Impact location (line trace result)
  * (if nothing was hit, use 'trace end' vector as desired target)
  * (line trace against multiple object types: WorldDynamic, WorldStatic)
* Blackhole Projectile
  * Done with blueprints, extending a base projectile C++ class that handles targeting, properties
* Dash/Teleport Projectile
  * Projectile class spawned via input key
  * ParticleComponent
  * Explodes after 0.2 seconds
  * Plays particle effect at point of detonation
  * Waits 0.2 seconds before teleporting PlayerCharacter
  * On hit with world: immediately stop movement and execute the same behavior

#### Assignment 3 - Attributes and Powerups

* Magic projectile audio and visual polish (C++)
* Player character hitflash (C++)
  * material + material parameters
* AttributeComponent (C++)
  * Health / MaxHealth / change health / health changed delegate
* Health UI (UMG)
  * implemented floating damage text
* Powerup Actors (C++)
  * React to Interact()
  * Deactivate for 10 seconds upon triggered (hide + disable collision)
  * Ignores interacting pawn if already at max health
  * Implemented base powerup class

#### Assignment 4 - Behavior Trees

* Behavior: Hide and heal when at low health:
  * (C++) BT Service - Check if 'low health'
  * Environment Query - Find hidden position away from player, close to AI
  * (C++) BT Task - "Heal" back to max hitpoints
  * 60 second cooldown during which AI behaves normally, even if at low health

#### Assignment 5 - Credit System

* Credits system to gain and spend (C++)
  * Credits remain even if player dies
  * Killing minions grants credits
  * Health powerup costs credits to interact
  * Coin powerup gives credits when interacting
* Randomly spawn coin and health potions throughout level on start (C++)

#### Assignment 6 - New Attributes, Buffs, Powerups

* New 'Rage' Attribute (C++)
  * Player receiving damage adds to rage value
  * Blackhole ability costs rage to use
  * Rage UI on screen
* 'Thorns' Buff (C++)
  * Deals fraction of damage received back to attacker
  * Doesn't reflect damage caused to self, or damage received from thorns ability
* 'Player Spotted' widget when AI minion sees player (C++)
* Ability granting Powerup (C++)
  * Interacting with powerup grants dash ability on interact
  * Only interactable if user doesn't have ability yet

#### Assignment 7 - Network Replicated Attributes & Powerups

* Replicate player credits
* Replicate powerups
  * Synchronize visibility & collision
  * Only server/authority can change credits/health
* Replicate player rage attribute
  * Only server/authority can change credits/health
* Replicate 'Player Spotted' widget
  * When AI (running on server) sees a player, show the player spotted widget above the AI on all clients
