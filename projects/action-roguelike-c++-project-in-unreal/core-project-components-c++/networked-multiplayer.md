# Networked Multiplayer

Every aspect of this project also works in networked multiplayer (client-server)!

* Attributes
* Actions
* Powerups
* Projectiles
* User Interface
* Actor spawning and movement (built-in from Unreal)
* AI (built-in from Unreal)

Leverages Unreal Engine's networking framework for replicating game state, and limits the direct modification of non-cosmetic game state to the server (`HasAuthority()`). Clients wishing to change game state must do so through requests to the server. Care was taken to limit use of RPCs (remote procedure calls) and reliable / tcp network calls and to limit passed data to the bare minimum needed for a client/server to update game state in order to preserve network performance.



### Networking Attributes:

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

![Player Attributes (Health and Rage) are replicated over the network.](<../../../.gitbook/assets/networked projectiles and health 2.gif>)

### Frame by Frame:

![Client Action\_ProjectileAttack::StartAction(), triggering the action on the client and server. Both the client and the server start the attack windup animation and VFX (purely cosmetic). The server starts a timer to spawn the attack's projectile.](<../../../.gitbook/assets/image (1).png>)



![The server's timer is up and it spawns a magic projectile. That projectile is replicated to the client.](<../../../.gitbook/assets/image (9).png>)

![The magic projectile collides with player on server and client (as this screenshot was taken on a local network, the projectiles locations are almost exactly in sync) and play explosion VFX in response to the collision. Only the server applies the damage and burning effect to the hit actor. However, the damage and burning effect are then replicated back to the client. The server Multicasts the health change, and so the client and server both respond with cosmetic effects (update health bars, show floating damage text, apply hitflash effect, etc.)](<../../../.gitbook/assets/image (5).png>)
