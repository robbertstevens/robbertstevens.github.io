---
layout: post
title:  "Implementing a simple AI for games in Godot"
date:   2021-02-10 13:37:00 +0100
---
In my spare time, I like to create games in so-called [game jams](https://en.wikipedia.org/wiki/Game_jam). An event in 
which you create a game in a set amount of time. My engine of choice is Godot, because it's simple and powerful.

In my game "Dave & The Machine", I've implemented a super simple enemy AI using a finite state machine, which worked 
wonders for the game I've created in [Mini Jam 73](https://robbertstevens.itch.io/dave-and-the-machine).

## What is a Finite State Machine(FSM)
In a game, an enemy has many behaviors like wandering, attacking, and chasing. These are your states. With an FSM you 
can make sure only the logic for one of these states will be executed at the same time. 

## Implementing this in Godot
In the enemy node script, I've created an enum with the states the enemy can be in.
```gdscript
enum EnemyState {
    IDLE,
    CHASE,
    ATTACK,
    HURT
}
```

Then in the `_process(delta)` method, I've created a `match`-statement that matches a variable called `current_state`. 
This variable holds the current state the enemy is in. For every state, I created a method for clarity.

```gdscript
func _process(delta):
    match(current_state):
        EnemyState.IDLE: _idle_state()
        EnemyState.CHASE: _chase_state()
        EnemyState.ATTACK: _attack_state()
```

Now you can implement the states and their behaviors, I am going to show two states the `IDLE`-state and the 
`CHASING`-state. This way I can also show how you can transition to another state.

The idle state is super basic. For my game, this was the default state an enemy was in once it spawned in the world. 
It looked like this.

```gdscript
func _idle_state():
    $AnimationPlayer.play("idle")
    
    target = find_target()
    
    if target != null:
        current_state = EnemyState.CHASE
        return
```
So what is happening here? 
1. First, when the `_idle_state()`-method is executed, the `idle`-animation is played so the player can visually see what 
   the enemy is doing. 
2. Then I search for a target(the implementation is not really relevant)
3. Then I do a check whenever there is a target. 
   
If that's the case I set the variable `current_state` to `EnemyState.CHASE`. The variable `current_state` is the same 
variable that is used in the `process`-method. Do not forget to at a return statement after you chase the state, 
otherwise, the method will finish and if you have additional logic this leads to strange behavior.

Now let's see how I created the `_chase_state()`-method.

```gdscript
func _chase_state():
    if target == null:
        current_state = EnemyState.IDLE
        return 
    
    var direction = position.direction_to(target)
    
    $AnimationPlayer.play("walking")
    
    motion = move_and_slide(direction * speed)
```

As you can see the method is very similar to the `_idle_state()`-method. This is what I do:
1. First I check if there is still a target, if not I go to the idle state after all the enemy can't case nothing.
3. If there is a target, then the enemy moves in the direction of the target.

You can find the complete implementation of my game on my [github](https://github.com/robbertstevens/minijam73).

## Conclusion 
This is a very bare-bones implementation of the Finite State Machine design pattern, but for my use case, it was enough.
I've also experimented with a version of this design pattern [using different nodes for every state](https://github.com/robbertstevens/godot-statemachine) 
but I felt that it was too complicated to be used in a game jam. 

This implementation is also not very suitable for when you have a lot of different states, because your code will become a mess.
