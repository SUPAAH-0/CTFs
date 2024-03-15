
# Description
`Far off in the distance, you hear a howl. Your blood runs cold as you realise the Kara'ka-ran have been released - vicious animals tortured beyond all recognition, starved to provide a natural savagery. They will chase you until the ends of the earth; your only chance of survival lies in a fight. Strong but not stupid, they will back off if they see you take down some of their number - briefly, anyway...`



# Method #1: Play the game

This challenge could have been solved by simply visiting the docker `HOST:PORT`, in your browser of choice, and following the `/rules`:

![[Pasted image 20240314104614.png]]

You use either the WASD or arrow keys, to go from the tile with the player (which we are assuming is out of ammunition), to the tile with the much less advanced crossbow (which we know to be the difference between life and death).

Take this example map:
`Time left: 10`
![[Pasted image 20240314105232.png]]

Our objective is to travel, from the middle-right tile, to the bottom-left tile.
If we look at the "Time Costs" section of the rules, we can see that heading down-left-left will cost us a total of:
```
p -> p = 1,
p -> m = 5,
m -> p = 2

1 + 5 + 2 = 8
```

As a time cost of 8 is less time than the 10 that we are given for this level, and there are no cliffs, geysers, or empty tiles to take into consideration, this is a valid path.

Heading back to the game and pressing `S A A`, we are greeted with this alert:
![[Pasted image 20240314110514.png]]

Follow the logic of `/rules` 100 times and you are given the flag.

# Method #2: Do it for the snek
`(Full code at the end)`

Who needs Dijkstra's algorithm? Well, we should use that, but that wouldn't be nearly as fun.

For this method, we adapt the Breadth-First Search (BFS) algorithm, which is not designed to be used with weighted edge graphs (which is what this game can be reduced to).
A weighted edge graph is a collection of vertices that may or may not be connected by an edge, that has a value attached. We can picture this value to be the distance (or length) of a road, in which the vertices represent a city.

Doing it for the snek, we ignore the weights and find the shortest path between the player and a weapon (as though all weights were of equal value).

My initial draft of this used the closest weapon to start, but it severely reduced the number of snakes in my program. Instead, I have looped through each weapon that was found during the creation of the local map state.

## Creating the local map state

Before we can create the local map state, we need to create a python file and send a request to the challenge server, to receive the game data.

```python
import requests
import json


def post_request(uri, data):
	"""sends a post request to '[host]:[port]/[uri]', with the supplied data"""
	# required for "update", but doesn't affect "map"
	headers={'Content-type':'application/json', 'Accept':'application/json'}
	
	return requests.post("http://" + ip_port + "/" + uri, data=data, headers=headers)


def main():
	"""program spine"""
	current_map = post_request("map", "").json()
	with open("map.json", "w") as map_json: # make a map.json file in the same directory as this file
		json.dump(current_map, map_json)
	with open("map.json", "r") as map_json: # avoid race conditions
		current_map = json.load(map_json)

	print(current_map)


if __name__ == "__main__":
	ip_port = "83.136.250.41:45025" # changes depending on docker spawn
	main()
```

As I was receiving errors trying to use the returned JSON as it was received (and using other JSON operations on the response in-script), I used a dirty workaround. I decided on manually creating a `map.json` file in the directory, then used a few lines to write the JSON to it, and then load it back in. Doing this in two opens, instead of `r+` avoided the race condition where the wrong data was loaded back in.

Running this code, we get:
```
{'height': 11, 'player': {'position': [9, 2], 'time': 26}, 'tiles': {'(0, 0)': {'has_weapon': False, 'terrain': 'E'}, '(0, 1)': {'has_weapon': True, 'terrain': 'P'}, ... '(9, 9)': {'has_weapon': False, 'terrain': 'E'}}, 'width': 15}
```
The values will be different, depending on your specific instance, as they are procedurally generated. Completing the level (either by failing or succeeding) will generate a fresh map.

Next, we need to sift through the JSON and pick out values that we will need later on:
```python
# store relevant data in global variables
global map_tiles, map_height, map_width, player_position, remaining_time
map_tiles = current_map["tiles"]
map_height = current_map["height"]
map_width = current_map["width"]
player_position = current_map["player"]["position"]
remaining_time = current_map["player"]["time"]
```


Now that we are done with the request for the server's state of the map, it's time to make our own board, that will be easier for our BFS function to search.

