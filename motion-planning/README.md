# **Project: 3D Motion Planning**
___
## **Task 1:** Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  

You're reading it! Below I describe how I addressed each rubric point and where in my code each point is handled.
This file contains four sections:
1. Task 1: Provide a Writeup / README.
2. Task 2: Explain the Starter Code. 
3. Task 3: Implementing Your Path Planning Algorithm.
4. BONUS IMPLEMENTATIONS: Playing with Deadzones, Playing with Vehicle Heading, Medial Axis, Voronoi, Probabilistic Roadmap
___

## **Task 2:** Explain the Starter Code.

The goal here is to explain the functionality of what's provided in `motion_planning.py` and `planning_utils.py`. I will divide the task in parts A and B, one for each file.

### **Part A:** Motion Planning Script.

The `motion_planning.py` script is divided in four main parts:

1. Imports.
2. States Class.
3. Motion Planning Class.
4. Code Evaluation.

#### A.1. Imports.
This part is self-explanatory. It imports all packages necessary for sound running of the code. 

#### A.2. States Class.
This part defines a class that inherits from `enum` class and enumerates possible states the drone can be in,  automatically associating constant values to them, and making possible to use them as symbolical variables.

#### A.3. Motion Planning Class.
This part defines a class that inherits from `Drone` class. It is subdivided in the following blocks:
1. Constructor Function.
2. Callback Functions.
3. Transition Functions.
4. Path Planning Function.
5. Start Function.

##### Constructor Function.
This block overwrites some of the superclass properties and register callbacks for each of the MsgIDs.

##### Callback Functions.
This block defines through `local_position_callback`, `velocity_callback` and `state_callback` how to react to different events using transition functions. One should note that the deadband, which permits transition between target positions, is defined here in `local_position_callback`. Other similar parameters that permits transition between states are also in this block.

##### Transition Functions.
This block defines each of the transition functions (`arming_transition`, `takeoff_transition`, `waypoint_transition`, `landing_transition`, `disarming_transition` and `manual_transition`) which command actions and change the vehicle's flight state.

##### Path Planning Function.
This block defines the path (sequence of waypoints) that the vehicle should follow. It receives target altitude, local position and goal position and, by processing obstacles data and safety distance from them, defines a safe path to reach goal position. It also send these waypoint coordinates to the simulator, in order to visualize them.

##### Start Function.
This block starts a log file, starts the API connection and, after mission completion, stops the log file.

### A.4. Code Evaluation.
This part of the code just runs everything.

___
### **Part B:** Planning Utilities Script
The `planning_utils.py` script is divided in four main parts:
1. Imports.
2. Create Grid Function.
3. Action Class and Valid Actions Function.
4. A* Functions.

#### B.1. Imports.
This part is self-explanatory. It imports all packages necessary for sound running of the code. 

#### B.2. Create Grid Function.
This function, `create_grid` creates a 2D grid with ones and zeros, where one are obstacles. The function asks for `data`, `drone_altitude` and `safety_distance`. Obstacle position is provided by `data`. The altitude in which drone is going to operate is given by `drone_altitude` and will help set obstacles in the grid only if their height is greater than `drone_altitude`. Finally, the obstacles in the grid are set greater than they actually are, to provide extra safety.  This offset is given by `safety_distance` value.

#### B.3. Action Class and Valid Actions Function.
The planning algorithm (A*) needs information on how the vehicle will move in the grid and this is provided by `Action` class. This class inherits from `Enum` class and sets possible movements: west, east, north, south (and diagonals). The class also sets properties `cost` and `delta`, which access the values of the movement. The function `valid_actions` checks if the movement that will be used by A* will not leave the grid boundaries or hit obstacles. If any of this consequences happens, the action is removed from `valid_actions` list.

#### B.4. A* Function.

The planning algorithm (A*) writes a list `path` of waypoints from `start` position to `goal` position. For each point in the path, beginning from `start` position, the algorithm adds valid motions and moves to the direction of lesser `queue_cost`. The `queue_cost` is composed by two costs: the first is `actions_cost` (that can be one to lateral motions and square root of 2 for diagonal motions)  and the other is the `heuristic_cost` that is the distance from `next_node`, i.e. node after motion, to the `goal`. This makes the algorithm to prefer paths that try to reach the goal as fast as possible.

___
## **Task 3:** Implementing Your Path Planning Algorithm.

### 1. Set your global home position
read the first line of the csv file, extract lat0 and lon0 as floating point values.
```
with open("colliders.csv") as f:
    lat_str, lon_str = f.readline().split(',')
    lat0, lon0 = float(lat_str.split(' ')[-1]), float(lon_str.split(' ')[-1])
    print(lat0, lon0)
```

