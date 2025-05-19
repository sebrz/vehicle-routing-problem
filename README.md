# Vehicle Routing Problem (VRP)

## Problem Description

A delivery company has a fleet of vehicles with limited capacity and must deliver packages to a set of customers located at different addresses. All vehicles depart from a single depot and must return to it after completing their route.

Each customer must be visited exactly once by a single vehicle, and the number of packages transported by a vehicle must not exceed its capacity.

## Objective

Design routes for all vehicles in such a way that the total distance traveled is minimized, while meeting all capacity and assignment constraints.

## Interpretation

The problem is defined by the following entities:

- **Customers**: Geographic locations on the map that require the delivery of one or more packages, represented by their `demand` characteristic. The distance between customers is calculated using the Euclidean distance between their coordinates.

- **Vehicles**: Agents that travel through the geographic space, departing from a depot located at the center of the map, and returning to it after delivering all packages. Each vehicle has a capacity (provided by file for the *small* case, variable for the *medium* case, and with a value of 100 for the *large* case). To satisfy the demand of customers on its route, the sum of each customer's demands on the route cannot exceed the vehicle's capacity.

- **Routes**: The sequence of stops on the map that each vehicle follows. It is necessary for each route to start and end at the depot.

## Data Structures

The implementation uses the following key data structures to represent the VRP problem:

- **Client and Depot Representation**: Each client and the depot are represented as dictionaries with the following keys:
  - `ID`: A unique identifier (0 for depot, positive integers for clients)
  - `X`, `Y`: Coordinates in the 2D plane
  - `Demand`: Number of packages to be delivered (always 0 for the depot)

- **Vehicle Representation**: Each vehicle is represented as a dictionary with:
  - `Vehicle_ID`: A unique identifier
  - `Capacity`: Maximum load the vehicle can carry

- **Solution Representation**: A solution is represented as a dictionary where:
  - Keys are vehicle IDs
  - Values are ordered lists of node IDs (route), always starting and ending with 0 (depot)
  
- **Client Map**: For efficient access, a lookup dictionary maps client IDs to their corresponding data structures.

In the implemented algorithms, these data structures facilitate:
1. Route distance calculations using Euclidean distance
2. Capacity constraint validations
3. Solution modifications during the optimization process (destruction and repair operations in LNS)
4. Visualization of routes on the map

## Approach

The objective of the problem is to optimize the routes that a fleet of vehicles will travel in order to satisfy all customer demands in each case. For small and medium cases, the problem is not too complex, but for large cases, it becomes more difficult to thoroughly explore the solution space to reach the global minimum. Our goal is to try to minimize the total distance by using a combination of algorithms and heuristics that are capable of optimizing at the local level (restructuring routes) as well as at the global level (iterating with the objective of exchanging segments between routes). Part of the challenge of this problem is due to the constraint given by each vehicle's capacity.

In order to optimize the *Large* case, it was decided to initialize the number of vehicles and start filling the route using a greedy algorithm. This generated very inefficient routes, so we tried to complement it with 2-opt. But this did not yield good results since 2-opt is a local search algorithm, so if the routes are poorly developed from their initialization, 2-opt will not be able to significantly reduce the distance. Due to this, the following method was chosen: clustering was performed using K-nearest neighbors to define groups of customers that are close to each other as part of a route. Subsequently, Large Neighborhood Search was applied followed by crossover exchange in order to optimize globally. Once this was done, 2-opt and backward-tracking were used as methods to optimize particular routes.

## Methodology

The three cases that appear in the notebook are solved as follows:

### Small and Medium VRP

**Initial Heuristic Solution:** First, an initial feasible solution is generated heuristically (a "base" solution). In the code, customers are evenly distributed among the available vehicles and converge at the node that represents the depot. A local 2-opt optimization is applied to each initial route to shorten the circuit of that vehicle. The result is a valid solution where each vehicle makes a journey from the depot, visits a certain subset of customers, and returns to the depot. This initial solution is not optimal but serves as a starting point.

