---
layout: default
title: Sprouts of Safety
---

# Sprouts of Safety

`Godot, GDScript, GitHub`

[GitHub](https://github.com/RasheedC131/MorseCodeDefense) - [Devpost](https://devpost.com/software/morse-code-defense)

---

## Overview
Sprouts of Safety is an educational game that teaches you Morse code using your eyes. In this tower defense, you must deploy troops by blinking out the letters of the incoming enemies.

This project was made for Knight Hacks IX, a 36 hour hackathon. Our team tackled the challenge presented by Full Sail, to make a serious game or training simulation using face detection and/or pathfinding, we chose both. I took the charge on pathfinding and agent behavior, as well as assisting our teammates who were new to game development.

---

## Pathfinding and Navmesh
Godot allows you to bake a tileset into a navmesh with its NavigationRegion2D node. I built out the map using 2 layers: The floor as a Navigation Layer, and the walls as a physics layer.

![Tilemap Navmesh that was under the illustration](../assets/sprouts/navmesh.png)

Using this, agents with a NavigationAgent2D node can path find with the algorithm of your choice. I went with A* as we only needed the quickest path to the target.

![Agent Pathfinding Showcase](../assets/sprouts/pathfind.png)

---

## Balanced Enemy Spawning
Enemies in this game are broken into tiers based on the length of their letter code. Shorter codes represent weak but fast enemies, while longer codes represent strong but slow ones.
```gdscript
const LETTER_TIERS = {
	1: ["E", "T"],
	2: ["A", "I", "M", "N"],
	3: ["D", "G", "K", "O", "R", "S", "U", "W"],
	4: ["B", "C", "F", "H", "J", "L", "P", "Q", "V", "X", "Y", "Z"]
}
```
Due to the way morse code is built, the ratio of weak to strong letters is vastly unbalanced. If you spawned by choosing completely random, there is a 46% chance of getting the strongest enemy type, and only 8% of getting the weakest. To remedy this, I implemented a weighted algorithm. 
* Each tier is assigned a weight, higher numbers being more common.
* The weights are summed, a random number is picked from 0 to the random value.
* You loop around the tiers checking if the random value is lower than the current weight.
* If not, the value is subtracted by the current tier weight, and the loop moves to the next index.
* The loop is completed once the value finds a weight higher than it.
  
```gdscript
const TIER_WEIGHTS = {
	1: 17,
	2: 55,
	3: 25,
	4: 3
}

func get_weight_balanced_letter() -> String:
  #Add up weights
  var sum : int = 0
  for weight in TIER_WEIGHTS.values():
    sum += weight
	
  #Pick random 0-sum
  var rand_val = randi() % sum
  var rand_tier = 1
		
  #Find weight lower than random value, 
  #if not subtract weight from value and try again
  for tier in TIER_WEIGHTS:
    var current_weight = TIER_WEIGHTS[tier]
      if rand_val < current_weight:
        rand_tier = tier
        break
      else:
        rand_val -= current_weight
			
    return LETTER_TIERS[rand_tier].pick_random()
```

---

## Dynamic Target Priority
Allies of the same letter can help each other out if they outnumber the enemy, and pivot when a new threat enters the game. To implement this I utilized bool flags: The enemy tracks a flag for whether it's being targeted, and the allies track a flag for whether they are currently helping. 

### Allies follow these steps to find their target
* When an ally is spawned it searches through all enemies to find a match
* If there is no match, they leave
* If there is a match that isn't being targeted, it chases
* If there is a match that is already targeted, it helps attack

```gdscript
func find_target():
	#Get all enemies in scene
	var enemies : Array[Enemy] = []
	enemies.assign(get_tree().get_nodes_in_group("enemy"))
	
	#Find Enemy with matching letter
	var occupied_target: Enemy = null
	if not enemies.is_empty():
		for i in enemies:
			if i.is_queued_for_deletion():
				continue
			if i.letter == target_letter: #Match Found
				if i.is_targeted:
					if not is_instance_valid(occupied_target): #Implemented so they help most recent
						occupied_target = i
				else:
					lock_target(i)
					break
	
	#Didnt find a solo match
	if not is_instance_valid(nav_agent.target):
		#Revert to helping
		if is_instance_valid(occupied_target):
			lock_target(occupied_target)
		else: #No enemies of the letter at all
			await get_tree().create_timer(1).timeout
			queue_free()

func lock_target(enemy: Enemy) -> void:
	curr_speed = speed
	active_target = enemy
	nav_agent.target = active_target
	
	active_target.defeated.connect(_on_target_defeated)
	
	#Update Flags
	if not enemy.is_targeted:
		enemy.is_targeted = true
		enemy.rival = self
	else:
		is_helping = true
```
When a new enemy is added to the scene, helping allies will shift priority.
```gdscript
func _on_node_added_to_game(new_node: Node) -> void:
	if not is_helping or new_node is not Enemy:
		return
	
	if new_node.letter == target_letter:
		print("Found Own")
		is_helping = false
		lock_target(new_node)
```

---

[back](../)