The steps to create the local map state are:
- Create a 2D array, where the only values are the tile's terrain or one of our special characters (Empty, Player and Weapon all have their own global variable)
- Return the map_state and the list of weapons


I personally use human readable symbols in my development & testing, as I am a human (I am definitely not 6 ducks in a trench coat...). As such, I have used "£" to denote an empty cell, "+" to denote that a weapon is present in the cell and a "\*" for the player. These are global variables, that I have defined before running the main function:

```python
if __name__ == "__main__":
	ip_port = "83.136.250.41:45025" # changes depending on docker spawn

	# store global constants
	player_icon = "*"
	weapon_icon = "+"
	empty_icon = "£"

	main()
```

```python
def get_map_state():
	"""generates the current map state, returning the map grid and list of weapon coords"""
	map_state = []
	weapons = []

	for y in range(map_height):
		row = []
		for x in range(map_width):
			current_coord = "(" + str(x) + ", " + str(y) + ")" # same as str(tuple(x,y))
			coord = "" # initiate coord as empty string

			current_tile = map_tiles[current_coord]
			tile_terrain = current_tile["terrain"]

			if(player_position == [x, y]):
				coord = player_icon
			elif(current_tile["has_weapon"] == True):
				coord = weapon_icon
				weapons.append([x, y])
			else:
				if(tile_terrain == "E"):
					coord = empty_icon
				else:
					coord = tile_terrain

			row.append(coord)
		map_state.append(row)

	return map_state, weapons
```

This can be printed with the following (passing in any map_state):

```python
def print_map(map_state):
	"""prints the provided map state"""
	for row in map_state:
		print(" ".join(row))
		print()
```

![[Pasted image 20240314123901.png]]


## Recursion Hell

Now that we have a local map state and list of weapons, it's time to enter recursion hell (if you need to increase your recursion limit, you're most likely doing it wrong, but it makes for more snakes)

```python
cheap_path = False
for weapon_to_use in weapons:
	print("weapon_to_use: " + str(weapon_to_use))
	weapons_to_remove = copy.deepcopy(weapons)
	weapons_to_remove.remove(weapon_to_use)

	new_map_state = copy.deepcopy(map_state)
	for each_weapon in weapons_to_remove:
		new_map_state[each_weapon[1]][each_weapon[0]] = empty_icon
	print_map(new_map_state) # human readable map is printed to output

	path_for_weapon = BFS(new_map_state)

	queue_of_paths = deque()
	if(path_for_weapon[-1] == tuple(weapon_to_use[::-1])):

		path_to_append = deque(path_for_weapon)
		path_to_append.appendleft(new_map_state)
		queue_of_paths.appendleft(path_to_append)
		all_tested_states = set()

		working_path_and_cost = rerun_until_cheap(queue_of_paths, all_tested_states, False, 0)
		if(working_path_and_cost[0] != False and len(working_path_and_cost[0]) > 1):
			cheap_path = working_path_and_cost[0]
			path_cost = working_path_and_cost[1]
			output_directions = working_path_and_cost[2]
			print("new cheap path: " + str(cheap_path))
			break
```

A lot is happening here, as this is the main spine of the program. We can break this down into smaller sections:

Where `copy.deepcopy` is used for copying lists, the following:
- Tells us which weapon we currently want to use
- Creates a list of weapons that we don't want to use
- Copies the main map_state, to store the edited version
- Edits the copied map state, to remove all of the unwanted weapons
```python
import copy

# ...

print("weapon_to_use: " + str(weapon_to_use))
weapons_to_remove = copy.deepcopy(weapons)
weapons_to_remove.remove(weapon_to_use)

new_map_state = copy.deepcopy(map_state)
for each_weapon in weapons_to_remove:
	new_map_state[each_weapon[1]][each_weapon[0]] = empty_icon
```

As mentioned at the beginning, this is not going to result in the fastest completion, on average, but it will ensure that we have a lot of snakes (which is the reason that this method uses a sub-optimal algorithm).

Now that we have a new map state, with only our target weapon (to reduce the number of iterations, since I still needed to calculate a solution before the event ended), it is time to find the shortest path.

```python
path_for_weapon = BFS(new_map_state)
```

This calls the following `BFS` function and stores the returned path:

```python
from collections import deque

# ...

def BFS(map_state):
	"""Breadth-First Search"""
	start = player_position[1], player_position[0]
	queue = deque()
	queue.append(start)
	visited_tiles = set()

	parents = dict()
	parents[start] = None
	do_break = False
	while queue:
		
		tile = queue.popleft()
		if(map_state[tile[0]][tile[1]] == '+'):
			break # found weapon
		else:
			visited_tiles.add(tile)
			for neighbour in neighbours(map_state, tile):
				if neighbour not in visited_tiles:
					parents[neighbour] = tile
					queue.append(neighbour)

	path = list()
	while tile != None:
		path.append(tile)
		tile = parents[tile]

	path = path[::-1]
	return path
```

This is the first time that we use deque in this program. Simply put, it is a more efficient way to use lists. We want to see as many snakes as possible, but also in the shortest time possible.

In this function, we also call `neighbours()`, which takes the current `tile` and the `map_state`, to find the next tile in our path:

```python
def neighbours(map_state, tile):
	"""gets the next valid neighbour"""
	modifiers = [(0, 1), (0, -1), (1, 0), (-1, 0)]
	returned_coords = list()
	for modifier in modifiers:
		newr = modifier[0] + tile[0]
		newc = modifier[1] + tile[1]
		if(newr < 0 or newr >= len(map_state) or newc < 0 or newc >= len(map_state[0])):
			continue

		direction = ""

		if(modifier == (0, 1)):
			direction = "R"
		elif(modifier == (0, -1)):
			direction = "L"
		elif(modifier == (1, 0)):
			direction = "D"
		elif(modifier == (-1, 0)):
			direction = "U"

		current_tile_terrain = map_tiles["(" + str(newc) + ", " + str(newr) + ")"]["terrain"]
		if(current_tile_terrain in ["C", "G"]):
			if(current_tile_terrain == "C" and direction in ["L","U"]):
				continue
			elif(current_tile_terrain == "G" and direction in ["R","D"]):
				continue

		if(map_state[newr][newc] == empty_icon):
			continue

		returned_coords.append((newr, newc))
	return returned_coords
```

During this phase of the program, we are checking to make sure that the neighbour is a valid tile, based on the `/rules` of the game. This checks to make sure that the tile is not empty and that the tile can be visited from the previous tile (in the case of cliffs and geysers)

At the end of the `BFS`, we then snake through the dictionary of parents, to get our path from the weapon to the player position. This then gets reversed, so that we are returning the path from the player to the weapon.

```python
path = path[::-1]
return path
```


Now that we have our returned path, we are left with this block of code:

```python
queue_of_paths = deque()
if(path_for_weapon[-1] == tuple(weapon_to_use[::-1])):

	path_to_append = deque(path_for_weapon)
	path_to_append.appendleft(new_map_state)
	queue_of_paths.appendleft(path_to_append)
	all_tested_states = set()

	working_path_and_cost = rerun_until_cheap(queue_of_paths, all_tested_states, False, 0)
```

Making sure that we have indeed been able to find the weapon, we:
- Add our modified map state to the front of the path (so that we later know which map state the path stems from)
- Create a blank set, that we will later fill with all of our modified map states, for this weapon
- Call the function to give us our glorious snakes, and return a list of either a valid path, cost and directions, or False, False, False

This function:
- Gets the first path in the queue & the map state tied to that path
```python
def rerun_until_cheap(queue_of_paths, all_tested_states, cheap_path_found, total_recursion):
	"""SNAKES"""
	queue_to_loop = queue_of_paths.popleft() # get the 'player to weapon' queue

	map_state = queue_to_loop.popleft() # get the passed map_state, from the queue
```

- Calls `get_path_cost()` and stores the returned values
```python
path_cost, output_directions = get_path_cost(queue_to_loop)
```