use the self.set_home_position() method to set `global_home`
```
self.set_home_position(lon0, lat0, 0)
```

### 2. Set your current local position
Global position can be obtained by the GPS and barometer readings of the drone. A local variable `global_position` is then set to:
```
global_position = (self._longitude, self._latitude, self._altitude)
```
Now, `local_position` is obtained using `global_to_local()` that calculates NED coordinates by feeding `global_position` relative to `global_home`.
```
local_position = global_to_local(global_position, self.global_home)
```

### 3. Set grid start position from local position
Defining `start` position as `local_position`. To run the start point in A*, we need to change coordiantes to account for the grid offset. Thus, `start_goal` is simply obtained by taking first two elements of the goal array, subtracting offset values from the grid and transforming into integers and putting in a tuple.
```
start = local_position
grid_start = (int(start[0] - north_offset), int(start[1] - east_offset))
```
### 4. Set grid goal position from geodetic coords
Any latitude and longitude within the map can be chosen as goal, for this task I chose:
```
goal_lat = 37.794760
goal_lon = -122.401120
```
Now `goal` is obtained through `global_to_local()` as an array containing NED coordinates from a tuple containing EFEC coordinates, using `global_home` position as reference.
```
goal = global_to_local((goal_lon, goal_lat, 0), self.global_home)
```
Again, to run the goal in A*, we need to change coordiantes to account for the grid offset. Therefore, `grid_goal` is simply obtained by:
```
grid_goal = (int(goal[0] - north_offset), int(goal[1] - east_offset))
```
### 5. Modify A* to include diagonal motion (or replace A* altogether)
Before running A*, we need to modify `Action(Enum)` class and `valid_actions()` function. Regarding to `Action(Enum)` class, it is necessary to add new types of motion. Diagonal motion can be added using the following tuples:
```
NORTH_WEST = (-1, -1, np.sqrt(2))
SOUTH_EAST = (1, 1, np.sqrt(2))
NORTH_EAST = (-1, 1, np.sqrt(2))
SOUTH_WEST = (1, -1, np.sqrt(2))
```
These tuples give combined motion and add cost relative to diagonal length of a square of side equal to one. Adding constraints in `valid_actions()` function, as shown in code lines below, include checking if the combined motion do not hit obstacles or leave the grid boundaries.
```
if (x - 1 < 0 or y - 1 < 0) or grid[x - 1, y - 1] == 1:
    valid_actions.remove(Action.NORTH_WEST)
if (x + 1 > n or y + 1 > m) or grid[x + 1, y + 1] == 1:
    valid_actions.remove(Action.SOUTH_EAST)
if (x - 1 < 0 or y + 1 > m) or grid[x - 1, y + 1] == 1:
    valid_actions.remove(Action.NORTH_EAST)
if (x + 1 > n or y - 1 < 0) or grid[x + 1, y - 1] == 1:
    valid_actions.remove(Action.SOUTH_WEST)
```
The planned path is generated using the A* for grids (given). It returns a list of tuple waypoints `path`.
```
path, _ = a_star(grid, heuristic, grid_start, grid_goal)
```
### 6. Cull waypoints 

The A* algorithm returns `path`, but the list of waypoints that come from the A* algorithm will often have unnecessary waypoints, either too close to each other, or collinear. To cull the waypoints, I have developed two functions, one for each method: collinearity test and ray tracing (Bresenham). The `path` can be fed to `prune_path()` for collinearity test or `bres_pruned_path`.
```
pruned_path = prune_path(path)
```
#### Collinearity pruning
The `prune_path()` function access two consecutive points to the i-th point in the path and check their collinearity (using `collinearity_check()` function shown below). If they are collinear, the algorithm removes the middle one, it keeps checking the two consecutive points to the i-th point until no collinearity is found and the reference point is iterated.

```
def prune_path(path):
    pruned_path = [p for p in path]
    i = 0
    while i < np.shape(pruned_path)[0] - 2:
        p1 = pruned_path[i]
        p2 = pruned_path[i+1]
        p3 = pruned_path[i+2]
        if collinearity_check(p1 ,p2 , p3):
            pruned_path.remove(pruned_path[i+1])
        else:
            i += 1
    return pruned_path
pruned_path = prune_path(path)
```
The function `collinearity_check` checks the area of the triangle formed by three points. If the area is sufficiently close to zero, i.e. below a threshold `epsilon`, the function returns that the points are collinear. Function is given by:
```
def collinearity_check(p1, p2, p3, epsilon=1e-6):
    m = np.concatenate((point(p1), point(p2), point(p3)), 0)
    det = np.linalg.det(m)
    return abs(det) < epsilon
```
where `point` is:
```
def point(p):
    return np.array([p[0], p[1], 1.]).reshape(1, -1)
```
#### Ray-tracing method: Bresenham pruning

