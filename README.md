# IPCP Prefetcher Enhancement <!-- omit in toc -->
# Table of Contents <!-- omit in toc -->
- [Introduction](#introduction)
- [The Team](#the-team)
- [Running Champsim](#running-champsim)
  - [Compiling Champsim](#compiling-champsim)
  - [Running Champsim](#running-champsim-1)
  - [Champsim results](#champsim-results)
- [Implementation details](#implementation-details)
  - [L1D level prefetching](#l1d-level-prefetching)
  - [L2C level prefetching](#l2c-level-prefetching)
- [Improvement Achieved](#improvement-achieved)


# Introduction
Our Goal is to identify limitations of IPCP-Prefetcher (Courtesy : [DPC-3](https://dpc3.compas.cs.stonybrook.edu/?final_programs)) for graph traces and suggest enhancements. We use a modified version of Champsim (Courtesy : [ChampSim_Customized](https://github.com/coderaavan/ChampSim_Customized) from [coderaavan](https://github.com/coderaavan) ) to run our simulations and obtain insights.

ChampSim is a trace-based simulator for a microarchitecture study. You can sign up to the public mailing list by sending an empty mail to champsim+subscribe@googlegroups.com. Traces for the 3rd Data Prefetching Championship (DPC-3) can be found from here (https://dpc3.compas.cs.stonybrook.edu/?SW_IS).

# The Team
This is our team **Risk-5** which was formed as a part of [CS 305 : Computer Architecture](https://www.cse.iitb.ac.in/~biswa/courses/CS305/main.html) at IIT-Bombay.

| Student           | Roll No.  | Contribution                                                                                       |
| ----------------- | :-------: | -------------------------------------------------------------------------------------------------- |
| Chandrasekhar D   | 190050031 | Enhancing IPCP at L1d and L2c. Implementing code for the same.                                     |
| Akash Reddy G     | 190050038 | Enhancing IPCP at L1d and L2c. Implementing code for the same.                                     |
| Satwik M          | 190050107 | Proposed ideas for enhancing IPCP. Prepared README.                                                |
| Kranthi Chandra A | 190050022 | Understanding memory access for various IPs. Python implementation for making graphs for the same. |
| Vishwanth K       | 190050131 | Understanding IPCP and limitations for graph workloads. Prepared slides for presentation.          |


<!-- Ctrl + Shift + I  to format the table, on linux. Extension Markdown All in One
<!-- Alt  + Shift + I  to format the table, on windows. Extension Markdown All in One
-->

# Running Champsim
## Compiling Champsim

ChampSim takes many parameters: Branch predictor, prefetchers, replacement policies and number of cores. 


```

Usage : ./build_champsim.sh [branch_pred] [l1i_pref] [l1d_pref] [l2c_pref] [llc_pref] [itlb_pref] [dtlb_pref] [stlb_pref] [btb_repl] [l1i_repl] [l1d_repl] [l2c_repl] [llc_repl] [itlb_repl] [dtlb_repl] [stlb_repl] [num_core]

$ ./build_champsim.sh bimodal no ipcp ipcp ipcp no no no lru lru srrip srrip srrip lru lru lru 1

```
**Caution** : While running the above commands, your current working directory should be the Champsim folder. 



## Running Champsim

Execute `run_champsim.sh` with proper input arguments as mentioned below. 


```
Usage: ./run_champsim.sh [BINARY] [N_WARM] [N_SIM] [TRACE_DIR_IN_TRACER] [TRACE] [OPTION]

$ ./run_champsim.sh bimodal-no-ipcp-ipcp-ipcp-no-no-no-lru-lru-srrip-srrip-srrip-lru-lru-lru-1core 10 10 ../traces/ sssp-5.trace.gz

${BINARY}: ChampSim binary compiled by "build_champsim.sh" 
${N_WARM}: number of instructions for warmup (10 million)
${N_SIM}:  number of instructinos for detailed simulation (10 million)
${TRACE_DIR_IN_TRACER} : path to the trace folder, realtive path is fine (../traces)
${TRACE}: trace name 
${OPTION}: extra option for "-low_bandwidth" (src/main.cc)
```
Simulation results will be stored under "results_[N_SIM]M" as a form of "[TRACE]-[BINARY]-[OPTION].txt".<br> 

**Caution** : In case of running with same binary and same options, the output results will overwrite any existing results.

**Note** : Champsim simulator can also be used to carry out multi-core simulations. But we here discuss only single core simulations.

## Champsim results

ChampSim measures the IPC (Instruction Per Cycle) value as a performance metric.
There are many other useful metrics printed out at the end of simulation.



# Implementation details

Much of the implementation and has been done in `ipcp.l1d_pref` and `ipcp.l2c_pref` files present in the `prefetcher` directory inside Champsim.


## L1D level prefetching
* In **`ipcp.l1d_pref`**, the following changes were thought of, and implemented :
  * **Change**: Thrashing Protection to avoid next-line prefetches for streaming IPs<br/>
  **Motivation**: Streaming IP's have very less reuse. They kind of access the memory randomly. So any simple attempt to prefetch for such an IP, would only pollute the cache.<br/>
  **Implementation**: For a given IP, let the previously accessed address, and current requested address be M<sub>old</sub> and M<sub>curr</sub> respectively. Then we consider the request to be streaming if |M<sub>old</sub> - M<sub>curr</sub>| > 1page(i.e. 64 cache lines) <br/>
  
  * **Change**: Prefetcher throttling (IP-independent) for **only Global Streaming** class of IPs to increase prefetching accuracy<br/>
  **Motivation**: We've observed that one of the GS dominant traces(sssp) showing very low prefetch accuracy. One reason for this is because ipcp goes too aggressive when it finds a GS access which is not required for graph workloads.<br/>
  **Implementation**: We've used the `pf_useful` and `pf_useless` attributes of the cache to find the prefetch-accuracy. And then we increase the prefetch degree when the accuracy is high and decrease it when the accuracy is low. As a straight forward consequence of the implementation, our throttling is IP independent.<br/>

  * **Change**: Prefetcher throttling (IP-independent) for **all classes** of IPs to increase prefetching accuracy.<br/>
  **Motivation**: Same as above. We just used this as an alternative idea to only GS class throttling.<br/>
  **Implementation**: Same as above.<br/>
  **Note**: When we've tried both the `only GS throttling` and `al class throttling` ideas, the `only GS throttling` idea worked well. One reason is because the CS class which is based on 2-bit counter, takes time to update.<br/>
  **Example**: Stride: 1,1,1,1,2,2, ?. At the question mark, the CS prefetcher will still predict a stride = 1. But consider the case where true answer at ? is  = 2. Our CS prefetcher will miss it if its degree == 1. But if its degree >1 , it wouldn't miss this. **So in short, having slightly higher prefetch degrees for CS and CPLX prefetchers is **good**!!**.<br/>
  
  * **Change**: Reducing the GS prefetch degree as compared to original IPCP details.<br/>
  **Implementation**: We've **manually tuned** and reduced the GS-prefetch degree so as to increase the prefetcher accuracy.<br/>
  
  * **Change**: Implemented a debugging output for analysing IP access behaviour. This is a key component of our work. To turn on L1D debugging output we just need to uncomment out `#define AKASH_LOAD_PRINT` line in `ipcp.l1d_pref` file..

## L2C level prefetching
All the motivations described above for L1D apply exaclty for L2C as well.
* In **`ipcp.l2c_pref`**, the following changes were thought of, and implemented :
  * **Change**: Thrashing Protection to avoid next-line prefetches for thrashing IPs<br/>
  **Implementation**: Streaming IPs anyway won't belong to CS or CPLX or GS class. So only we need to avoid NL prefetching for those. So to implement this, we've not prefetched for any LOAD request using the `NL` prefetcher at L2C.<br/>
  
  * **Change**: Supressing the GS prefetch degree as compared to original IPCP details.<br/>
  
  * **Change**: Implemented a debugging output for analysing IP's memory access behaviour. To turn on L2C debugging output we just need to uncomment out `#define AKASH_LOAD_PRINT_L2` line in `ipcp.l2c_pref` file.<br/>
  
* **Caution**: Dont turn on the debugging at both L1D and L2C simultaneously.

The files [analyseL1D](./analyseL1D.ipynb) and [analyseL2C](./analyseL2C.ipynb) are used to analyse and plot the memory access behaviour of various IPs at L1D and L2C respectively using the debug outputs of the same. Just give the  right debug file path to these `ipynb` files and run all cells.
  
# Improvement Achieved

After iteratively upgrading and tweaking the prefetchers at L1D and L2C level, we managed to obtain the following:
	
<!-- Convert TSV to table.	Shift + Alt + T	Yes (only when selecting range) -->
| Traces      | NO IPCP | Org.  IPCP | % wrt NO IPCP | Enhcd. IPCP | % wrt no IPCP | % wrt Org. IPCP |
| :---------- | :------ | :--------- | :------------ | :---------- | :------------ | :-------------- |
| Bellmanford | 0.1193  | 0.1204     | 0.90          | 0.1245      | 4.33          | 3.41            |
| BFS         | 0.1528  | 0.1992     | 30.39         | 0.1850      | 21.09         | -7.14           |
| Components  | 0.1072  | 0.1044     | -2.65         | 0.1109      | 3.44          | 6.25            |
| MIS         | 0.0732  | 0.0800     | 9.20          | 0.0780      | 6.50          | -2.48           |
| sssp-5      | 0.2099  | 0.2368     | 12.82         | 0.2478      | 18.05         | 4.63            |

We achieved a average of 4.76% betterment over the original IPCP for 3 traces (Bellmanford, Components, sssp-5).
Two of the traces have a worser IPC when compared to org. IPCP prefetcher.

# Pros
* Robust to streaming IP’s both at L1D and L2C

* Can automatically set it’s prefetch degree according to the prefetch accuracy

* Improved L1D accuracy significantly.
 

# Cons/ Possible Improvements
* We've only explored the traces with single-core simulations. We need to explore them with multi-core as well

* Need to counter the decrease in IPC of BellmanFord trace. Almost -5% of the -7.14% was because of thrashing protection at L1D. That means there is some locality within the streaming IP as well which we in our implementation were not able to leverage

* Accuracy at L2C is less than 15% right now. It can be improved.