- If the cost is too expensive, loop through the path and add a new map state & path to the queue, where each looped coordinate is replaced with an `empty_icon`, which is the symbol used for an unpathable tile
```python
if(path_cost > remaining_time):
	for each_unpathable in queue_to_loop:
		if((each_unpathable != queue_to_loop[0]) and (each_unpathable != queue_to_loop[-1])):
			map_state_copy = copy.deepcopy(map_state)
			map_state_copy[each_unpathable[0]][each_unpathable[1]] = empty_icon
			test_map_state = tuple(tuple(a) for a in map_state_copy) # create a tuple form of the map_State, to be stored in the set

			if(test_map_state not in all_tested_states): # make sure map_state has not already been tested (speeds up the search and stops infinite loop)
				new_path = BFS(map_state_copy) # perform the new shortest path search
				if(new_path[-1] == tuple(queue_to_loop[-1])):
					path_cost, output_directions = get_path_cost(new_path)

					if(path_cost > remaining_time):
						new_path_to_append = deque(new_path)
						new_path_to_append.appendleft(map_state_copy)
						queue_of_paths.append(new_path_to_append)
						all_tested_states.add(test_map_state)

					else:
						cheap_path_found = True
						break
```

- Else (if the path is equal to or cheaper than the map's time limit), tell the program that we have now indeed found a cheap enough path and set the `new_path` (returned path) to be equal to the path that is currently being looked at
```python
else:
	new_path = queue_to_loop
	cheap_path_found = True
```

- The last block of this enter us into recursion, if the cheap path is still not found, up to a total of our defined `snake_number`. Just before we originally called `main()`, we also add some extra depth to python's recursion. This is because 1000 (the default value) was not deep enough for some of my tested number of snakes. For this algorithm, the more the merrier, in case a more difficult map requires more of our snakes
```python
if(cheap_path_found):
	cheap_path = new_path
else:
	if(len(queue_of_paths) > 0):
		total_recursion += 1
		if(total_recursion < snake_number):
			cheap_path, path_cost, output_directions = rerun_until_cheap(queue_of_paths, all_tested_states, cheap_path_found, total_recursion)
		else:
			return False, False, False

	else:
		cheap_path = False
		path_cost = False

return cheap_path, path_cost, output_directions
```
```python
import sys

# ...

if __name__ == "__main__":

	# ...
	
	snake_number = 400
	sys.setrecursionlimit(1000+snake_number)

	main()
```


The `get_path_cost()` function is nothing fancy. It defines the cost of moving from one type of terrain to another, with a set of nested `if` statements for cliffs, geysers, and same-to-same terrains. It also checks which direction this move is taking place in, so that a valid path can later be sent back to the server. It returns the total cost of the path as well as the list of directions:
```python
def get_path_cost(path):
	"""calculates how expensive a path is"""
	output_directions = []
	returned_cost = 0
	movement_costs = {("P", "M"): 5, ("M", "P"): 2, ("P", "S"): 2, ("S", "P"): 2, ("P", "R"): 5, ("R", "P"): 5, ("M", "S"): 5, ("S", "M"): 7, ("M", "R"): 8, ("R", "M"): 10, ("S", "R"): 8, ("R", "S"): 6}
	for i in range(len(path)-1):
		current_tile_terrain = map_tiles["(" + str(path[i][1]) + ", " + str(path[i][0]) + ")"]["terrain"]
		next_tile_terrain = map_tiles["(" + str(path[i+1][1]) + ", " + str(path[i+1][0]) + ")"]["terrain"]

		direction = ""

		if(path[i+1][0] < path[i][0]):
			direction = "U"
		elif(path[i+1][0] > path[i][0]):
			direction = "D"
		elif(path[i+1][1] < path[i][1]):
			direction = "L"
		elif(path[i+1][1] > path[i][1]):
			direction = "R"

		output_directions.append(direction)

		if(current_tile_terrain == "G" or next_tile_terrain == "G"):
			if(next_tile_terrain == "G" and direction in ["R","D"]):
				print("invalid path")
				returned_cost = "invalid"
				break
			else:
				returned_cost += 1
		elif(current_tile_terrain == "C" or next_tile_terrain == "C"):
			if(next_tile_terrain == "C" and direction in ["L","U"]):
				print("invalid path")
				returned_cost = "invalid"
				break
			else:
				returned_cost += 1
		elif(current_tile_terrain == next_tile_terrain):
			returned_cost += 1
		else:
			movement_cost = movement_costs[(current_tile_terrain, next_tile_terrain)]

			returned_cost += movement_cost

	return returned_cost, output_directions
```



## Final mile

Back in our main function, we now need to get some values from the recursion hell:
```python
if(working_path_and_cost[0] != False and len(working_path_and_cost[0]) > 1):
	cheap_path = working_path_and_cost[0]
	path_cost = working_path_and_cost[1]
	output_directions = working_path_and_cost[2]
	break
```

As a failsafe, to make sure that the returned path is not empty or only the player location, we check for `False` and the length of the returned path. Then, we parse the returned variable and break out of the for loop that this whole program has been running inside of (as there is no need to search for other weapons, as we have found one).

Now, it's time to send our winning path back to the server. We call `post_directions_to_server(output_directions)`, which is a simple for loop, to post each letter direction (URDL) to the server, as was mentioned on the `/api` page:
```python
def post_directions_to_server(directions_to_upload):
	"""sends the directions to the server, to complete the level"""
	for each_direction in directions_to_upload:
		data = {"direction": each_direction}
		response = post_request("update", json.dumps(data))
	return response.json()
```

The printed responses will be in this format, giving us the `new_pos` and new `time` that we have remaining (which our method ignores, as we used a local map state):
![[Pasted image 20240314151124.png]]
The important response for us is the final one of the sequence. When all levels have been solved, this is where the flag will be displayed. At the end of the for loop, this is the one that will still be stored as `response`. So, we can just return this value

As there are 100 levels, we call the `main()` function 100 times. As we do not need the iteration number, `_` suffices for the variable name:
```python
for _ in range(100):
	main()
```

## Impossible generations

That's right, they exist. I received confirmation from the author and have come across them a couple of times. They will terminate before the `snake_number` is hit and return False, with an `UnboundLocalError`:
![[Pasted image 20240314153005.png]]
(and then the `UnboundLocalError`)

To combat this, we try/except the line causing the `UnboundLocalError`, returning the last response, if successful. If there is an impossible generation, we regenerate the game (starting again from 0 completed levels) by sending a `get` request to `/regenerate`, and tell the code looping `main()` that it needs to continue looping:

```python
try:
	last_response = post_directions_to_server(output_directions)
except UnboundLocalError:
	get_request("regenerate")
	last_response = {}
return last_response
```

Note: This will also regenerate your game for maps that are pathable, but not within your `snake_number`.

Since we no longer know how many loops to expect, we also need to change our for loop to loop until the flag is found:
```python
last_response = {}
while "flag" not in last_response:
	last_response = main()
print(last_response["flag"])
```
![[Pasted image 20240314163931.png]]


## But where is the snek?

That's it. Just let it run and your snakes come to life.

I have written another script, to use the paths that the snake(s) take, and output a visual.
The script outputs an animated `.gif` file:
![[snake_1.gif]]

In order to visualise the snek, we need to collect our map states & paths and send them off to another function (I have separated this into its own python module).

Note, you will need to create an empty `images` folder, in the directory that the main python file and `snake_gif.py` file are in

By editing the rerun function, we can add the path, with the map state, to our `deque` of frames. The following two code snippets are the modified lines, that can be found in the main python file:
```python
def rerun_until_cheap(queue_of_paths, all_tested_states, cheap_path_found, total_recursion):
	"""SNAKES"""
	queue_to_loop = queue_of_paths.popleft() # already in the main code
	send_to_gif = copy.deepcopy(queue_to_loop) # deepcopy doesn't remove an element from the queue
	queue_of_frames.append(send_to_gif) # queues are appended from anywhere
```

```python
import snake_gif

# ...

image_number = 1
last_response = {}
while "flag" not in last_response:
	queue_of_frames = deque()
	last_response = main()
	snake_gif.make_snake_gif(queue_of_frames, image_number)
	image_number += 1
	print(last_response)
```

`import snake_gif`??? That's this file, that will need to be created and stored in the same directory as the main solve file:

```python
from PIL import Image

def make_snake_gif(gif_frame_queue, image_number):
	width = 21
	height = 21
	border_thickness = 2

	cell_border = Image.new("RGB", (width + (border_thickness * 2), height + (border_thickness * 2)), (0, 0, 0)) # black background (tile)
	snake_border = Image.new("RGB", (width + (border_thickness * 2), height + (border_thickness * 2)), (255, 255, 255)) # white background (snake)

	teal_tile = Image.new("RGB", (width, height), (190, 235, 247)) # rgb(190, 235, 247) - weak teal 
	green_tile = Image.new("RGB", (width, height), (160, 217, 149)) # rgb(160, 217, 149) - dampened green
	blue_tile = Image.new("RGB", (width, height), (76, 172, 188)) # rgb(76, 172, 188) - greeny-blue
	yellow_tile = Image.new("RGB", (width, height), (246, 227, 137)) # rgb(246, 227, 197) - beige-yellow
	grey_tile = Image.new("RGB", (width, height), (183, 183, 183)) # rgb(183, 183, 183) - slate grey
	pink_tile = Image.new("RGB", (width, height), (214, 152, 135)) # rgb(214, 152, 135) - soft pink
	black_tile = Image.new("RGB", (width, height), (0, 0, 0)) # empty, so black square
	snake_tile = Image.new("RGB", (width, height), (245, 7, 83)) # snake section, so vibrant square - rgb(245, 7, 83)

	mapping = {
		"P": green_tile,
		"M": grey_tile,
		"S": yellow_tile,
		"C": pink_tile,
		"R": blue_tile,
		"G": teal_tile,
		"£": black_tile
		# * (player) and + (weapon) are in path
	}

	output_frames = []
	for frame in gif_frame_queue:
		map_state = frame.popleft()
		max_width = len(map_state[0])
		image_width = int((max_width * (width + (border_thickness * 2))) + (border_thickness * 2))
		image_height = int((len(map_state) * (height + (border_thickness * 2))) + (border_thickness * 2))

		frame_to_save = Image.new("RGB", (image_width, image_height), (0, 0, 0))

		for y, row in enumerate(map_state):
			for x, cell in enumerate(row):
				if((y, x) not in frame):
					tile_to_add = cell_border.copy()
					tile_to_add.paste(mapping[cell], (border_thickness, border_thickness))
					

				else:
					tile_to_add = snake_border.copy()
					tile_to_add.paste(snake_tile, (border_thickness, border_thickness))

				frame_to_save.paste(tile_to_add, box=((border_thickness + (x * (width + border_thickness * 2))), border_thickness + (y * (height + border_thickness * 2))))

		output_frames.append(frame_to_save)

	image_to_save = output_frames[0]
	if(len(output_frames) > 1):
		output_frames = output_frames[1:]

	image_to_save.save("images/snake_" + str(image_number) + ".gif", append_images=output_frames, save_all=True, duration=200, loop=0)

```

Potential customisation options include:
- Changing the `duration` (in ms) that each frame stays visible for (longer duration = slower gif).
- Changing the number of `loop`s that the gif runs for. 0 is to infinitely loop (as I have used in my example), 1 will stop after the first play through, 2 will stop after the second, etc.
- Changing the colours of the tiles. I have gone with a fairly decent representation of the terrain, in my example.
- Changing the colour of the snake.

As a fun challenge: Try to randomise your snakes' colours, and give them a cute face

An optional QoL step, that I have found useful, is to clear the `images` directory on impossible generation. This is my new except, for `UnboundLocalError`:
```python
import os

# ...

except UnboundLocalError:
	get_request("regenerate")
	global image_number
	image_number = 1
	for remove_number in range(1, 101):
		try:
			os.remove("images/snake_" + str(remove_number) + ".gif")
		except OSError:
			break

	last_response = {}
```


## The code, as promised

294 lines of pure beauty /s
It is slightly improved upon, from what I used in the event (namely the impossible generation resolver), but I tested it in the `After Party` event and it works a charm.
This code does not include the snake visual (as it is for the solve itself).

You will need to edit the `ip_port`, to match your docker spawn, and `snake_number` can be changed to give more or less attempts to find a correct path for that weapon (which may not exist), but here it is:
```python
# Standard imports, required for the solution
import requests
import copy
import json
from collections import deque
import sys


def post_request(uri, data):
	"""sends a post request to '[host]:[port]/[uri]', with the supplied data"""
	headers={'Content-type':'application/json', 'Accept':'application/json'}
	return requests.post("http://" + ip_port + "/" + uri, data=data, headers=headers)


def get_request(uri):
	"""sends a get request to '[host]:[port]/[uri]'"""
	return requests.get("http://" + ip_port + "/" + uri)


def get_map_state():
	"""generates the current map state, returning the map grid and list of weapon coords"""
	map_state = []
	weapons = []

	for y in range(map_height):
		row = []
		for x in range(map_width):
			current_coord = "(" + str(x) + ", " + str(y) + ")" # same as str(tuple(x,y))
			coord = "" # initiate coord as empty string

			current_tile = map_tiles[current_coord]
			tile_terrain = current_tile["terrain"]

			if(player_position == [x, y]):
				coord = player_icon
			elif(current_tile["has_weapon"] == True):
				coord = weapon_icon
				weapons.append([x, y])
			else:
				if(tile_terrain == "E"):
					coord = empty_icon
				else:
					coord = tile_terrain

			row.append(coord)
		map_state.append(row)

	return map_state, weapons


def print_map(map_state):
	"""prints the provided map state"""
	for row in map_state:
		print(" ".join(row))
		print()


def BFS(map_state):
	"""Breadth-First Search"""
	start = player_position[1], player_position[0]
	queue = deque()
	queue.append(start)
	visited_tiles = set()

	parents = dict()
	parents[start] = None
	do_break = False
	while queue:
		
		tile = queue.popleft()
		if(map_state[tile[0]][tile[1]] == '+'):
			break # found weapon
		else:
			visited_tiles.add(tile)
			for neighbour in neighbours(map_state, tile):
				if neighbour not in visited_tiles:
					parents[neighbour] = tile
					queue.append(neighbour)

	path = list()
	while tile != None:
		path.append(tile)
		tile = parents[tile]

	path = path[::-1]
	return path


def neighbours(map_state, tile):
	"""gets the next valid neighbour"""
	modifiers = [(0, 1), (0, -1), (1, 0), (-1, 0)]
	returned_coords = list()
	for modifier in modifiers:
		newr = modifier[0] + tile[0]
		newc = modifier[1] + tile[1]
		if(newr < 0 or newr >= len(map_state) or newc < 0 or newc >= len(map_state[0])):
			continue

		direction = ""

		if(modifier == (0, 1)):
			direction = "R"
		elif(modifier == (0, -1)):
			direction = "L"
		elif(modifier == (1, 0)):
			direction = "D"
		elif(modifier == (-1, 0)):
			direction = "U"

		current_tile_terrain = map_tiles["(" + str(newc) + ", " + str(newr) + ")"]["terrain"]
		if(current_tile_terrain in ["C", "G"]):
			if(current_tile_terrain == "C" and direction in ["L","U"]):
				continue
			elif(current_tile_terrain == "G" and direction in ["R","D"]):
				continue

		if(map_state[newr][newc] == empty_icon):
			continue

		returned_coords.append((newr, newc))
	return returned_coords


def get_path_cost(path):
	"""calculates how expensive a path is"""
	output_directions = []
	returned_cost = 0
	movement_costs = {("P", "M"): 5, ("M", "P"): 2, ("P", "S"): 2, ("S", "P"): 2, ("P", "R"): 5, ("R", "P"): 5, ("M", "S"): 5, ("S", "M"): 7, ("M", "R"): 8, ("R", "M"): 10, ("S", "R"): 8, ("R", "S"): 6}
	for i in range(len(path)-1):
		current_tile_terrain = map_tiles["(" + str(path[i][1]) + ", " + str(path[i][0]) + ")"]["terrain"]
		next_tile_terrain = map_tiles["(" + str(path[i+1][1]) + ", " + str(path[i+1][0]) + ")"]["terrain"]

		direction = ""

		if(path[i+1][0] < path[i][0]):
			direction = "U"
		elif(path[i+1][0] > path[i][0]):
			direction = "D"
		elif(path[i+1][1] < path[i][1]):
			direction = "L"
		elif(path[i+1][1] > path[i][1]):
			direction = "R"

		output_directions.append(direction)

		if(current_tile_terrain == "G" or next_tile_terrain == "G"):
			if(next_tile_terrain == "G" and direction in ["R","D"]): # should never be true, at this stage, but just in case
				print("invalid path")
				returned_cost = "invalid"
				break
			else:
				returned_cost += 1
		elif(current_tile_terrain == "C" or next_tile_terrain == "C"):
			if(next_tile_terrain == "C" and direction in ["L","U"]): # should never be true, at this stage, but just in case
				print("invalid path")
				returned_cost = "invalid"
				break
			else:
				returned_cost += 1
		elif(current_tile_terrain == next_tile_terrain):
			returned_cost += 1
		else:
			movement_cost = movement_costs[(current_tile_terrain, next_tile_terrain)]

			returned_cost += movement_cost

	return returned_cost, output_directions


def rerun_until_cheap(queue_of_paths, all_tested_states, cheap_path_found, total_recursion):
	"""SNAKES"""
	queue_to_loop = queue_of_paths.popleft() # get the 'player to weapon' queue

	map_state = queue_to_loop.popleft() # get the passed map_state, from the queue
	path_cost, output_directions = get_path_cost(queue_to_loop)

	if(path_cost > remaining_time):
		for each_unpathable in queue_to_loop:
			if((each_unpathable != queue_to_loop[0]) and (each_unpathable != queue_to_loop[-1])):
				map_state_copy = copy.deepcopy(map_state)
				map_state_copy[each_unpathable[0]][each_unpathable[1]] = empty_icon
				test_map_state = tuple(tuple(a) for a in map_state_copy) # create a tuple form of the map_State, to be stored in the set

				if(test_map_state not in all_tested_states): # make sure map_state has not already been tested (speeds up the search and stops infinite loop)
					new_path = BFS(map_state_copy) # perform the new shortest path search
					if(new_path[-1] == tuple(queue_to_loop[-1])):
						path_cost, output_directions = get_path_cost(new_path)

						if(path_cost > remaining_time):
							new_path_to_append = deque(new_path)
							new_path_to_append.appendleft(map_state_copy)
							queue_of_paths.append(new_path_to_append)
							all_tested_states.add(test_map_state)

						else:
							cheap_path_found = True
							break
	else:
		new_path = queue_to_loop
		cheap_path_found = True

	if(cheap_path_found):
		cheap_path = new_path
	else:
		if(len(queue_of_paths) > 0):
			total_recursion += 1
			if(total_recursion < snake_number):
				cheap_path, path_cost, output_directions = rerun_until_cheap(queue_of_paths, all_tested_states, cheap_path_found, total_recursion)
			else:
				return False, False, False

		else:
			cheap_path = False
			path_cost = False

	return cheap_path, path_cost, output_directions


def post_directions_to_server(directions_to_upload):
	"""sends the directions to the server, to complete the level"""
	for each_direction in directions_to_upload:
		data = {"direction": each_direction}
		response = post_request("update", json.dumps(data))
	return response.json()


def main():
	"""program spine"""
	current_map = post_request("map", "").json()
	with open("map.json", "w") as map_json: 
		json.dump(current_map, map_json)
	with open("map.json", "r") as map_json: # avoid race conditions by closing and re-opening the file 
		current_map = json.load(map_json)

	# store relevant data in global variables
	global map_tiles, map_height, map_width, player_position, remaining_time
	map_tiles = current_map["tiles"]
	map_height = current_map["height"]
	map_width = current_map["width"]
	player_position = current_map["player"]["position"]
	remaining_time = current_map["player"]["time"]

	map_state, weapons = get_map_state()

	cheap_path = False
	for weapon_to_use in weapons:
		weapons_to_remove = copy.deepcopy(weapons)
		weapons_to_remove.remove(weapon_to_use)

		new_map_state = copy.deepcopy(map_state)
		for each_weapon in weapons_to_remove:
			new_map_state[each_weapon[1]][each_weapon[0]] = empty_icon

		path_for_weapon = BFS(new_map_state)

		queue_of_paths = deque()
		if(path_for_weapon[-1] == tuple(weapon_to_use[::-1])):

			path_to_append = deque(path_for_weapon)
			path_to_append.appendleft(new_map_state)
			queue_of_paths.appendleft(path_to_append)
			all_tested_states = set()

			working_path_and_cost = rerun_until_cheap(queue_of_paths, all_tested_states, False, 0)
			if(working_path_and_cost[0] != False and len(working_path_and_cost[0]) > 1):
				cheap_path = working_path_and_cost[0]
				path_cost = working_path_and_cost[1]
				output_directions = working_path_and_cost[2]
				break

	try:
		last_response = post_directions_to_server(output_directions)
	except UnboundLocalError:
		get_request("regenerate")
		last_response = {}
	return last_response


if __name__ == "__main__":

	ip_port = "83.136.250.41:45025" # changes depending on docker spawn

	# store global constants
	player_icon = "*"
	weapon_icon = "+"
	empty_icon = "£"
	snake_number = 1000
	sys.setrecursionlimit(1000+snake_number)

	last_response = {}
	while "flag" not in last_response:
		last_response = main()
		print(last_response) # prints {'maps_solved': <number>, 'solved': <bool>}
	print(last_response["flag"])

```