The `bresenham_prune_path()` function access two consecutive points to the i-th point in the path. In this, Bresenham between the i-th and the (i+2)-th point is run and, if all squares in the grid given by the algorithm, are free from collision, the (i+1)-th waypoint can be removed. This is iterated through waypoints list.

```
def bresenham_prune_path(grid, path):
   pruned_path = [p for p in path]
   i = 0
   while i < np.shape(pruned_path)[0] - 2:
       p1 = pruned_path[i]
       p3 = pruned_path[i+2]
       if all((grid[pp] == 0) for pp in bresenham(int(p1[0]), int(p1[1]), int(p3[0]), int(p3[1]))):
           pruned_path.remove(pruned_path[i+1])
       else:
           i += 1
   return pruned_path
```
P.S. Thanks to Mike Hahn in Slack forum I find out about the use of function `all()`.

### 7. Execute the flight

Test configuration:
Global home latitude and longitude = (37.79248, -122.39745)
Global position = (-122.3974501, 37.7924788, 0.089)
Local position = (-0.17767039,  0.27920842, -0.08998074)
North offset = -316, east offset = -445
Local Start and Goal in Grid:  (315, 445) (566, 120)

Here's a picture of my planned path before pruning:

![Planned path](./images/planned_path.png)
Number of waypoints:  469

After pruning using collinearity:

![Pruned planned path](./images/pruned_planned_path.png)
Number of waypoints (collinear pruning):  27

After pruning using ray tracing (Bresenham):

![Bresenham pruned planned path](./images/bres_pruned_planned_path.png)
Number of waypoints (Bresenham pruning):  13

Second starting position:

![Second path: From Embarcadero](./images/embarcadero_route.png)

Third starting position:

![Third path: From South Neighborhood](./images/south_route.png)


## **BONUS IMPLEMENTATIONS** 

### Playing with Deadzones
A deadzone is the region near each waypoint where the vehicle feels satified that it reached the target and changes its aim to the next one. Trying different values of deadzone, I understood that deadzones too large will make the vehicle overshoot and loosely follow the its path. If the deadzone is too small, the vehicle will almost stop from waypoint to waypoint. From trial and error, I found that deadzone of radius 2 fits well the problem. Other types of transition can lead to better performance: combining position and velocity, or combining position and turning angle to the next waypoint.
```
def local_position_callback(self):
    # Set radius for deadzone
    DEAD_ZONE = 2.0
    # State checking
    if self.flight_state == States.TAKEOFF:
        if -1.0 * self.local_position[2] > 0.95 * self.target_position[2]:
            self.waypoint_transition()
    elif self.flight_state == States.WAYPOINT:
        if np.linalg.norm(self.target_position[0:2] - self.local_position[0:2]) < DEAD_ZONE:
            if len(self.waypoints) > 0:
                self.waypoint_transition()
            else:
                if np.linalg.norm(self.local_velocity[0:2]) < 0.2:
                    self.landing_transition()
```

### Playing with Vehicle Heading
The vehicle can perform a more organic path is vehicle heading related to the waypoints position is added. To do so, the positions of the waypoints and be considered and `arctan2(delta_y/delta_x)` can return the needed yaw angle, as shown in code lines below. It seems, from my experiments, that vehicle heading also makes the vehicle overshoot less.
```
# Convert path to waypoints
waypoints = [[p[0] + north_offset, p[1] + east_offset, TARGET_ALTITUDE, 0] for p in pruned_path]
# Give heading values related to trajectory
for i in range(len(waypoints)-1):
    wp1 = waypoints[i]
    wp2 = waypoints[i+1]
    waypoints[i+1][3] = np.arctan2((wp2[1]-wp1[1]), (wp2[0]-wp1[0]))

# Set self.waypoints
self.waypoints = waypoints
# Send waypoints to sim (this is just for visualization of waypoints)
self.send_waypoints()
```

### Representation: Medial Axis Graph
To check this implementation out, instead of `motion_planning.py`, try `motion_planning_medial.py`. 

