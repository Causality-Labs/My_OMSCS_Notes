
#### Performance

- Performance usually refers to the speed of the processor. 
- Speed can be broken down into two aspects of speed: 
	- Latency: how much time does it take to go from the start to the finish 
	- Throughput: how many tasks can be completed in one time unit. 
- Throughput is NOT always equal to 1/Latency

#### Comparing Performance

- Performance can be compared using speedup. Speedup is defined as “X is N times faster than Y”. Speedup: 
	- N = Speed(X) / Speed(Y) 
	- = Throughput(X) / Throughput(Y) 
	- = Latency(Y) / Latency(X) 
- Note the difference when using Latency Speedup < 1 : The performance has gotten worse with the newer version. The degradation is because of worse performance. 
- Speedup > 1: The performance has gotten better with the newer version. The improvement is through shorter execution time or with Higher Throughput. Performance is proportional to 1/latency


#### Speedup

- Speedup > 1
	- Improved Performance
	* Shorter execution time
	* Higher throughput

- Speedup < 1 
	- Worse Performance
	- Performance ~ Throughput
	
- Performance ~ 1/Latency

#### Measuring Performance

How do we measure performance?
Actual User Workload:
* Many programs
* Not representative of other users
* How do we get workload data?

Instead, use Benchmarks

#### Benchmarks
- Benchmarks are standard tasks for measuring processor performance.
- A benchmark is a suite of programs that represent common tasks.


#### Types of Benchmarks

* Real Applications
  * Most Representative
  * Most difficult to set up
* Kernels
  * Find most time-consuming part of applicatio
  * May not be easy to run depending on development phase
  * Best to run once a prototype machine is available
* Synthetic Benchmarks
  * Behave similarly to kernels but simpler to compile
  * Typically good for design studies to choose from design
* Peak Performance
  * In theory, how many IPS
  * Typically only good for marketing

#### Benchmark Standards

- A benchmarking organization takes input from academia, user groups, and manufacturers, and curates a standard benchmark suite. Examples:
	- TPC (databases, web), 
	- EEMBC (embedded), 
	- SPEC (engineering workstations, raw processors). 
		- For example, SPEC includes GCC, Perl, BZIP, and more.


#### Summarizing Performance

![[Pasted image 20260117075130.png]]Summarize performance using the average execution time. If Speedup is compared to Speedup; the Geometric mean, NOT the average, must be used. Geometric mean = (Product of the terms) 1/Number of terms


####  Iron Law of Performance

**CPU Time** = (# instructions in the program) * (cycles per instruction) * (clock cycle time)
Each component allows us to think about the computer architecture and how it can be changed:
* Instructions in the Program
  * Algorithm
  * Compiler
  * Instruction Set
* Cycles Per Instruction
  * Instruction Set
  * Processor Design
* Clock Cycle Time
  * Processor Design
  * Circuit Design
  * Transistor Physics

#### Iron Law for Unequal Instruction Times

![[Pasted image 20260117080826.png]]CPU Time = [Sum of(Inst/Program * cycles/Inst)] * Time/cycle


#### Amdahl's Law

- Used for measuring speedup when only a fraction of the system is improved. 
- Speedup = 1/((1­Frac of Enhancement) + (Frac Enhancement/Speedup Enhancement))
- Frac of Enhancement = % of original execution time that is affected by enhancement.
![[Pasted image 20260117083900.png]]


####  Amdal's Law Implications

Use Amdahl’s law to determine which design decisions will yield the best system. Make the Common Case Fast : small improvements for a large percentage of the system are better than large improvements on a small percentage of the system.
![[Pasted image 20260117084322.png]]

#### Lhadma's law
* Amdahl: Make common case fast
* Lhadma: Do not mess up the uncommon case too badly

#### Diminishing  Returns
![[Pasted image 20260117085139.png]]Consequence of Amdahl's law. If you keep trying to improve the same area, you get diminishing returns on the effort. Always reconsider what is now the dominant part of the execution time.