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
  * Thrashing Protection to avoid next-line prefetches for thrashing IPs
  * Prefetcher throttling (IP-independent) for Global Streaming class of IPs to increase prefetching accuracy
  * Prefetcher throttling (IP-independent) for all classes of IPs to increase prefetching accuracy
  * Reducing the GS prefetch degree as compared to original IPCP details.
  * Implemented a debugging output for analysing IP access behaviour. 

## L2C level prefetching
* In **`ipcp.l2c_pref`**, the following changes were thought of, and implemented :
  * Thrashing Protection to avoid next-line prefetches for thrashing IPs
  * Supressing the GS prefetch degree as compared to original IPCP details.
  * Implemented a debugging output for analysing IP's memory access behaviour. 
  

The files `analyseL1d.ipynb` and `analyseL2c.ipynb` are used to analyse and plot the memory access behaviour of various IPs.
  
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


