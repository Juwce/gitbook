---
description: Minion AI in this project is powered by Behavior Trees
cover: ../../../.gitbook/assets/Untitled design.png
coverY: -46.33016265337739
---

# Minion AI (Behavior Trees)

## Code and Assets

Code and assets for this project's AI implementation can be found on GitHub. Individual files will also be linked to in the sections below.

* AI Classes: [Private / .cpp](https://github.com/Juwce/ActionRoguelike/tree/main/Source/ActionRoguelike/Private/AI) || [Public / .h](https://github.com/Juwce/ActionRoguelike/tree/main/Source/ActionRoguelike/Public/AI)
* Behavior Tree, EQS, another other Unreal AI assets: [AI Assets](https://github.com/Juwce/ActionRoguelike/tree/main/Content/ActionRoguelike/AI)

## Setup:

Behavior Trees run on an AI Controller, which assumes control of an AI Character or Pawn. Set up both as follows:

1\. Extend the AAIController class, adding a BehaviorTree reference:

{% code title="ATAIController.h" %}
```cpp
UPROPERTY(EditDefaultsOnly, Category = "AI")
UBehaviorTree* BehaviorTree;
```
{% endcode %}

2\. Run the Behavior Tree in your new class's BeginPlay()

{% code title="ATAIController.cpp" %}
```
if (ensureMsgf(BehaviorTree, TEXT("BehaviorTree is nullptr! Please assign "
                                  "BehaviorTree in your AI controller.")))
{
	RunBehaviorTree(BehaviorTree);
}
```
{% endcode %}

3\. Create a Blueprint with your new AI Controller class and assign your behavior tree of choice to it

4\. Assign your new AI Controller Class to your AI Pawn.

5\. If you plan on spawning your AI minion and want AI to active immediately, set the following in your AI Character class constructor:

{% code title="ATAICharacter.cpp" %}
```cpp
// Unless this is set, AI is not activated by default on spawned AI Characters
AutoPossessAI = EAutoPossessAI::PlacedInWorldOrSpawned;
```
{% endcode %}

## Minion Behavior Tree

![The full behavior tree for the ranged minion AI (broken down below). Services are Green || Decorators are Blue || Composites are Gray || Tasks are Purple](<../../../.gitbook/assets/bt ui.png>)

<details>

<summary>Click to Expand... <strong>Unreal Behavior Trees Terminology (Quick Reference)</strong></summary>

[Differences in UE4 Behavior Trees](https://docs.unrealengine.com/4.27/en-US/InteractiveExperiences/ArtificialIntelligence/BehaviorTrees/BehaviorTreesOverview/#differencesinue4behaviortrees) (compared to traditional behavior trees)

* **Blackboard** - Key:value store for sharing data between behaviors in the tree (optimized for access and performance).
* Nodes:
  * <mark style="color:green;">Service Nodes</mark> - Execute at a defined frequency as long as their branch is being executed. Often used to make checks and update the Blackboard.
  * <mark style="color:blue;">Decorator Nodes</mark> - Attach to other nodes and make decisions on whether or not a branch or node in the tree can execute. Decorator nodes are able to change the flow of a tree by aborting lower priority executing nodes and executing their branch immediately (for example, to have an AI immediately stop whatever it was doing to flee when its health drops low, you might put a decorator node that monitors its health status value in the blackboard and aborts other running nodes when it's set to "Low").
  * <mark style="color:purple;">Task Nodes</mark> - Actionable things to do. Task nodes perform some behavior and don't have an output connection.
  * **Composite Nodes** - The root of a branch that defines how the branch is executed (in sequence, parallel, or select one). Composite nodes can have decorators applied to them to control entry into the branch, and services that will only be active if the children of the composite are being executed.

</details>

## Behavior: Move Near Target and Attack

![AI behavior to move near target and attack it (left) and the behavior tree powering it (right).](<../../../.gitbook/assets/bt demo move and attack.gif>)

### Implementation:

![Services are Green || Decorators are Blue || Selectors are Gray || Tasks are Purple](<../../../.gitbook/assets/image (4).png>)

**Attack Target if within Attack Range**

If the target is within attack range, the AI will attack the target three times and then go on a three second cooldown. Note the service running the attack range check is not pictured here as that runs as a service at the root of the behavior tree (every half second or so).

**Move Closer to Target Actor**

If the target is not within the attack range, the AI minion will find a random location on the navmesh nearby and with line of site to the target actor and move to it. If there is no target actor (a target has not been spotted by the AI yet), the AI "cheats" and moves towards the controlling player (or the first player to join the match in multiplayer).&#x20;

### C++ Classes:

Unreal's behavior trees provide a plethora of [built-in nodes](https://docs.unrealengine.com/4.27/en-US/InteractiveExperiences/ArtificialIntelligence/BehaviorTrees/BehaviorTreeNodeReference/) that can be used to run basic behaviors. This AI leverages these built-ins for basic behaviors, but for custom ones I needed to write my own functionality in C++. I wrote the following nodes in C++:

#### BT Services

* CheckAttackRange <mark style="background-color:green;">(Service)</mark>
  * Continually monitors if the controlled AI actor has line of site to the Target Actor, storing the result in the Blackboard key for easy access by the rest of the BT.
  * See implementation on GitHub: [TBTService\_CheckAttackRange.cpp](https://github.com/Juwce/ActionRoguelike/blob/main/Source/ActionRoguelike/Private/AI/TBTService\_CheckAttackRange.cpp)

**BT Tasks**

* RangedAttack <mark style="background-color:purple;">(Task)</mark>
  * Fires a projectile at the target actor from the controlled AI character's muzzle.&#x20;
  * A configurable deviation is applied to tune AI difficulty (higher deviations make the attack more inaccurate)
  * See implementation on GitHub: [TBTTask\_RangedAttack.cpp](https://github.com/Juwce/ActionRoguelike/blob/main/Source/ActionRoguelike/Private/AI/TBTTask\_RangedAttack.cpp)

### Unreal Built-Ins:

I configured and used the following built-in Unreal nodes:

#### **BT Tasks**

* Finding a random location on the navmesh closer to the target actor and line of site checks are performed by Unreal's [Environmental Query System](https://docs.unrealengine.com/4.27/en-US/InteractiveExperiences/ArtificialIntelligence/EQS/), configured with Blueprints (Query in a donut shape around the player, projecting locations onto the navmesh and checking for line of site. Assign each valid location a weight based on distance to the player \[closer is better] and pick the one with the highest score).
* The movement along the navmesh to the target location also used Unreal's built-in "Move To" task.

## Behavior: Hide and Heal

![Upon dropping to low health, the AI will stop whatever it is doing immediately to go hide and heal.](<../../../.gitbook/assets/bt demo hide and heal.gif>)

### Implementation:

The hide and heal sequence is set up to abort any lower priority executing nodes whenever the "LowHealth" key is set.

![Services are Green || Decorators are Blue || Selectors are Gray || Tasks are Purple](<../../../.gitbook/assets/image (3).png>)

**Stop all other actions when health drops low:**

The decorator "Health Low" at the top of this behavior's branch is set to Abort Lower Priority whenever the "HealthLow" key changes. Since this is the highest priority node in the tree, this means the AI will always immediately stop whatever else it was doing to go hide and heal when its health drops low. A custom C++ service constantly monitor's the AI's health, setting the "HealthLow" BlackBoard key when it drops.

**Find a hiding spot and move to it:**

This leverages Unreal's Environmental Query System to find a spot on the navmesh away from the attack target and without line of site. If it can't find a hidden location, it picks the furthest possible spot away from the target actor, but within a maximum radius so the AI does not run away too far.

**Heal Self and Cooldown:**

Finally, once it reaches its determined safe spot, the AI heals itself via a custom C++ task. This ability goes on cooldown for 60 seconds (meaning the AI will not perform this sequence nor abort other lower priority nodes the next time the AI's health drops low as long as this cooldown is active).

### C++ Classes:

I wrote the following nodes in C++:

#### BT Services

* CheckHealth <mark style="background-color:green;">(Service)</mark>
  * Continually monitors the health of the controlled AI actor (via its AttributeComponent), storing the result in a Blackboard key for easy access by the rest of the BT.
  * See implementation on GitHub: [TBTService\_CheckHealth.cpp](https://github.com/Juwce/ActionRoguelike/blob/main/Source/ActionRoguelike/Private/AI/TBTService\_CheckHealth.cpp)

**BT Tasks**

* HealSelf <mark style="background-color:purple;">(Task)</mark>
  * Heals the controlled AI actor
  * Configurable `HealingType`, value, and duration (able to perform a heal over time, split into multiple 'ticks')
    * Healing Types:
      * `EHealingType::HealthPoints` - heal a fixed number of health points
      * `EHealingType::PercentOfHealthMax` - heal a set percent of health points
  * See implementation on GitHub: [TBTTask\_HealSelf.cpp](https://github.com/Juwce/ActionRoguelike/blob/main/Source/ActionRoguelike/Private/AI/TBTTask\_HealSelf.cpp)

### Unreal Built-Ins:

I configured and used the following built-in Unreal nodes:

**BT Tasks**

* Find location hidden from players - leverages Unreal's EQS editor to find a location on the navmesh within a radius of the target actor but without line of site.
* The hide and heal behavior also leverages Unreal's built-in "Observer Aborts" feature to stop the execution of lower priority nodes (e.g. the move and attack behavior) immediately whenever the AI's health drops low. I configured the "Health Low?" decorator to monitor the "LowHealth" blackboard key set by the CheckHealth service, and immediately start executing the hide and heal one.
