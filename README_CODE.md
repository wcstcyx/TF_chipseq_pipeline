PROGRAMMING GUIDELINES FOR BDS PIPELINES
===============================================

### BigDataScript (BDS)

BigDataScript is a scripting language specialized for NGS pipelines/workflows. Basic functions and syntax for BDS are documented at <a href=http://pcingola.github.io/BigDataScript/bigDataScript_manual.html>here</a>. The grammar is quite similar to Java but there is no high-level data structure like classes and multi-dimensional arrays.


### BDS basics

You have basic variable types such as `int`, `real`, `string` and `bool`. You can omit declaration of those variables using `:=`. For example, `int n = 10` is equivalent to `n := 10`. Also, if you add a help context for a variable with `help`, it automatically becomes a command line parameter with a help context. For example, `bds pipeline.bds -num_rep 3 -callpeak macs2` and `bds pipeline.bds -h` shows help for all parameter variables.
```
int num_rep = 2		help # replicates (default: 2).
callpeak := "spp" 	help Peak caller definition (default: spp).
```

BDS also provides list types such as map `{}` and array `[]`. You can iterate over those variables with the following code:
```
int[] array = [1,2,3]

array.add(4)

for ( int n : array ) {
	...
}
for ( int i=0; i< array.size(); i++) { // C style
	n := array[i]
	...	
}

int{} map

map{"first"} = 1 	// adding element to a map
map{"second"} = 2

for ( int n : map ) {
	...
}

for ( string key : map.keys() ) {
	n := map{key}
	...
}
```


### BDS syntax (basics)

Before moving on to next sections, read carefully about BDS syntax particularly for `sys`, `task`, `wait` and `par` on <a href=http://pcingola.github.io/BigDataScript/bigDataScript_manual.html>here</a>. There are more advanced syntax such as `goal` and `dep` but currently they are not used for BDS pipelines in Kundaje group (I actually found thatn they are buggy when combined with `par`). Parallelization of tasks are implemented by only using `sys`, `task`, `wait` and `par`.

You can refer to a BDS variable in a shell with prefix $
```
var1 	:= "test.txt"
var2 	:= "$var.tmp"
var_3	:= "test2.txt"
var_4 	:= "$var_3"+"_tmp" 	// be careful when you use an underscore in a BDS variable name.
//var_4 := "$var_3_tmp" 	// error (var_3_tmp does not exist)

sys echo "input: $var1, $var2, $var_3, $var_4"		# sys is followed by an actual shell command.
```

The above BDS command is equivalent to the following shell commands. You can see that `$var1` is replaced with `test.txt`.
```
#!/bin/bash
echo "input: test.txt, test.txt.tmp, test2.txt, test2.txt_tmp"
```

`sys` is for a single-line shell command execution. Each `sys` command is executed <b>independently</b> by an <b>interative BASH (defined in `./bds.config`) shell</b> generated by BDS. In the following example, two interactive shells will be created by BDS, which means shell variables (not BDS ones) defined in each `sys` command cannot be shared.
```
sys echo "starting..."
sys echo "closing..."
```


### BDS syntax: `task`

To make `sys` commands serially executed in the same interactive shell, use `task`. In such task block, you can define any shell variables shared in the shell. Now we need to distinguish between BDS variables and shell variables. You can refer to a shell variable by `${VAR_NAME}`.
```
var1 	:= "test.txt"

task {
	sys echo "BDS var: $var1"
	
	sys var2="test2.txt"
	sys echo "Shell var: ${var2}"
}
```

It is important to understand a BDS operator `<-` which compares timestamp of files. In the following example, if any of files in `in` and `out` is empty or if any of files in `out` is older than any of files in `in`, it becomes true. if `task ( false ) {}`, the task is skipped. This is how a BDS pipeline can skip tasks which are already done in a previous run.
```
in 	:= [input1, input2]
out 	:= [output] // both a string or an array of string are allowed

task ( out<-in ) {

	sys echo "inputs : $input1, $input2"
	sys echo "output : $output"
	sys cat input1 input2 > output
}
```

