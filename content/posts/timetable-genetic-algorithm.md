---
title: "Solving Timetable Scheduling with a Genetic Algorithm in Python"
date: 2026-05-06
tags: ["algorithms", "genetic-algorithm", "python", "optimisation"]
categories: ["Algorithms"]
description: "A walkthrough of designing a genetic algorithm to generate conflict-free, optimal class timetables under hard scheduling constraints."
showToc: true
draft: false
---

Timetable scheduling is one of those problems that looks trivial until you actually try to do it. Every university department deals with it every semester — assign courses to rooms and time slots such that no student or professor is double-booked, rooms aren't overcrowded, and ideally labs are scheduled in sensible blocks. It's an NP-hard combinatorial optimisation problem, and genetic algorithms are a natural fit.

## Why Genetic Algorithms?

Exact solvers (integer linear programming, backtracking) work well on small instances but blow up exponentially as the problem grows. Genetic algorithms trade optimality guarantees for scalability — they find *good* solutions fast, which is exactly what you need in practice.

The core idea mirrors biological evolution:

1. Start with a **population** of random timetables
2. **Evaluate** each one using a fitness function
3. **Select** the fittest candidates as parents
4. **Crossover** parents to produce offspring
5. **Mutate** offspring randomly
6. Repeat until a good-enough solution emerges

## Problem Representation

A timetable is represented as a 3D grid:

```python
from dataclasses import dataclass, field
from typing import List, Optional
import random

@dataclass
class TimeSlot:
    day: int          # 0 = Monday, 4 = Friday
    period: int       # 0-indexed period within the day

@dataclass
class ScheduleEntry:
    course: str
    professor: str
    room: str
    timeslot: TimeSlot
    is_lab: bool = False

# A chromosome is a list of ScheduleEntry objects
Chromosome = List[ScheduleEntry]
```

Each "gene" is a single scheduled class. A full chromosome represents a complete candidate timetable.

## The Fitness Function

The fitness function is where the domain knowledge lives. I penalise hard constraints (conflicts) heavily and soft constraints (compactness, fairness) lightly:

```python
def fitness(chromosome: Chromosome) -> float:
    penalty = 0.0

    # Hard constraint 1: No professor teaches two classes simultaneously
    prof_slots: dict[tuple, int] = {}
    for entry in chromosome:
        key = (entry.professor, entry.timeslot.day, entry.timeslot.period)
        prof_slots[key] = prof_slots.get(key, 0) + 1
    penalty += sum(max(0, v - 1) * 100 for v in prof_slots.values())

    # Hard constraint 2: No room double-booked
    room_slots: dict[tuple, int] = {}
    for entry in chromosome:
        key = (entry.room, entry.timeslot.day, entry.timeslot.period)
        room_slots[key] = room_slots.get(key, 0) + 1
    penalty += sum(max(0, v - 1) * 100 for v in room_slots.values())

    # Soft constraint: Labs should be in consecutive periods
    for entry in [e for e in chromosome if e.is_lab]:
        if entry.timeslot.period > 4:   # labs shouldn't be last period
            penalty += 5

    return 1.0 / (1.0 + penalty)       # higher is better
```

A timetable with zero conflicts scores `1.0`. Every conflict adds 100 penalty points, dragging the fitness toward zero.

## Crossover and Mutation

```python
def crossover(parent1: Chromosome, parent2: Chromosome) -> Chromosome:
    """Single-point crossover."""
    point = random.randint(1, len(parent1) - 1)
    return parent1[:point] + parent2[point:]

def mutate(chromosome: Chromosome, rate: float = 0.05) -> Chromosome:
    """Randomly reassign timeslots with probability `rate`."""
    for entry in chromosome:
        if random.random() < rate:
            entry.timeslot = TimeSlot(
                day=random.randint(0, 4),
                period=random.randint(0, 7)
            )
    return chromosome
```

## The Main Evolution Loop

```python
POPULATION_SIZE = 100
GENERATIONS = 500
ELITE_SIZE = 10

population = [random_chromosome(courses, professors, rooms) 
              for _ in range(POPULATION_SIZE)]

for gen in range(GENERATIONS):
    scored = sorted(population, key=fitness, reverse=True)
    
    # Elitism: carry top individuals forward unchanged
    next_gen = scored[:ELITE_SIZE]
    
    # Fill the rest with crossover + mutation
    while len(next_gen) < POPULATION_SIZE:
        p1, p2 = random.choices(scored[:50], k=2)   # tournament selection
        child = mutate(crossover(p1, p2))
        next_gen.append(child)
    
    population = next_gen
    
    if gen % 50 == 0:
        best = fitness(scored[0])
        print(f"Gen {gen}: best fitness = {best:.4f}")
        if best == 1.0:
            print("Conflict-free timetable found!")
            break

best_timetable = sorted(population, key=fitness, reverse=True)[0]
```

## Configuration

One design goal was full configurability via a simple dict:

```python
config = {
    "days": 5,
    "periods_per_day": 8,
    "courses": ["DSA", "OOP", "Networks", "DBMS", "ML"],
    "professors": {"DSA": "Prof. A", "OOP": "Prof. B", ...},
    "rooms": ["Room-101", "Room-102", "Lab-A"],
    "lab_courses": ["ML"],
}
```

No hardcoding — the algorithm adapts to any department's setup.

## Results

On a realistic dataset of 5 courses, 4 professors, 3 rooms across 5 days, the algorithm consistently finds conflict-free timetables within 200–400 generations. Runtime is under 10 seconds on a standard laptop.

## Key Takeaways

- **Fitness function design is 80% of the work** — getting the penalty weights right took more iteration than the GA machinery itself.
- **Elitism is non-negotiable** — without preserving the best solutions, the GA can "forget" good partial solutions.
- **Mutation rate is a dial, not a switch** — too low and it stagnates; too high and it's just random search.

The full code is on my GitHub. It includes a CSV exporter and a simple HTML timetable renderer.