**Application of Metaheuristic with Large Neighborhood Search:** On the current solution, the LNS algorithm iteratively performs the following key steps:

1. Partial destruction: A subset of customers is randomly selected (for example, for small VRP it was 30% and for medium VRP it was 20% of the customers in the implementation) and removed from their current routes. This leaves gaps in some routes.

2. Repair: Next, the removed customers are reinserted into the solution. The reinsertion is done in a greedy manner, choosing for each removed customer the route and position that increases the total distance as little as possible. The capacity of each vehicle is respected when relocating customers, and if a customer does not fit into any existing route, a new route can be opened (using an available unassigned vehicle).

3. After reinserting all customers, 2-opt is applied again to each route to internally refine the route and ensure that no optimizations are pending. This destroy/repair process is equivalent to exploring a large neighborhood of the current solution: new assignments of several customers are tested simultaneously, something that simple local searches (like 2-opt alone) cannot achieve because they only move a pair of edges.

4. Acceptance and memory: If the repaired solution obtained has a lower total distance than the current solution, then it is accepted as the new current solution (replacing the previous one). If it also improves the best solution found so far, this historical optimum is updated. If the new solution does not improve, it can be discarded or taken with a certain probability; in the presented implementation, a strict greedy criterion was chosen (movements are only accepted if they improve the distance) and the number of iterations without improvement is counted. The algorithm iterates this destroy-and-repair cycle multiple times (up to a predefined maximum, e.g., 200 iterations, or until there are 50 iterations without improvement).

### Large VRP

For the case of the largest problem, a multi-phase optimization approach was developed after determining that initialization with a greedy algorithm generated inefficient routes that could not be adequately improved with local techniques such as 2-opt. The implemented methodology consists of three main phases:

**1. Initialization through clustering:** K-means was used to group geographically close customers, thus creating more coherent initial routes that serve as a starting point for subsequent optimizations. This approach provides a logical spatial structure to the initial solutions.

**2. Global optimization in two stages:**

   - Large Neighborhood Search (LNS): Allows exploration of the solution space through partial destruction and reconstruction of routes, facilitating the reassignment of customers between different vehicles to escape local optima.

   - Cross-Exchange: Complements LNS by exchanging complete segments between routes, allowing restructuring that LNS cannot perform. This technique achieves additional reductions of 5-10% in total distance while maintaining vehicle capacity constraints.

**3. Local refinement:**

   - 2-opt Algorithm: Restructures segments within each individual route, eliminating inefficient crossings.

   - Backtracking detection and correction: Identifies patterns where a vehicle unnecessarily returns over already traveled routes, especially valuable in scenarios with high customer density.

This hierarchical combination of techniques allows optimization at both global and local levels, achieving improvements in solution quality at each step that would not be possible with a single or simple sequential approach.

## Conclusions

- The notebook solution employs heuristics (2-opt, greedy insertion) for local decisions, wrapped in an LNS metaheuristic that guides the global search by partially breaking down and reconstructing the solution.

- No genetic algorithm, ant colony, or other population-based method is used; the approach is closer to a specialized iterated local search for VRP. Nor is a classic exact method (such as integer linear programming) applied, due to the aforementioned complexity of VRP for larger sizes.

- In the larger VRP, where initially perhaps not all vehicles were fully loaded, the LNS algorithm managed to have some vehicles cover all deliveries, leaving empty routes and spare vehicles. For example, if up to 8 vehicles were allowed but the demand could be satisfied with 7, the algorithm effectively left 1 vehicle without a route, using only 7 and thus reducing the fleet. This means benefits for the company in practice, as it implies fewer vehicles on the road and lower operating costs.

- K-Means was used as an initial clustering method to spatially group customers and thus create more coherent base routes. This improves the quality of initial solutions, reduces the work of optimization algorithms, and helps methods like LNS or 2-opt converge toward more efficient solutions.