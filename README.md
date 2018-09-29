- [Interesting Fuzzing](#interesting-fuzzing)
    - [Coverage-based Greybox Fuzzing as Markov Chain(CCS 16)](#coverage-based-greybox-fuzzing-as-markov-chainccs-16)
    - [T-Fuzz: fuzzing by program transformation(S&P 18)](#t-fuzz-fuzzing-by-program-transformationsp-18)
    - [CollAFL: Path Sensitive Fuzzing(S&P 18)](#collafl-path-sensitive-fuzzingsp-18)
    - [Driller: Argumenting Fuzzing Through Selective Symbolic Execution(ndss 16)](#driller-argumenting-fuzzing-through-selective-symbolic-executionndss-16)
    - [VUzzer: Application-aware Evolutionary Fuzzing(ndss 17)](#vuzzer-application-aware-evolutionary-fuzzingndss-17)

- [Directed Fuzzing](#directed-fuzzing)
    - [Directed Greybox Fuzzing(CCS 17)](#directed-greybox-fuzzingccs-17)
    - [Hawkeye: Towards a Desired Directed Grey-box Fuzzer(CCS 18)](#hawkeye-towards-a-desired-directed-grey-box-fuzzerccs-18)

- [Fuzzing Machine Learning Model](#fuzzing-machine-learning-model)
    - [TensorFuzz: Debugging Neural Networks with Coverage-Guided Fuzzing(18)](#tensorfuzz-debugging-neural-networks-with-coverage-guided-fuzzing18)
    - [Coverage-Guided Fuzzing for Deep Neural Networks](#coverage-guided-fuzzing-for-deep-neural-networks18)

# Interesting Fuzzing

## Coverage-based Greybox Fuzzing as Markov Chain(CCS 16)

* [paper](https://github.com/bin2415/fuzzing_paper/tree/master/paper/CCS16_aflfast.pdf)

- Search Strategy
- Power Schedule
- 通过改变前面两个方法来使程序更大概率地走到low-density region.

## T-Fuzz: fuzzing by program transformation(S&P 18)

* [paper](https://github.com/bin2415/fuzzing_paper/tree/master/paper/oakland18_T-Fuzz.pdf)

- Fuzzer: T-Fuzz uses an existing coverage guided fuzzer to generate inputs. T-Fuzz depends on the fuzzer to keep track of the paths taken by all the generated inputs and realtime status infomation regarding whether it is "stuck". As output, the fuzzer produces all the generated inputs. Any identified crashing inputs are recorded for further anlysis.
- Program Transformer: When the fuzzer gets "stuck", T-Fuzz invokes its Program Transformer to generate tranformed programs. Using the inputs generated by the fuzzer, the Program Transformer first traces the program under test to detect the NCC candidates and then transforms copies of the program by removing certain detected NCC candidates.
- Crash Analyzer: For crashing inputs found against the transformed programs, the Crash Analyser filters false positives using a symbolic-execution based analysis technique.

### T-Fuzz Design

- Detecting NCCs: NCCs are those sanity checks which are present in the program logic to filter some orthogonal data, e.g., the check for a magic value in the decompressor example above. NCCs can be removed without triggering spurious bugs as they are not intended to prevent bugs. This paper uses a lightweight method to find the NCCs. Firstly, they define the concept of boundary edges: the edges connecting the nodes that were covered by the fuzzer-generated inputs and those that were not. The method that find the NCCs in this paper is over-approximation, so they find two ways to prune undesired NCC condidates.
- Program Transformation: After finding NCCs, T-Fuzz should "remove" the NCCs conditions to guide the execution to the another branch. T-Fuzz transforms programs by replacing the detected NCC candidates with negated conditional jump.
- Filtering out False Positives and Reproducing Bugs: As the removed NCC candidates might be meaningful guards in the original program(as opposed to, e.g., magic number checks), removing detected NCC edges might introduce new bugs in the transformed program. Consequently, T-Fuzz's Crash Analyzer verifies that each bug in the transformaed program is also present in the original proram, thus filtering out false positives. The Crash Analyser uses a transformation-aware combination of the preconstrained tracing technique leveraged by Driller and the Path Kneading techniques proposed by ShellSwap to collect path constraints of the original program by tracing the program path leading to a crash in the transformed program.

## CollAFL: Path Sensitive Fuzzing(S&P 18)

* [paper](https://github.com/bin2415/fuzzing_paper/tree/master/paper/oakland18_collafl.pdf)

该paper主要对AFL有两个改进:

- AFL是coverage-based greybox fuzzing，它通过对源程序进行轻量级的插桩，来跟踪每次fuzzing的input覆盖哪些路径，然后将路径hash，从而判断每个input是否到达了一个新的路径，如果到达新的路径，则说明该input较好，将该input作为seed。但由于hash可能会发生collision，可能会导致某些input到达新的路径，却没有将该input作为seed。该paper主要针对这一点，采用了一个新的算法，解决了路径hash collision问题，产生的效果也是比较显著的。
- 提供了一些策略来将seed进行排序，促使fuzzer去探索没有到达的路径。具体做法就是如果某条路径有很多没有探索到的邻居分支，则对该input进行更多的变异；如果某条路径有很多没有探索到的邻居后代，则对该input产生更多的变异。还有一个策略来帮助发现更多的漏洞：如果某条路径进行更多的内存访问，则对该input产生更多的变异。

我个人认为，该论文的主要贡献是提供了一个机制来解决路径的hash collision问题，使得coverage判断更加准确。

### AFL Coverage Measurements

AFL使用bitmap(默认64KB)来跟踪edge coverage。没一个字节都对应特定edge的hit count。AFL通过对每个basic block进行插桩，为每个basic block都随机分配一个id，当执行每条路径时，对该路径上的每个basic block都进行如下操作:

```c
cur_location= <COMPILE_TIME_RANDOM>;
shared_mem[cur_location ^ prev_location]++;
prev_location = cur_location >> 1;
```
其中上面的prev_location右移一位主要是为了区分路径A->B和B->A。由于每个basic block的id是随机分配的，所以这种hash方法很容易产生collision，特别当程序比较大的时候，collision rate也越大。

### CollAFL's Solution to Hash Collision

CollAFL通过三种方式来解决hash collision:

1.  ![公式1](./image/s1.png)
    通过贪心算法，为每个basic block分配x和y的值，保证每条edge计算的hash值都是不同的。
2. 如果每个basic block只有一个前继basic block，即只有一条边到达该basic block，所以只需要将该basic block的id来表示该edge即可。
3. 如果前面两种方法无法解决，则动态的时候为每条边分配不同的id。


## Driller: Argumenting Fuzzing Through Selective Symbolic Execution(ndss 16)

* [paper](https://github.com/bin2415/fuzzing_paper/tree/master/paper/NDSS16_driller.pdf)

我们都知道，fuzzing对于一些比较宽松的限制(比如x>0)能够很容易的通过变异产生一些输入达到该条件；而symbolic execution非常擅长求解一下magic value(比如x == deadleaf)。这是一篇比较经典的将concolic execution和fuzzing结合在一起的文章，该文章的主要思想就是先用AFL等Fuzzer根据seed进行变异，来测试程序。当产生的输入一直走某些路径，并没有探测到新的路径时，此时就"stuck"了。这时，就是用concolic execution来产生输入，保证该输入能走到一些新的分支。从而利用concolic execution来辅助fuzz。

## VUzzer: Application-aware Evolutionary Fuzzing(ndss 17)

* [paper](https://github.com/bin2415/fuzzing_paper/tree/master/paper/ndss17_vuzzer.pdf)

Vuzzer是公认的比较好的类AFL fuzzer。它主要利用Data-flow features和Control-flow features来辅助fuzzer变异和进行seed的选择。

### Data-flow features

利用dynamic taint analysis 来推断input的结构和类型，以及某段数据在input的偏移。比如，它通过对每个cmp指令进行插桩来判断input的哪些字节与输入有关，并且知道与它比较的另外一个值。同时，Vuzzer也可以对lea指令进行插桩，从而检测*index*操作是不是与input某些bytes有关。

### Control-flow features

Control-flow features可以让Vuzzer推断出执行路径的重要性。比如，某些执行路径最后到达了*error-hanling blocks*。Vuzzer就通过静态的方法识别出了一下*error-handling code*。同时，Vuzzer通过对每个basic block赋予特定的权重，来促使fuzzer走到更深的路径中去。 

# Directed Fuzzing

## Directed Greybox Fuzzing(CCS 17)

* [paper](https://github.com/bin2415/fuzzing_paper/tree/master/paper/CCS17_aflgo.pdf)

```c
Input: Seed Input S

repeat
    s = CHOOSENEXT(S)
    p = ASSIGNENERGY(s)    //This paper focus
    for i from 1 to p do
        s' = MUTATE_INPUT(s)
        if t' crashes then
            add s' to Sx
        else if ISINTERESTING(s') then
            add s' to S
        end if
    end for
until timeout reached or abort-signal

Output: Crashing Inputs Sx
```

类AFL的fuzzing一般步骤如上所示，该paper主要关注于ASSIGNENERGY(s)这一操作，他们通过对不同的seed s赋予不同的energy，即如果一个seed s'产生的trace距离目标基本块targetB较近，则其energy(p)就较大，基于种子s'进行的变异操作就会变多。所以该paper主要有两个contributation: 设计一套算法计算seed s'产生的trace与targetB的距离；通过模拟退火算法来为每个seed s分配energy。

## Hawkeye: Towards a Desired Directed Grey-box Fuzzer(CCS 18)

* [paper](https://github.com/bin2415/fuzzing_paper/tree/master/paper/ccs18_hawkeye.pdf)

### Desired Properties of Directed Fuzzing

- P1. The DGF should define a **robust** distance-based mechanism that can guide the directed fuzzing by avoiding the bias to some traces and considering all traces to the targets.
- P2. The DGF should strike a balance between overheads and utilities in static analysis.
- P3. The DGF should select and schedule the seeds to rapidly reach target sites. AFL determines how many new inputs(i.e., "energy") should be generated from a seed input to improve the fuzzing effectiveness(i.e., increase the coverage); this is termed "power scheduling".
- P4. The DGF should adopt an adaptive mutation strategy when the seeds cover the different program states. The desired design is that when a seed has already reached the target sites(including target lines, basic blocks or functions), it should be given less chances for coarse-grained mutations(e.g., chunk replacement).

### AFLGo's Solution

- 针对P1，AFLGo只是选择路径最短的那条，然而路径最短的那条可能无法触发某个漏洞。
- For P2. AFLGo only considers the explicit call graph information. As a result, function pointers are treated as the external nodes which are ignored during distance calculation. Besides, AFLGo counts the same callee in tis callers only once, and it does not differentiate multiple call patterns between the caller and callee.
- For P3. AFLGo applies a simulated annealing based power scheduler: it favors those seeds closer to the targets by assigning more energy to them to be mutated; the applied cooling sechedule initially assigns smaller weight on the effecte of "distance guidance", until it reaches the "exploitation" phrase. The issue is that there is no prioritization procedure so the newly generated seeds with smaller distance may wait for a long to be mutated.
- For P4. The mutation operators of AFLGo come from AFL's two non-deterministic strategies: 1) havoc, which does purely randomly mutations such as bit flips, bytewise replace, etc; 2) splice, which generates seeds from some random byte parts of two existing seeds. Notably, during runtime AFLGo excludes all the deterministic mutation procedures and relies purely on the power scheduling on havoc/splice strategies.

### Suggestions to improve DGFs:

- For P1, a more accurate distance definition is needed to retain trace diversity, avoiding the focus on short traces.
- For P2, both direct and indirect calls need to be analyzed; various call patterns need to be distinguished during static distance calculation.
- For P3, a moderation to the current power scheduling is required. The distance-guided seed prioritization is also needed.
- For P4, the DGF needs an adaptive mutation strategy, which optimally applies the fine-grained abd ciarse-graubed nytatuibs wgeb tge dustabce between the seed to the targets is different.

### Hawkeye's Design

![overview](https://github.com/bin2415/fuzzing_paper/blob/master/image/s1.png)


# Fuzzing Machine Learning Model

## TensorFuzz: Debugging Neural Networks with Coverage-Guided Fuzzing(18)

* [paper](https://github.com/bin2415/fuzzing_paper/tree/master/paper/tensorfuzz.pdf)

## Coverage-Guided Fuzzing for Deep Neural Networks(18)

* [paper](https://github.com/bin2415/fuzzing_paper/blob/master/paper/18_coverage-guided-fuzzing-for-deep-neural-networks.pdf)