You can also specify computer resources settings (# thread, max. memory and walltime) for a task. There are special (system) predefined BDS variables for them. <b>Do not try to redefine them in a global scope (or in a global function scope)</b>. It is recommended to make a function and define a `task` block in it. If you skip any of those special variables, default (predefined in a global scope) value will be used.
```
// Note that system variables are not defined with ':='.
// Defining them with ':=' or 'int cpus' will result in a redefinition error.

cpus 	= 1 	// set default # threads for all tasks as 1.
timeout = 3600 	// set default walltime for all tasks as 3600 seconds.

func()

void func {
	// Do not modify sys variables in a global scope. Define new ones in a function scope.
	taskName 	:= "task name here"
	cpus 		:= 3 		// integer
	mem 		:= 3000000000 	// in bytes
	timeout 	:= 7200 	// in seconds
	system		:= "local" 	// system for a task; "local" : task runs on a UNIX thread, "sge" : task is submitted to Sun Grid Engine, ...

	task {

	}
}
```


### BDS syntax for parallelization: `task`, `wait` and `par`

Parallelization of the pipeline can be implemented with `task`, `wait` and `par`. Consecutive tasks in a function are non-blockable. In the following example, two tasks will be queued and executed at the same time <b>without</b> `wait`, which is not an expected behavior. So explicltly add `wait` between tasks if you want them to be serialized.
```
void map( int replicate_id ) {

	task {
		echo "align ($replicate_id)"
	}

	wait // without wait, two tasks will go in parallel

	task {
		echo "post-align ($replicate_id)"
	}
	
	wait
}
```

`wait` is sort of a barrier for parallel tasks, you can actually get task id (tid) as a return value of a `task` block and then specify `wait` to wait for a specific task with tid. `wait` alone waits for all tasks defined in the scope it belongs to.
```
void map( int replicate_id ) {

	tid := task {
		echo "align ($replicate_id)"
	}

	wait tid

	task {
		echo "post-align ($replicate_id)"
	}

	wait // wait for all tasks in this function scope finish
}
```

Now we want to map two replicates and want them to go in parallel. Add a prefix `par` to the function. Note that `wait` in a global scope waits for <b>all tasks</b> in the pipeline. Therefore, the following code performs mapping of n replicates and wait for all mapping finish and then call peaks.
```
par map(1)
par map(2)
...

// parallel tasks for n replicates
for (int i=3; i<=n; i++) {
	par map(i)
	// tid := par map(i), you can also get a task id for a parallel function.
}

wait

callpeaks()
```

Note that `par` function can only return a task id even though it explicitly has a return type.
```
ret := par sum( 1, 2 ) // ret is not 3 it's a task id for par sum()

int sum( int i, int j ) {
	return i + j
}
```

This is why <b>parallelized</b> BDS pipelines must use global variables to store output file names.
```
int{} ret 	// global variable to store output of sum(), a map is recommended here

par sum( 1, 2 ) // ret is not 3 it's a task id for par sum()
par sum( 3, 4 )
par sum( 5, 6 )

void sum( int i, int j ) {

	key := "$i,$j" // make a key "i,j"
	ret{key} = i + j
}
```


### Hierarchy of BDS Pipelines

BDS is not object-oriented so that the hierarchy of modules is implemented by including a parent module in a child module. All variables and functions in a module are global so they can be referred and called in any of the successor modules, so it is important to explicitly leave comments to tell if variables and funciton are global or local. A module is loaded only once for a global scope so variable declaration and function calls in a global scope is called only once when the module is loaded first.

The hierarchy of basic modules (top to bottom: parent to child) are like the following:
```
base.bds 			// basic string/file/math functions
 └ conf.bds 			// varaibles and functions to read command line argument or configuration/environment files
   ├ general.bds 		// output directory, cluster resource (#threads, memory and walltime) and shell environment settings
   │ ├ species.bds 		// species definition, species specific parameters and functions to read species files
   │ ├ report_graphviz.bds 	// automatic graphviz diagram (SVG) generation
   │ └ report_filetable.bds 	// automatic HTML filetable generation
   ├ input_fastq.bds 		// parse command line argument or read configuration file to get fastqs (-fastq ...)
   ├ input_bam.bds 		// parse command line argument or read configuration file to get bams (-bam ...)
   ├ input_tag.bds  		// parse command line argument or read configuration file to get tagaligns (-tag...)
   ├ input_peak.bds 		// parse command line argument or read configuration file to get peaks (-peak ...)
   └ align_multimapping.bds 	// multimapping parameter

[report_graphviz.bds, report_filetable.bds]
 └ report.bds 			// HTML report generation, log/qc file parser, WashU browser track generation.

```

Other non-basic child modules typically include `species.bds` and `report.bds` since most modules need species specific parameters and need to generate graphviz diagram and filetable.
```
[species.bds, report.bds]
 ├ align_bwa.bds  		// bwa parameters, bwa resource settings, align fastqs to get raw bam
 ├ align_trim_adapters.bds  	// bowtie2 parameters, bowtie2 resource settings, align fastqs to get raw bam
 ├ postalign_bed.bds 		// postalign functions for bed and tagaligns (including cross-corr. analysis)
 ├ callpeak_spp.bds  		// peak calling with spp
 ├ callpeak_macs2.bds  		// peak calling with macs2 (separate macs2 function for chipseq and atac)
 ├ callpeak_etc.bds  		// naive overlap threshold, filtering peak files
 ├ idr.bds  			// IDR and its QC functions
 └ signal.bds  			// signal track generation using deepTools (bamCoverage) and Anshul's align2rawsignal.

// the following two modules share align_multimapping.bds for the parameter '-multimapping'

[species.bds, report.bds, align_multimapping.bds]
 ├ align_bowtie2.bds  		// bowtie2 parameters, bowtie2 resource settings, align fastqs to get raw bam
 └ postalign_bam.bds 		// postalign functions for bam (including filtering bam and bam_to_tagalign...)
```


### Module template
```
include "modules/species.bds"
include "modules/report.bds"

// variables globally used in all child modules (typically you want them to be command line parameters)
globalvar1 := ""	help Data file
glovalvar2 := 1 	// not a comand line parameter

// variables locally used in this module
localvar1 := 3


init_module_name() // initialization for a module (called only once)

chk_module_name() // check if parameters are consistent and correct, and all required data files exist (called only once)

// functions locally used in this module

void init_module_name() {

	// read from configuration file if it exists
	// use get_conf_val() for string get_conf_val_int() for integer, and so on

	globalvar1 	= get_conf_val( globalvar1,		["globalvar1"] ) 	// key name for the parameter is globalvar1
	glovalvar2 	= get_conf_val_int( glovalvar2, 	["glovalvar2"] )	// key name for the parameter is globalvar2
	...
}

void chk_module_name() {

	if ( !path_exists( globalvar1 ) ) error("globalvar1 ($globalvar1) does not exists!")
	...
}

// functions globally used in all child modules

void _func1() {
}

// more functions
...

```

Global functions typically have the following structure. `wait_par()` and `register_par()` is for optimized parallelization according to `-nth` (total # threads available for the whole pipeline). `wait_par()` waits if # threads for all running tasks reaches the limit (defined by `-nth`). `register_par()` registers a task id and # threads for the task to the monitoring thread of the pipeline and counts total # threads running.
```
mem_func1 	:= "3G" 	help Max. memory for func1
wt_func1 	:= "23h"	help Walltime for func1


// input, output_dir, label (info), # threads
string _func1( string input, string o_dir, string label, int nth_func1 ) {

	// get prefix path for input (remove extension of input and change its directory)
	prefix := replace_dir( rm_ext( input, ["ext1","ext",...] ), o_dir )

	// manipulate input file name for intermediate files used in a task block
	temp1 := "$prefix.tmp"
	...

	output := "$prefix.out"
	
	in 	:= [ input ] 				// make sure that 'in' is an array
	out 	:= output 				// for multiple outputs, use an array and change function return type to string[]
							// usage: (ret1, ret2, ...) = _funct( ... )

	taskName:= "taskname here"
	cpus 	:= nth_func1				// # threads for this task
	mem 	:= get_res_mem(mem_func1,nth_func1)	// get_res_mem() parses a string mem_func1 and divide it by nth_func1
	timewout:= get_res_wt(wt_func1) 		// get_res_wt() parses a string wt_func1

	wait_par( cpus )

	tid := task ( out<-in ) {

		sys $shcmd_init		# initialization shell command for all tasks
		//sys $shcmd_init_py3	# initialization shell command for all tasks using python3

		sys ...
	}

	register_par( tid, cpus )

	return out
}
```

You can omit resource related variables (such as `mem_func1` and `wt_func1`) and system varaibles definition (`cpus`, `mem` and `timeout`) for single-threaded tasks.
```
string _func2( string input, string o_dir, string label ) {

	// get prefix path for input (remove extension of input and change its directory)
	prefix := replace_dir( rm_ext( input, ["ext1","ext",...] ), o_dir )

	// manipulate input file name for intermediate files used in a task block
	temp1 := "$prefix.tmp"
	...

	output := "$prefix.out"
	
	in 	:= [ input ] 				// make sure that in is an array
	out 	:= output

	taskName:= "taskname here"

	wait_par( cpus ) // cpus are already defined as 1 in general.bds

	tid := task ( out<-in ) {

		sys $shcmd_init		# initialization shell command for all tasks
		//sys $shcmd_init_py3	# initialization shell command for all tasks using python3

		sys ...
	}

	register_par( tid, cpus )

	return out
}
```

If you need to use very short and light tasks, use the following template. These tasks will be not be counted by the monitoring thread which limits # threads running to `-nth`. But I recommend to use BDS built-in functions instead of a task block to make things simple and clear.
```
void func3() {

	system := "local" 	// this task will not be sumitted to a cluster engine (even though you specify it in the command line)

	task {
		sys ...		
	}
}
```


### Pipeline template

<b> Do not use `wait` in a global scope (or in a global function scope).</b> Use `wait_clear_tids()` instead. See more details in the following `Bugs in BDS` sections.
```
include "modules/any_module_you_want.bds"
...

main()

// global scope

void main() { 

	// global function scope
	
	init()
	chk_input()
	align()
	call_peaks()
	idr()
	report()	
}

void init() {...}

void chk_input() {...}

void align() {

	...
	wait_clear_tids() // wait for all aligning tasks are done
}

void call_peaks() {

	...
	wait_clear_tids() // wait for all peak-calling tasks are done
}
```


### Bugs in BDS

1) thread safety issue for global variables

It is already explained that we need global variables (typically map of string to store output file names; map's key is typically replicate id here) for parallelized BDS pipelines. There is a bug in handling global variables (locking/unlocking them).Reading (as r-value) and writing (as l-value) on a global variable in a `par` function will result in a crash of a pipeline. A hacky way to prevent this problem is <b>not to read global variables</b> in a `par` function. It's safe to read gloval variables in a `par` function only when all parallel tasks are finished (`wait` or `wait_clear_tids()` in a global scope). Also, add `monitor_par()` at the end of all `par` function. This function is sort of a barrier marking jobs as done when they finish.

```
string{} bam, filt_bam, tagalign, peak	// global variables to store pipeline outputs (filenames)

par align_OKAY( 1 )
par align_OKAY( 2 )

wait_clear_tids() 			// 'wait' alone is okay too, but not recommended.

callpeak_OKAY()

void align_ERROR( int replicate_id ) {

	key := replicate_id

	fastq := get_fastq( key )

	bam{key} = _bwa( fastq ) // it's okay to write on bam{}

	wait

	filt_bam{key} = _dedup_bam( bam{key} ) // it's NOT OKAY to read bam{key}

	wait 

	tagalign{key} = _bam_to_tag( filt_bam{key} ) // it's NOT OKAY to read filt_bam{key}
}

void align_OKAY( int replicate_id ) {

	key := replicate_id

	fastq := get_fastq( key )

	bam_ := _bwa( fastq )
	bam{key} = bam_

	wait

	filt_bam_ := _dedup_bam( bam_ ) // it's SAFE since bam_ is a local variable
	filt_bam{key} = filt_bam_

	wait 

	tagalign_ := _bam_to_tag( filt_bam ) // it's SAFE
	tagalign{key} = tagalign_

	monitor_par() // IMPORTANT
}

void callpeak_OKAY() {
	
	// we can safely read tagalign{} since all 'par' tasks finished due to `wait_clear_tids()` or `wait` in the global scope.

	tag1 := tagalign{1}
	
	peak_ := macs2( tag1 )		// again make a temporary local variable for peak{}
	peak{1} = peak_

	...
}
```

2) `tid.isDone()` not working

<b> Do not use `wait` in a global scope.</b> Use `wait_clear_tids()` instead.`wait` itself works fine but the pipeline uses its own monitoring thread to count # thread running (and limit it by `-nth`). This monitoring thread is based on the global array `string[] tids_all` and iterate over task ids with `tid.isDone()` to check if each task is done. `tid.isDone()` does not work in a global scope (it only works in a `par` function scope). Therefore, it is necessary to clear `tids_all` manually when all `par` functions finish. This is due to a BDS bug that does not mark finished jobs as done in a member function `tid.isDone()`. This issues has been reported to the BDS github repo. <b>You can still use `wait` in a `par` function scope.</b>

This bug is fixed in the latest BDS (06/06/2016).


3) `goal` and `dep`

`goal` and `dep` seem to be buggy (pipeline crashes) when combined with `par` prefix. Stay away from those syntax.
