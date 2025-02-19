# TSP Solver Go Implementation

This framework implements:

-   Various GA operators (tournament selection, crossover, mutation)
-   2-opt Local Search
-   Memetic algorithm with GX crossover ([Paper](https://wpmedia.wolfram.com/uploads/sites/13/2018/02/13-4-1.pdf))
-   Greedy heuristics

## How to Run

Implemented on Go version `1.12.5`.

### Installation of Go

For installation of Go, refer to https://golang.org/doc/install, or https://zetawiki.com/wiki/%EC%9A%B0%EB%B6%84%ED%88%AC_go_%EC%B5%9C%EC%8B%A0%EB%B2%84%EC%A0%84_%EC%84%A4%EC%B9%98

Clone this source code to the path `$HOME/go/src/github.com/greenmonn/tsp-go/`, or `$GOPATH/src/github.com/greenmonn/tsp-go/` if you set your custom `GOPATH`.

Then run `go get ./...` to get dependencies (The dependencies are just for tests)

### Build

```bash
..greenmonn/tsp-go $ cd main
..greenmonn/tsp-go $ go build
```

### Run

```
$ ./main -filename=bier127 -p=10 -f=10 -v=false
```

### Add Custom Problems from TSPLIB

Currently, it's only compatible with EUC2D, Symmetric TSP instances.

The package structure is shown as below.

```
.
├── container
├── graph
├── main
├── operator
├── problems
│   ├── a280.tsp
│   ├── bier127.tsp
│   ├── burma14.tsp
│   ├── burma8.tsp
│   ├── ch130.tsp
│   ├── ch150.tsp
│   ├── fl1400.tsp
│   ├── fl3795.tsp
│   ├── fnl4461.tsp
│   └── rl11849.tsp
├── solver
└── utils
```

If you want to add problems, just locate the `tsp` file from TSPLIB to `problems` folder.

### Use Binary Executables

You can just execute with the given binary without installing go, if you're using Linux/amd64 platform.

```bash
./solver_linux -filename=fl1400 -p=10 -f=100 -o=3 -v=false
```

### Options

1. `-filename`

    - default: rl11849
    - Reads the file in `problems/` directory. If you want to add new `.tsp` file, just add the file to the directory and give the filename without extension(`.tsp`) as the option

2. `-p`

    - default: 10
    - population number

3. `-f`

    - default: 100
    - fitness evaluations, same as generations in GA

4. `-o`

    - default: 2
    - optimization count, only needed if using iterative local search

5. `-v` (verbosity)
    - default: true
    - if true, the log is printed to stdout.
    - if false, log is saved to the file (log-filename.txt)

### Output

For the result, the path is saved as csv file, for example, `rl11849.csv`.

The distance of the path is printed to the console.

By setting `-v=false`, log is printed to a file, instead of stdout.

### Change Experiment Settings

The default function is implementation of Memetic Algorithm. Currently, Up to `N < 2000` instances are solved in feasible time (< 20mins), `N < 4000` instances are tested (< 1 hour) on 10 population, 10 fitness evaluations.

#### MA w/ 2-Optimization (fl1400.tsp)

    * Best known: 20127
    * 10 Population / 10 Generations =>
        - 20972.668601824516 (4% error)

#### MA w/ 2-Optimization (fl3795.tsp)

    * Best known: 28723
    * 10 Population / 10 Generations =>
        - 29565.121753298718 (2.9% error)

You can test diverse settings by changing function in `main.go`.

```go
func main() {
	parseArguments()

	rand.Seed(time.Now().UnixNano())

	if !printLog {
		fpLog, err := os.OpenFile("log-"+filename+".txt", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0666)
		if err != nil {
			panic(err)
		}
		defer fpLog.Close()

		log.SetOutput(fpLog)
	}

	graph.SetGraphFromFile("problems/" + filename + ".tsp")

	startTime := time.Now()

    // MAFromGreedyPopulation() => Change here to below line
	LocalSearchFromPartialGreedyTour()

	duration := time.Now().Sub(startTime)

	log.Println("Duration: ", duration)
}
```

## Memetic Algorithm

Memetic Algorithm solver is provided in the `solver` package in the Go implementation.

```go
func SolveMA(initialTours []*graph.Tour, populationNumber int, generations int, optimizeGap int) *graph.Tour {
    // Memetic Algorithm (GA + Local Search)

    N := populationNumber
    var population *Population

    if len(initialTours) == 0 {
        population = NewRandomPopulation(N)
    } else {
        population = NewPopulation(N, initialTours)
    }

    for i := 0; i < generations; i++ {
        log.Printf("\n%dth Generation\n", i+1)

        population = EvolvePopulation(population)

        // Optimize whole population: individuals would be 'near' local optimum
        if i%optimizeGap != 0 {
            continue
        }

        for _, tour := range population.Tours {
            operator.FastLocalSearchOptimize(tour)

        }
    }

    operator.LocalSearchOptimize(population.BestTour(), -1)

    return population.BestTour()
}

```

## Initial Population

The individual tour is represented as type `Tour` in this framework.

```go
type Tour struct {
    Path      []*Node
    Distance  float64
    Edges     map[string]*Edge
    FlexEdges []*Edge
}
```

For algorithms that use path-based representation, a `Tour` object maintains path that consists of list of nodes.

`Edges` and `FlexEdges` are only used for the Memetic Algorithm implementation based on GX crossover(will be introduced later).

First, random population is used for both GA and local optimization. random tours are generated by randomly shuffling nodes in path. However, starting from random population has some issues for both GA and local optimization.

-   for GA: with large N, each random individual has relatively very little portion to be preserved. I tried with several crossover operators such as Order Crossover and Edge Recombination Crossover, GA needed too many generations and doesn’t converge to the near-optimum solution. It is because the random population didn’t have many ‘good’ edges in the beginning, and the good edges can only be introduced by mutation and implicit mutation during the crossover(introducing foreign edges). So it is challenging to make the large TSP instances to build the near-optimum tour solely with GA operators.

    -   fl1400.tsp: (10 population, 100000 generations, 24min38s) 654753.9982867754

*   for Local Search: local search from a random individual, in this framework, 2-opt(Pairwise Edge Exchange) could approach to the local optimum in more feasible time. For N=1400 instance, the margin of error was < 10% of the known best solution. However, the global optimum is not guaranteed, and is depends on the ‘luck’ of randomly getting feasible initial tour.
    -   fl1400.tsp: (5814 generations, 56s) 22235.961790

### Partially Greedy Random Tour

Hence, I implemented Partially Greedy Random Tour, which is suggested in the [paper](<(https://wpmedia.wolfram.com/uploads/sites/13/2018/02/13-4-1.pdf)>). The paper also analyze TSP fitness landscape, which indicates the local optimums share about 75% among the edges in general cases. Hence, suboptimal solutions from greedy heuristics can be used as seeds.
However, instead of simply mutating the greedy tour, 1/4 of the edges are connected first in random manners, and the remaining edges are connected greedily. In detail,

-   Random connect: actually, it is not totally random but uses preliminarily calculated nearest neighbors information. The random unvisited node is selected and connected to the nearest neighbor(p=0.66) or the second nearest neighbor(p=0.33).

-   Greedy connect: for this phase (determining 3/4 of the edges), the remaining edges are stored in a priority queue and check the each edge can be added to tour in increasing distance. This greedy heuristic is known as slightly better than the nearest neighbor approach.

### Evaluation

The generated partially greedy tour was not too far optimum(error ~ 25%, N=1400) Hence, it significantly reduces time for local optimization.

However, simply applying GA to the greedy population didn’t work well. The initial tours were too good to explore larger space. By enabling elitism, the best initial tour tends to remain to the end.

## GA Operators

### Selector

With the [initial version in Python](https://github.com/greenmonn/tsp-solver), k-Tournament selection and ranking-based roulette selection are considered, but because k-Tournament selection was significantly better than roulette selection in the preliminary experiments, only k-Tournament selection is implemented as default.

### Crossover

4 crossover operators are explored:

1. Order Crossover
    - Choose two random points, split the tour into 2 sections and paste the slice of each parent to the each section. It preserves exact order of one parent, and the partial edges of the other parent.
    - The most basic operator, reach to the near-optimum answer for hundreds of cities
    - Fast reduction in the early phase of GA, but become very slow in later generations.
2. Edge Recombination Crossover
    - Prioritize edges present in parents
    - Preserve most edges both in the parents, and some edges only in one parent.
    - Better performance than order crossover for instances of N>1000
3. CX2 Crossover
    - Enhanced version of CX(Cycle Crossover), generates offspring from parents using cycles
    - CX2 Crossover: suggested in the paper **Genetic Algorithm for Traveling Salesman Problem with Modified Cycle Crossover Operator**
4. GX Crossover (Only compatible with Random Greedy Tour)
    - 4 Steps: use 3 parameters(cRate, nRate, iRate)
    -   1. Copy common edges according to cRate
    -   2. Insert new edges using nearest neighbor information (pre-calculated and saved) by nRate
    -   3. Inherit some edges by iRate
    -   4. Greedily append remaining edges (using edge’s priority queue)
    - Saves set of edges, and ‘non-fixed’ edges along with path information
    - fixed edges are common edges of parents
    - On local search, fixed edges are omitted and only ‘flexible’ edges are considered. It significantly lowers the cost of local search, since the number of flexible edges starts from about 1/5 of the all edges, and also decrease along the generations, so the complexity for local search goes down.

3 of them, Order, Edge Recombination, CX Crossover are implemented in Python version of implementation(which is an older one).

On the other hand, GX, Order, Edge Recombination Crossover are implemented in Go implementation.

### Mutation

2 mutation operators are provided:

1. Random swap of two nodes in the path
2. Switch two edges randomly

The second one was more complicated to implement (I used node’s connection links instead of manipulating path). However, switching edge mutations seems to be efficient because swapping two nodes changes up to 4 edges, and the each 2 pair of edges are neighboring edges - this can cause bias and limit exploration.

## Local Search Operators

I implemented basic 2-opt Local Search(Pairwise Edge Exchange). Although 3-opt, or k-opt local searches are known to show better performance, I chose 2-opt because of the the complexity of implementation, and the performance issue (exploring possible neighbors are O(N^3) for 3-opt).

However, for large instances (N>=10000), Local Search was very costly. For instance rl11849, 6 hours were taken to optimize greedy solution to local optimum solution.

1. Iterative Local Optimize: instead of continuing local optimization until no better neighbors are found, we can just swap edges by iterating over a path. I was inspired by the python greedy tsp-solver implementation((https://github.com/dmishin/tsp-solver/blob/master/tsp_solver/greedy.py)

```go
func Optimize(tour *graph.Tour) {
    // 2-opt pairwise exchange (iterate over a path)
    tour.UpdateConnections()

    N := graph.GetNodesCount()

    for i := 0; i < N; i++ {
        for j := i + 2; j < i+N-1; j++ {
            SwapTwoEdges(tour, i, j, true)
        }
    }
}
```

Surprisingly, this showed similar performance of original local search.

2. Local Search (Original): I implemented general local search using connections in nodes. `Node` type is similar to doubly-linked lists because it has pointers to the 2 connected nodes. It guarantees to exactly find 2-opt local optimum, but costly for the instances >= 10000.

```go
type Node struct {
    ID        int
    X         float64
    Y         float64
    Connected []*Node
    Degree    int
}
```

`Degree` is the number of connected nodes, and it should be 2 for all nodes in a valid path.

3. Reduced-Cost Local Search ignoring fixed edges
   By using the list of edges in the given tour, the cost of local search decrease because the number of considerable edges are reduced.

GX crossover passes list of edges and flexible edges, and the implementation of edge-based local search considers only the flexible edges to be swapped.

-   For example, (rl11849)
    -   Number of flexible edges for the first local search: 2000-3000
    -   Second local search:

## Implementation Issue

I switched to the Go implementation because of the computational efficiency. For very large instances `N >= 10000`, Incorporating Local Search with GA took infeasible amount of time.

I tried limiting iterations of local search, (not waiting until no neighbors are found) but there was a problem:

-   Some of the tours reach to the local optimum in shorter iterations-others are not. But it is possible that some tours can have better local optimum although it takes more iterations to approach there. Simply limiting iterations cause unexpected ‘race’ between tours, and the winning tour in the earlier generations tends to remain as best until the end. Determining local optimum early is not intended behavior, so I tried to find another way to reduce computational cost of local search.
