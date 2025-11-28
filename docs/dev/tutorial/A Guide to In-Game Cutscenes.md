---
sidebar_position: 4
---
# Transition Animation Hook Logic Introduction

## Overview

This document outlines two core hook methods for controlling the character's entrance cutscene into the throne room. These methods must work in tandem through both static and dynamic approaches to achieve the playback and logical control of character animations.

I shall use the cutscene depicting Wang Shou meeting the king as an example.
First, we must examine the ==dc.cin== class, which governs all cutscenes.
Should we wish to rewrite the entire method, we must simultaneously hook two or more methods to collaboratively execute the cutscene.
```csharp
Hook_EnterThroneRoomAsKing.update += Hook_EnterThroneRoomAsKing_update;//Dynamic Class
Hook__EnterThroneRoomAsKing.__constructor__ += Hook__EnterThroneRoomAsKing__constructor__;//Static class
```


---

## Code structure

### 1. Constructor Hook (`Hook_EnterThroneRoomAsKing_constructor_`)

**Functionality**: Executes during the initialisation of cutscene objects, primarily responsible for the **static playback** of character animations.

```csharp
private void Hook_EnterThroneRoomAsKing_constructor_(Hook_EnterThroneRoomAsKing_orig_constructor_ orig, EnterThroneRoomAsKing _hero, Hero game)
{
    orig(_hero, game); // Safe invocation of the original constructor
    // Since we have dynamically overridden the new logic, we can safely call orig.

    var boss = _hero.boss;
    if (boss.cx - _hero.hero.cx > 3) // Judging the Distance to Heroism
    {
        if (boss.spr.groupName?.ToString() != "runShield")
            // Play the 'Running with Shield' animation statically and loop it 999 times.
            boss.spr.get_m().play("runShield".AsHaxeString(), 0, false).loop(999); 
    }
    // Example: Play the animation statically. Have Wang Shou run.
}
```

**Key logic**：

- **Distance Check**: Verify the horizontal distance between the Boss and hero characters exceeds 3 units.
    
- **Animation Transition**: If the Boss is not currently in the `runShield` state, play the shield-bearing dash animation.
    
- **Loop Playback**: Configure the animation to loop 999 times, ensuring continuous execution.
    

### 2. Update function Hook (`Hook_EnterThroneRoomAsKing_update`)

**Functionality**: Executes upon each frame update, primarily responsible for **dynamically modifying** character attributes.

```csharp
private void Hook_EnterThroneRoomAsKing_update(Hook_EnterThroneRoomAsKing_orig_update orig, EnterThroneRoomAsKing self)
{
    orig(self); // Call the native update function
    
    self.cm = new Cinematic((int)self.tmod); // Initialise the cutscene controller
    var boss = self.boss;
    boss.dx = 0.5 * (double)boss.dir; // Set movement speed according to direction
}
```

**Key Logic**:

- **​ cinematic Control: Create a new transition animation controller

:::warning
> **A new Cinematic instance must be created** > If a new `Cinematic` object is not explicitly created, the game engine will execute the default original cutscene logic.。
:::
---
# After running, it looks like this.

<iframe src="//player.bilibili.com/player.html?isOutside=true&aid=115627398269311&bvid=BV1iCSEBzEQn&cid=34337590920&p=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>

---


## Dynamic and Static Method Collaboration Mechanism

### Static Methods

- **Execution Timing**: Executed once during initialisation
    
- **Primary Function**: Play Sprite animations, set initial state
    
- **Example**: `boss.spr.get_m().play()` - Retrieve and play corresponding animation
    

### Dynamic Methods (Dynamic)

- **Execution Timing**: Executed during each frame update
    
- **Primary Function**: Modify position, velocity, and other attributes in real-time
    
- **Example**: `boss.dx = 0.5 * boss.dir` - Controls movement speed `Moves towards the boss's facing direction at 0.5 speed`. This automatically triggers the walking animation.

### Collaborative Relationship

![项目结构图](./img/Pasted.png)
1. **Static methods lay the groundwork**: Setting up animations to be played during initialisation
    
2. **Dynamic methods adjust in real time**: Continuously updating object states whilst the game is running
    
3. **Both work in tandem**: Static animations provide visual presentation, whilst dynamic logic delivers behavioural control
    

---

### Animation Control Techniques

- **State Checking**: Verify the current state before playing animations to prevent redundant playback.
    
- **Distance Triggering**: Conditionally activate animations based on character proximity.
    
- **Frame Synchronisation**: Ensure animations play in tandem with positional updates.
    

---

This separation of animation and logic design guarantees both stable visual performance and flexible behavioural control.