These lines are added in `plan_path` function.
```
start = local_position
grid_start = (int(start[0] - north_offset), int(start[1] - east_offset))

goal_lat = 37.794760
goal_lon = -122.401120
goal = global_to_local((goal_lon, goal_lat, 0), self.global_home)
grid_goal = (int(goal[0] - north_offset), int(goal[1] - east_offset))

skeleton = create_skeleton(grid)

skel_start, skel_goal = find_start_goal(skeleton, grid_start, grid_goal)
path, _ = a_star(invert(skeleton).astype(np.int), heuristic, skel_start, skel_goal)
```

Prints on console for test configuration:
```
Global home: (-122.39745, 37.79248, 0.0).
Global position: (-122.3974483, 37.7924799, 0.001).
Local position: (-0.0012076348066329956, 0.14484423398971558, -0.001501128077507019).
North offset = -316.
East offset = -445.
Grid Start: (315, 445).
Grid Goal: (566, 120).
Skel Start: (315, 445).
Skel Goal: (558, 128).
Found a path.
Number of waypoints before pruning: 524.
After Bresenham pruning: 7.
```
Medial Axis skeleton on grid and planned path:

![Medial Axis](./images/medial_axis_path.png)

Medial Axis skeleton on grid and planned path after being pruned:

![Pruned Medial Axis](./images/medial_axis_pruned_path.png)

### Representation: Voronoi Graph
To check this implementation out, instead of `motion_planning.py`, try `motion_planning_voronoi.py`.

These lines are added in `plan_path` function.
```
grid, edges, north_offset, east_offset = create_grid_and_edges(data, TARGET_ALTITUDE, SAFETY_DISTANCE)
graph = create_voronoi_graph(edges)

start = local_position
grid_start = (int(start[0] - north_offset), int(start[1] - east_offset))

goal_lat = 37.794760
goal_lon = -122.401120
goal = global_to_local((goal_lon, goal_lat, 0), self.global_home)
grid_goal = (int(goal[0] - north_offset), int(goal[1] - east_offset))

graph_start, graph_goal = find_start_goal(graph, grid_start, grid_goal)
path, _ = a_star(graph, heuristic, graph_start, graph_goal)
```
Prints on console for test configuration:
```
Global home: (-122.3974533, 37.7924804, 0.0).
Global position: (-122.39745, 37.792479, 0.024).
Local position: (-0.15377381443977356, 0.2881147265434265, -0.024175215512514114).
North offset = -316.
East offset = -445.
Grid Start: (315, 445).
Grid Goal: (566, 120).
Graph Start: (315.76114, 445.76846).
Graph Goal: (560.7610999999999, 130.76850000000002).
Found a path.
Number of waypoints before pruning: 52.
After Bresenham pruning: 8.
```

Voronoi graph and planned path:

![Voronoi](./images/voronoi.png)

Voronoi graph and planned path after being pruned:

![Pruned Voronoi](./images/pruned_voronoi.png)

### Representation: Probabilistic Roadmap Graph
To check this implementation out, instead of `motion_planning.py`, try `motion_planning_rsampling.py`.

These lines are added in `plan_path` function.
```

```
Prints on console for test configuration:
```
Global home: (-122.39745, 37.79248, 0.0).
Global position: (-122.3974511, 37.7924796, -0.019).
Local position: (-0.03972350433468819, -0.09788312762975693, 0.019598715007305145).
North offset = -316.
East offset = -445.
Grid Start: (315, 444, 5).
Grid Goal: (566, 120, 5).
Generated 140 / 200 samples so far
Generated 183 / 200 samples so far
Generated 193 / 200 samples so far
Generated 198 / 200 samples so far
Generated 200 / 200 samples so far
Number of nodes: 200.
Max number of edges per node: 10.
Graph Start: (287, 446, 6.208723009096124).
Graph Goal: (567, 120, 5.289101735346839).
Found a path.
Path: [(-0.04500405536964536, -0.09656390140298754, 5), (-28.537348617635132, 1.75240810447076, 6.208723009096124), (59.7473236992779, -2.8256173266967153, 8.73485667727002), (139.99117880511096, -65.74029960974826, 9.687161965279437), (130.3167590317591, -162.75795670713052, 8.95070053154198), (87.7451164096725, -275.91196171610443, 6.162469413438005), (179.20587578687815, -300.1736065571356, 8.553345748298783), (290.46385991636674, -308.937895445065, 7.8072570205953955), (250.88918597903103, -324.74694366264157, 5)]
```
Probabilistic Roadmap graph with 200 nodes and planned path:

![Probabilistic Roadmap](./images/prob_roadmap.png)
