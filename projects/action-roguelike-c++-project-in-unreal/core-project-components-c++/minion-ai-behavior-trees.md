---
description: Minion AI in this project is powered by Behavior Trees
cover: ../../../.gitbook/assets/Untitled design.png
coverY: -46.33016265337739
---

# Minion AI (Behavior Trees)

## Classes and Assets

C++ classes and Unreal assets for this project's AI can be found on GitHub:

* [AI Classes - github](https://github.com/Juwce/ActionRoguelike/tree/main/Source/ActionRoguelike/Private/AI) ([headers](https://github.com/Juwce/ActionRoguelike/tree/main/Source/ActionRoguelike/Private/AI))
  * AI Character classes: `TAICharacter`, `TAIController`
  * Behavior Tree Node Classes: `TBTService_CheckAttackRange`, `TBTService_CheckHealth`, `TBTTask_HealSelf`, `TBTTask_RangeAttack`
* [AI Assets - github](https://github.com/Juwce/ActionRoguelike/tree/main/Content/ActionRoguelike/AI)

### **Jump To Section:**

* [AI Setup](minion-ai-behavior-trees.md#ai-setup)
* [Minion Behavior Tree](minion-ai-behavior-trees.md#minion-behavior-tree)
* [Behavior: Move Near Target and Attack](minion-ai-behavior-trees.md#behavior-move-near-target-and-attack)
* [Behavior: Hide and Heal](minion-ai-behavior-trees.md#behavior-hide-and-heal)
* [Unreal Built-In Nodes Used](minion-ai-behavior-trees.md#unreal-built-ins-1)

## AI Setup

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

4\. Assign your new AI Controller Class to your AI Character or Pawn.

5\. If you plan on spawning your AI minion and want AI to active immediately, set the following in your AI Character class constructor:

{% code title="ATAICharacter.cpp" %}
```cpp
// Unless this is set, AI is not activated by default on spawned AI Characters
AutoPossessAI = EAutoPossessAI::PlacedInWorldOrSpawned;
```
{% endcode %}

## Minion Behavior Tree

![The full behavior tree for the ranged minion AI (broken down below). Services are Green || Decorators are Blue || Composites are Gray || Tasks are Purple](<../../../.gitbook/assets/bt ui.png>)

#### **Unreal Behavior Tree terminology overview:**

<details>

<summary>Click to Expand...</summary>

* [Differences in UE4 Behavior Trees](https://docs.unrealengine.com/4.27/en-US/InteractiveExperiences/ArtificialIntelligence/BehaviorTrees/BehaviorTreesOverview/#differencesinue4behaviortrees) (compared to traditional behavior trees)
* Terminology:
  * **Blackboard** - Key:value store for sharing data between behaviors in the tree (optimized for access and performance).
  * <mark style="color:green;">Service Nodes</mark> - Execute at a defined frequency as long as their branch is being executed. Often used to make checks and update the Blackboard.
  * <mark style="color:blue;">Decorator Nodes</mark> - Attach to other nodes and make decisions on whether or not a branch or node in the tree can execute. Decorator nodes are able to change the flow of a tree by aborting lower priority executing nodes and executing their branch immediately (for example, to have an AI immediately stop whatever it was doing to flee when its health drops low, you might put a decorator node that monitors its health status value in the blackboard and aborts other running nodes when it's set to "Low").
  * <mark style="color:purple;">Task Nodes</mark> - Actionable things to do. Task nodes perform some behavior and don't have an output connection.
  * **Composite Nodes** - The root of a branch that defines how the branch is executed (in sequence, parallel, or select one). Composite nodes can have decorators applied to them to control entry into the branch, and services that will only be active if the children of the composite are being executed.

</details>

## Behavior: Move Near Target and Attack

![AI moves near target and attack it (left) and the behavior tree powering it (right).](<../../../.gitbook/assets/bt demo move and attack.gif>)

### Custom nodes:

* [TBTService\_CheckAttackRange.cpp](https://github.com/Juwce/ActionRoguelike/blob/main/Source/ActionRoguelike/Private/AI/TBTService\_CheckAttackRange.cpp)
* [TBTTask\_RangedAttack.cpp](https://github.com/Juwce/ActionRoguelike/blob/main/Source/ActionRoguelike/Private/AI/TBTTask\_RangedAttack.cpp)

### Behavior tree branch:

![Services are Green || Decorators are Blue || Selectors are Gray || Tasks are Purple](<../../../.gitbook/assets/image (4).png>)

**Sequence: Attack Target if within Attack Range**

[TBTService\_CheckAttackRange.cpp](https://github.com/Juwce/ActionRoguelike/blob/main/Source/ActionRoguelike/Private/AI/TBTService\_CheckAttackRange.cpp) (not pictured) constantly monitors the distance between the AI's controlled minion and the target, setting the `WithinAttackRange` Blackboard key accordingly. If the target is within attack range when execution flow is passed to this branch, the AI executes [TBTTask\_RangedAttack.cpp](https://github.com/Juwce/ActionRoguelike/blob/main/Source/ActionRoguelike/Private/AI/TBTTask\_RangedAttack.cpp), firing a projectile from the AI minion's muzzle at the target three times. This attack has a configurable deviation that can be used to tune AI difficulty (higher deviations make the attack more inaccurate). The attack is then put on cooldown.

**Sequence: Move Closer to Target Actor**

If the target actor is not `WithinAttackRange` or the attack is on cooldown, the AI will run an [EQS](https://docs.unrealengine.com/4.27/en-US/InteractiveExperiences/ArtificialIntelligence/EQS/) query to find a random location on the navmesh nearby and with line of site to the target actor, and [move to](https://docs.unrealengine.com/4.26/en-US/BlueprintAPI/AI/Navigation/MovetoLocation/) that location. If there is no target actor set (a target has not been spotted by the AI yet), the AI "cheats" and picks Player 0 (first player to join the session) as its target instead.

## Behavior: Hide and Heal

![Upon dropping to low health, the AI will stop whatever it is doing immediately to go hide and heal.](<../../../.gitbook/assets/bt demo hide and heal.gif>)

### Custom nodes:

* [TBTService\_CheckHealth.cpp](https://github.com/Juwce/ActionRoguelike/blob/main/Source/ActionRoguelike/Private/AI/TBTService\_CheckHealth.cpp)
* [TBTTask\_HealSelf.cpp](https://github.com/Juwce/ActionRoguelike/blob/main/Source/ActionRoguelike/Private/AI/TBTTask\_HealSelf.cpp)

### Behavior tree branch:

![Services are Green || Decorators are Blue || Selectors are Gray || Tasks are Purple](<../../../.gitbook/assets/image (3).png>)

**Observer Aborts: Stop all other actions when health drops low:**

[TBTService\_CheckHealth.cpp](https://github.com/Juwce/ActionRoguelike/blob/main/Source/ActionRoguelike/Private/AI/TBTService\_CheckHealth.cpp) (not pictured) monitors minion's health, updating the `HealthLow` Blackboard key when it drops below a configurable threshold. The AI monitors the `HealthLow` key for changes (via the "HealthLow?" decorator). When it observes a change, this decorator [immediately aborts](https://docs.unrealengine.com/4.27/en-US/InteractiveExperiences/ArtificialIntelligence/BehaviorTrees/BehaviorTreeNodeReference/BehaviorTreeNodeReferenceDecorators/) lower priority executing nodes and starts executing the hide and heal sequence.

**Sequence: Find a hiding spot and move to it:**

The AI will then find a safe spot on the navmesh via Unreal's EQS and move to it. A "safe spot" is determined via EQS's [scoring system](https://docs.unrealengine.com/4.27/en-US/InteractiveExperiences/ArtificialIntelligence/EQS/EQSNodeReference/EQSNodeReferenceTests/). Candidates are scored based on line of site to the AI's target (strong weight), and then a distance to the AI's target (lower weight, farther is better). The end result is the AI picks from locations without line of site from the target ("hidden") first, and then based on distance to the target second (within a maximum radius of the target, so the AI doesn't run too far away).

**Sequence: Heal Self and Cooldown:**

Finally, once it reaches its determined safe spot, the AI heals itself via executing [TBTTask\_HealSelf.cpp](https://github.com/Juwce/ActionRoguelike/blob/main/Source/ActionRoguelike/Private/AI/TBTTask\_HealSelf.cpp). This branch then goes on cooldown for 60 seconds (meaning the AI will not perform this sequence nor abort other lower priority nodes the next time the AI's health drops low as long as this cooldown is active).
