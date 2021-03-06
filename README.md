Pin-based Constant Execution Checker (Pin-based CEC)
====================================================

Pin-based Constant Execution Checker (Pin-based CEC) is a dynamic binary instrumentation tool that checks for non-constant execution and memory-access patterns while a program is running. It does this by using the Intel PIN framework to trace every instruction that a targeted subroutine executes, logging all instruction pointers and memory addresses that get accessed, and comparing logs across subroutine invocations to ensure a constant execution profile. The tool uses taint analysis to determine if the execution differences are secret-dependent, to cut down on false positives.

Dependencies
------------

- Linux
- python 3.5 or greater
- gcc/g++/make
- diff

Building
--------

1. Download Intel PIN from https://software.intel.com/en-us/articles/pin-a-binary-instrumentation-tool-downloads and extract it somewhere.

2. Clone the source and make sure git submodules are fetched by making the clone with

```bash
git clone --recurse-submodules https://github.com/intel/pin_based_cec
```

or

```bash
git clone https://github.com/intel/pin_based_cec
git submodule init
git submodule update
```

3. Use make to build the pintool

```bash
cd src
make PIN_ROOT=<pin top-level path>
```

Usage
-----

1. Create a test program that exercises the Function Under Test. For example, if you want to test a modular
exponentiation implementation, do RSA signatures in a loop.

1. Include include/pin_based_cec.h file in your test program.

1. Add calls to PinBasedCEC_MarkSecret(uint64_t addr, uint64_t size) to mark your secret data for taint analysis. It is recommended to mark the data as close to the Function Under Test as you can. See example/test.c in the repo for an example.

1. Add calls to PinBasedCEC_ClearSecrets() when you are done processing your secret data.

1. Run the test program instrumented with Pin-based CEC. See the OPTIONS section. Any traces of the Function Under Test that showed execution or memory-access differences will have logs stored in the trace/ folder, and a corresponding taint analysis in the taint/ folder. See example/Makefile for an example of how to run the Pin-based CEC pintool.

1. Run the src/post_process.py script to apply the taint analysis results to the traces. The script will output the addresses of execution differences that are tainted with secret data.

Example
-------

See the example/ for steps on how to use Pin-based CEC to detect a potential side-channel vulnerability in an AES key expansion implementation.

Options
-------

```text
<pin binary> -t src/obj-intel64/CECTraceTool.so -f <func name> -l <lut addr arg index> -A <csv list of allocs to align> -s <summary file name> -- <target program>

Options:
-s <summary file name>
   Name of summary output file containing overall pass/fail results.

-f <func name>
   Target routine to instrument. Multiple target routines can be specified by repeating the -f option and
   tracing will only begin once all target routines have been entered.

Experimental Options:
-A <comma-seperated list of labels>
   Treat all the allocation labels (such as "lmem3") in the list as 64-byte aligned. This is useful to
   reduce false-positives when a program makes cacheline-aligned accesses into a buffer that itself might
   not be aligned to a cacheline.

-m <yes/no>
   If yes, enable LUT marking in the trace log.

-n <lut function index>
   If LUT marking is enabled, use the target routine specified by the given index to get the LUT address.
   Which argument of this function is treated as the LUT address is specified by -l.

-l <lut addr arg index>
   If LUT marking is enabled, treat the argument with the given index as the LUT address. The function
   that this is applied to is specified by -n.
```

Post Process

To post process results, run the src/post_process.py script in the directory containing that trace/ and taint/ folders:

```bash
python3 post_process.py [--verbose] [--branch] <result file>
```

--verbose enables verbose output.

--branch enables tainted branch checking (experimental). Any time RIP becomes tainted, the address of the tainting instruction is flagged and output. This can be used to detect scenarios where a conditional branch depends on secret data even if no execution differences are observed in the traces (e.g. when only one side of the branch is taken during the observed executions of a program).

The results (address that have been flagged as non-constant between executions) will be stored in the result file.

NOTES / CAVEATS

- __Tail call elimination__
If the Function Under Test has been optimized with tail call elimination (and therefore does not have a closing RET instruction), PIN will not be able to detect the end of the function, and so the analysis will not function correctly. This caveat only applies to the Function Under Test itself (the function passed as a parameter to Pin-based CEC), not subfunctions or other functions called during the execution of the Function Under Test.
