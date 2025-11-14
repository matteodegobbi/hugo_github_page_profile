+++
date = '2024-05-18T01:27:57+02:00'
draft = false 
title = 'Solving the Vehicle Routing Problem with genetic algorithms 🧬'
categories = ['Algorithms']
tags=['python','genetic algorithms']
image="/misc/vrp.jpg"
+++

## Capacitated vehicle routing problem
This semester during the artificial intelligence course I developed a programming project to learn more about genetic algorithms.

I set out to solve the capacitated vehicle routing problem, which is an extenstion of the traveling salesman problem where there are multiple salesmen/trucks that need to deliver some products from a storage facility to clients.

Each client requests some amount of goods and the trucks have a maximum capacity of goods they can transport. Also the routes of the trucks need to not have any client in common. The objective is to determine the minimum number of trucks necessary to satisfy all the clients and find their routes such that the total number of kilometers travelled is minimized.

This is a well known problem in the field of operations research and is very important for the logistics of transportation.

Finding the optimal solution is hard especially for big numbers of clients, even a number of clients in the hundreds makes finding the optimal solution using ILP solvers infeasible. For this reason heuristic approaches are often used to solve instances of this problem, in particular some options are:
- Tabu Search
- Simulated Annealing
- Iterated Local Search
- Genetic Algorithm

I decided to use genetic algorithms, I used as genetic operators:
- Partially mapped crossover
- Alternating edge crossover
- Elitism, meaning the best chromosome always survives)
- Random mutation, randomly swapping two genes with a small probability

I was able to find good solutions close to the optimal, and in general much better solutions than a greedy approach that always selects the closest client as the next stop for the truck.

Also to increase performance of the algorithm I used the library numba which uses a jit compiler to optimize functions when using type hints.

<figure>
  <figcaption>Comparison of a bad and good solution to a VRP instance</figcaption>
  <img src="/misc/bad_vrp.jpg" style="width:500px;" alt="Image 1">
  <img src="/misc/good_vrp.jpg" style="width:500px;" alt="Image 1">
</figure>
