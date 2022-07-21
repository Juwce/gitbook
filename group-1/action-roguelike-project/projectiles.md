---
cover: ../../.gitbook/assets/7505745748a13757e9c0878c487a267f.gif
coverY: 0
---

# Projectiles

### Projectile Targeting

{% hint style="info" %}
Note const correctness is left out of smaller code snippets to keep the snippets within the line character limit.
{% endhint %}

The player's projectiles should fire at the nearest target under the player's reticle. To find this target, we perform a line trace from the player's camera into the world where the player is looking.

```cpp
FVector TraceStart = CameraComp->GetComponentLocation();
FVector ControlRotation = InstigatorCharacter->GetControlRotation().Vector();
FVector TraceEnd = TraceStart + (ControlRotation * MaxAttackTraceDistance);
```



{% embed url="https://gist.github.com/Juwce/bf9539f8229241762c36e250c09815aa" %}

### Projectile Base Class

A projectile base class `TProjectileBase` handles the basic setup of a projectile.

* Can collide with the environment, exploding upon impact
* Provides the following virtual functions:
  * `OnActorHit()` - callback when projectile hits the environment (blocking collision)
  * `Explode()` - play cosmetic effects and destroy self. Triggered by `OnActorHit()` by default
* Contains the basic components shared by all projectiles:
  * `USphereComponent` - Sphere primitive used for collision detection
  * `UProjectileMovementComponent` - Handle velocity, acceleration, direction, etc.
  * `UParticleSystemComponent` - Projectile and explosion VFX
  * `UAudioComponent` - Projectile and explosion sounds

### Magic Projectile

Magic projectiles are the most basic attack a player can perform, dealing damage, applying debuffs, and more. They extend the projectile base class with the following functionality:

Gameplay:

* Deals damage on impact
* Can apply buffs/debuffs on impact
* Can be parried / reflected

Cosmetic:

* Applies camera shake around impact point

### Dash Projectile

Dash projectiles fly forward, exploding on impact or after a set duration expires. Upon exploding, there is a short delay before the player is teleported to the explosion's impact point. They extend the projectile base class with the following functionality:

Gameplay:

* Teleports player to projectile location on impact, or after a set duration expires
