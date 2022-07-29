# Networked Multiplayer

![Multiplayer support for any number of players.](../../../.gitbook/assets/bb547cfcbe98f258ec8337548dff8bee.gif)

Every aspect of this project also works in networked multiplayer for any number of players!

* Actions
* Attributes
* Powerups
* Projectiles
* User Interface
* Actor spawning and movement (built-in from Unreal)
* AI (built-in from Unreal)

This project leverages Unreal Engine's C++ networking framework for replicating game state, and limits the direct modification of non-cosmetic game state to the server (`HasAuthority()`). Clients wishing to change game state must do so through requests to the server. Care was taken to limit use of RPCs (remote procedure calls) and reliable / tcp network calls and to limit passed data to the bare minimum needed for a client/server to update game state in order to preserve network performance.

Instead of covering every detail of this project's networking, the below sections break down one of the most challenging networking implementations in the project, networked actions. Note that some of the logic comes from [the course](../../../group-1/action-roguelike-c++-project-in-unreal/) I took, but the thorough breakdown of how it all works, and diagrams below are entirely my own.

### Jump to Section...

* [Networked Action Framework](networked-multiplayer.md#networked-action-framework)
* [Frame-by-Frame Example](networked-multiplayer.md#frame-by-frame-example)

## Networked Action Framework

"Actions" are part of the Action Framework designed by [Tom Looman](https://github.com/tomlooman/ActionRoguelike). Diagrams and examples below are entirely my own.

In this Action framework, any actor assigned an Action Component ([TActionComponent.cpp](https://github.com/Juwce/ActionRoguelike/blob/main/Source/ActionRoguelike/Private/TActionComponent.cpp)) has access to an array of Action objects ([TAction.cpp](https://github.com/Juwce/ActionRoguelike/blob/main/Source/ActionRoguelike/Private/TAction.cpp)) that define arbitrary logic that can be started and stopped. Actions can be granted or revoked to an actor, giving them and taking away functionality at the start of or during gameplay (e.g. sprinting, attacks, jumping, etc.). The actions within an action component, as well as the starting and stopping of actions are replicated across the network, providing a useful way to manage the performing of arbitrary gameplay actions over the network to any number of players.

Actions exhibit the following networked behavior:

1. Only the server can grant or revoke actions.
2. Action instances stored in an actor's ActionComponent are replicated across the network.
3. Clients can start actions, but only if they have been granted that action server.
4. Whenever a client OR server starts an action, that action is started in all other sessions on the network.

### Code for Networked Actions

{% hint style="info" %}
Click on the "Diagram" tab to see a flow-chart walking through how the logic replicates between client and server.
{% endhint %}

{% tabs %}
{% tab title="Code" %}
**1.** Only the server can grant or revoke actions:

{% code title="ActionComponent.cpp" %}
```cpp
void UTActionComponent::AddAction(
    AActor* Instigator, const TSubclassOf<UTAction> ActionClass)
{
    if (!GetOwner()->HasAuthority)
    {
        UE_LOG(LogTemp, Warning,
            TEXT("Client attempting to AddAction, Actions should only be added"
                 " on the server. [Class: %s]"),
            *GetNameSafe(ActionClass));
	return;
    }
 
    // Create instance of ActionClass and add it to Actions array
    ...   
}
```
{% endcode %}



**2.** Action instances stored in an actor's ActionComponent are replicated across the network:

{% code title="ActionComponent.h" %}
```cpp
UPROPERTY(Replicated)
TArray<UTAction*> Actions;
```
{% endcode %}



{% code title="ActionComponent.cpp" %}
```cpp
void UTActionComponent::GetLifetimeReplicatedProps(
	TArray< FLifetimeProperty > & OutLifetimeProps ) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    // also mark as UPROPERTY(Replicated) in ActionComponent.h
    DOREPLIFETIME( UTActionComponent, Actions );
}

bool UTActionComponent::ReplicateSubobjects(
	UActorChannel* Channel, FOutBunch* Bunch, FReplicationFlags* RepFlags)
{
    bool WroteSomething = Super::ReplicateSubobjects(Channel, Bunch, RepFlags);
    // Replicate every action that had a change
    for (UTAction* Action: Actions)
    {
        if (Action)
        {
            WroteSomething |= Channel->ReplicateSubobject(
                                            Action, *Bunch, *RepFlags);
        }
    }
    // tells Unreal "there was a change in this component, please replicate my data"
    return WroteSomething;
}
```
{% endcode %}
{% endtab %}

{% tab title="Diagram" %}
![Flow diagram for 1 & 2](<../../../.gitbook/assets/image (10).png>)
{% endtab %}
{% endtabs %}



{% tabs %}
{% tab title="Code" %}
**3.** Anyone can start an action. If an action is started by the client, also start that action on the server:

{% code title="ActionComponent.cpp" %}
```cpp
bool UTActionComponent::StartActionByName(
    AActor* Instigator, const FName ActionName)
{
    ...
    if (!GetOwner()->HasAuthority()) // is client
    {
        ServerStartAction(Instigator, ActionName);
    }
    ...
}

// Set UFUNCTION(Server, Reliable) in TActionComponent.h
void UTActionComponent::ServerStartAction_Implementation(
    AActor* Instigator, FName ActionName)
{
    StartActionByName(Instigator, ActionName);
}
```
{% endcode %}



{% code title="ActionComponent.h" %}
```cpp
UFUNCTION(Server, Reliable)
void ServerStartAction(AActor* Instigator, FName ActionName);
```
{% endcode %}



**4.** Whenever a client OR server starts an action, that action is started in all other sessions on the network.

{% code title="TAction.h" %}
```cpp
struct FActionRepData
{
    GENERATED_BODY()
    
public:	
    UPROPERTY()
    bool bIsRunning;

    UPROPERTY()
    AActor* Instigator;
};

...

UPROPERTY(ReplicatedUsing="OnRep_RepData")
FActionRepData RepData;

UFUNCTION()
void OnRep_RepData();
```
{% endcode %}



{% code title="TAction.cpp" %}
```cpp
void UTAction::OnRep_RepData()
{
    if (RepData.bIsRunning)
    {
        StartAction(RepData.Instigator);
    }
    else
    {
        StopAction(RepData.Instigator);
    }
}
```
{% endcode %}
{% endtab %}

{% tab title="Diagram" %}
![Flow diagram for 3 & 4](<../../../.gitbook/assets/image (7).png>)
{% endtab %}
{% endtabs %}

## Frame-by-Frame Example:

This section breaks down the network logic for this networked attack sequence frame-by-frame:

![The player's attack replicates over the network.](<../../../.gitbook/assets/networked projectiles and health 2.gif>)

### 1. Client starts an Attack

![](<../../../.gitbook/assets/image (1) (2).png>)

In this frame we see the client's character starting an attack (in the form of playing a wind-up animation and spawning particle VFX at the hand). The attack is replicated on the server (note this was captured on a local network, both actions wouldn't necessarily be seen on the same frame in a higher latency environment).

**Sequence Diagram**

![Exact function names left out for brevity.](<../../../.gitbook/assets/replication example projectile dealing damage.drawio (4).png>)

The client pressed their attack key, immediately running the cosmetic parts of the attack locally. The client does not start a timer to spawn the attack projectile as projectiles are replicated server->client, making spawning one here redundant. The client makes a **reliable** RPC to the server to also start the action. The server then runs the action, playing cosmetic effects but this time also setting the timer to spawn the attack projectile. If there were a second client in this instance, the server would also replicate the Attack action's `IsRunning` property, triggering a RepNotify on the second client that would also play the cosmetic effects of the attack on that client.

**Code**

* [TAction\_ProjectileAttack.h](https://github.com/Juwce/ActionRoguelike/blob/main/Source/ActionRoguelike/Private/TAction\_ProjectileAttack.cpp) (also see Networked Actions section above)

### 2. Server spawns attack projectile

![](<../../../.gitbook/assets/image (9).png>)

The server's timer is up and it spawns a magic projectile. That projectile is replicated to the client.

**Sequence Diagram**

![](<../../../.gitbook/assets/image (1).png>)

The server's timer runs out and it computes the location to fire the projectile towards (from the player's hand position on the server towards the nearest actor underneath the player's reticle) and then spawns the projectile. The projectile is then replicated to both clients (including it's direction and movement properties).

Latency Factors to Consider:

* The attack target is computed based on the player's control rotation (with the way the camera is set up, this is directly under their reticle where they are looking). Especially with a mouse, if the client is looking around quickly their control rotation could vary quite a bit from the server's last recorded control rotation from the player, causing the projectile to fire in a different direction than the client saw on their screen as the animation played. This would require further testing under higher latency conditions to optimize.

**Code**

* Replication of projectiles is handled entirely by Unreal out-of-the-box.

### 3. Attack projectile damages and burns an opponent player

![](<../../../.gitbook/assets/image (5).png>)

The magic projectile collides with player on server and client (as this screenshot was taken on a local network, the projectiles locations are almost exactly in sync) and play explosion VFX in response to the collision. Only the server applies the damage and burning effect to the hit actor. However, the damage and burning effect are then replicated back to the client. The server Multicasts the health change, and so the client and server both respond with cosmetic effects (update health bars, show floating damage text, apply hitflash effect, etc.)

**Sequence Diagram**

![](../../../.gitbook/assets/image.png)

The projectile overlaps with another actor on both the server and the client (due to good latency, both projectiles are in sync), playing explosion effects on each. The server then applies damage to the overlapped actor. That damage is then replicated back to the client, and a **reliable** multicast RPC triggers a health change broadcast, notifying all listeners (UI, etc.). Listeners on the client play corresponding effects (damage popup text, health bar updates, etc.) and listeners on the server play effects and apply further attribute changes (for example updating rage which increases when damaged \[also networked but not pictured in diagram]). Finally, the server applies a burning effect to the overlapped actor which is replicated via the Action framework discussed above.

Latency considerations:

* In this example, all three projectiles are in sync and overlap with the same actor on the client and server. However this may not always be in case if the game states fall out of sync on high latency connections. In these cases, the server would remain the authority on the game state, but the client(s) would see odd visual behavior:
  * Projectile overlaps on client but not server:
    * Client will see explosion effects but damage and burning effect will not be applied
  * Projectile overlaps on server but not client
    * Client will not see an explosion (unless it overlaps or hits something else), but damage and burning effect will be applied
  * Projectile overlaps on both (pictured in frame)
    * Client and server see explosion effects and damage and burning effect are applied.

**Code**

* [TProjectile\_Magic.cpp](https://github.com/Juwce/ActionRoguelike/blob/main/Source/ActionRoguelike/Private/TProjectile\_Magic.cpp)

****

<details>

<summary>Work in progress...</summary>

### Networked Attributes

#### Replicated Properties

* Health
* HealthMax
* Rage
* RageMax

#### Limiting Authority

* Only the server (`HasAuthority()`) should be able to modify player attributes such as health and rage.
* Attribute properties are protected and can only be modified via the `ApplyHealthChange()` and `ApplyRageChange()` functions. These functions check that the caller `HasAuthority()` (is the server).

#### Broadcasting Attribute Change Events

* When an attribute is changed on the server via `ApplyHealthChange()`, the server sends out a `NetMulticast` (calls function on server and all clients) so that clients can respond cosmetically to attribute changes (such as health and rage UI elements), and the server can respond with further gameplay logic (e.g. reflecting damage dealt back to the attacker).
* `NetMulticast` is not the most optimal solution to do this, but it works well and is used sparingly in the code at this point so should not cause performance issues. With more time, I would work on a solution involving replicating structs containing all the requisite data on damage instigator, as well as counters to keep track of if property replication missed a state change, and using `RepNotify` to respond to changes. For example, if the server changed health 10->5->0, but the client only received 10->0 (the change to 5 was lost), the client would not be aware of the health change from 10->5 and would miss playing a hitflash effect for that instance of damage. However, if you instead replicated a struct that counts the number of times that property has been changed in addition to the new property value, the client could miss this change but still receive enough information to play the hitflash effect. For example, the server would change (10,0)->(5,1)->(0,2) and the client would receive (10,0)->(0,2). Since the client saw the counter go from 0->2, it knows that it missed a health change RepNotify and can respond cosmetically accordingly (play an extra hitflash, sound effect, etc.).

</details>

