# Go Memory Management

## Analysing 
1. Benchmark (component memory analysing of heap allocations, memory and cpu usage)
- Run benchmarks with `go test -bench=.`.
- To see heap allocations use:
    - `b.ReportAllocs()` function;
    - `benchmem` flag: `go test -bench=. -benchmem`.
- For memory profile use:
    - `-memprofile` flag: `go test -bench=. -memprofile mem.out`;
    - add heap profile server with file writting (see Profiler)
- For cpu profile use:
    - `-cpuprofile` flag: `go test -bench=. -cpuprofile cpu.out`;
    - add cpu profile server with file writting (see Profiler)

2. Runtime Statistics (detailed memory stats to look for possible memory leaks)
- Use `runtime.ReadMemStats(&r)`, where `r` is `var r runtime.MemStats` (see Runtime Statistics)
- Investigate `runtime/mstats.go` fir additional information about stats fields.

3. Escape Analysis (identifies allocation on heap, function complexity and inlining)
- Use `go build -gcflags="-m" .` to enable escape analysis logs.
- Use additional `-m`: `-gcflags="-m -m"` to have higher analisys mode.

4. GC Trace
- Use `export GODEBUG=gctrace=1` and run app, or `GODEBUG=gctrace=1 go run main.go` (see GC Trace)

## Development
1. Avoid slices (`string`, `[]byte`)
2. Use `StringBuffer`/`StringBuilder` for string concatenations.
3. Avoid `string` as map key type, it is better to use int (~44% less memory usage).
4. Use pointers for big arrays.
5. Use sync.Pool for data that can be reused.
6. Align struct fields (use libs e.g. `https://github.com/essentialkaos/aligo`).
7. Use `runtime/debug.SetMemoryLimit` or `GOMEMLIMIT` env (Go 1.19, https://tip.golang.org/doc/go1.19).

### Profiler
1. Memory Profiler (with lib)
```go
import "github.com/pkg/profile"

defer profile.Start(profile.MemProfile, profile.ProfilePath(".")).Stop()
```

2. CPU Profiler (with lib)
```go
import "github.com/pkg/profile"

defer profile.Start(profile.CPUProfile, profile.ProfilePath(".")).Stop()
```

### Runtime Statistics
```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func main() {
	go func() {
		for {
			var r runtime.MemStats
			runtime.ReadMemStats(&r)
			fmt.Println("\nTime: ", time.Now())
			fmt.Println("Runtime MemStats Sys: ", r.Sys) // 
			fmt.Println("Runtime Heap Allocation: ", r.HeapAlloc)
			fmt.Println("Runtime Heap Idle: ", r.HeapIdle)
			fmt.Println("Runtime Heap In Use: ", r.HeapInuse)
			fmt.Println("Runtime Heap HeapObjects: ", r.HeapObjects)
			fmt.Println("Runtime Heap Released: ", r.HeapReleased)
			time.Sleep(1 * time.Second)
		}
	}()

	time.Sleep(1 * time.Second)
}
```

### GC Trace
```bash
gc 6 @48.155s 15%: 0.093+12360+0.32 ms clock,
0.18+7720/21356/3615+0.65 ms cpu, 11039->13278->6876 MB, 14183 MB goal, 8 P
```

- `@48.155s` since program start
- `15%` of time spent in GC since program start
- `0.093+12360+0.32 ms clock` stop-the-world (STW) sweep termination + concurrent mark and scan + and STW mark termination
- `0.18+7720/21356/3615+0.65 ms cpu` (GC performed in line with allocation), background GC time, and idle GC time:
    - `0.18` sweepTermCpu, 
    - `7720` gcController.assistTime, 
    - `21356` gcController.dedicatedMarkTime + gcController.fractionalMarkTime,
    - `3615` gcController.idleMarkTime, 
    - `0.65` markTermCpu
- `11039->13278->6876 MB` heap size at GC start, at GC end, and live heap
8 P number of processors used