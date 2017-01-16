---
layout: post
title:  "Writing Batch Jobs For Iceberg"
date:   2017-01-16 12:00:00 +0100
categories: iceberg,rse
---

Part of my research involves running large-scale batch jobs using the university's HPC cluster. Iceberg uses Sun Grid Engine to manage jobs which provides a fairly primitive means of executing work. It provides a very thin wrapper around shell scripts that run on workers allocated by SGE.

The challenge of doing this is the lack of abstraction and control. Once a task is handed off to the queue there isn't much you can do to monitor it or check up on it. While SGE does have a few nice features that give a bit more control, e.g. via the DRMAA interface, ultimately, you still only have a low-grained means of interacting with the jobs.

This post briefly discusses some of the pain points and gotchas I've experienced and a bit of advice for how to avoid them. A big chunk of the problem stems from trying to separate the concerns by making jobs agnostic to the machine, and the annoying coupling to SGE primitives and shell scripts for job control.

You can't have a message queue on Iceberg, nor can you run something like Hadoop. If you want to run large-scale batches on Iceberg with little or no supporting infrastructure you're going to have to talk SGE. What that doesn't mean is that your workload is forever tied to Iceberg if you're careful.

# Expect Debugging

Part of the problem with using any queue system is the lack of observability you get. With Iceberg there's no guarantee you can get onto the node that a job failed on to dig around, so it's really useful to generate as much information as you can. Setting the `-x` flag in your scripts logs every command to give some trace of what happened.

Once your jobs are executing, be prepared to spend a lot of time debugging them. Using logging beforehand simplifies the challenge of rerunning jobs to see where they failed. Once your jobs do start running, be aware that it's a huge pain to track when they fail.

## Why Jobs Fail

There are a lot of reasons your jobs might not fail. Some of them will be down to mistakes, others will be down to things you can't control. Most of this advice is for building something that runs correct code in a way that doesn't make everything unpleasant. It won't stop any of these things happening, but it makes re-running jobs a lot smoother.

* Your environment isn't setup properly (attempt to access a file that doesn't exist)
* Paths aren't configured properly (wrong version of binaries)
* Wrong version of application code or inputs
* Transient machine faults
* Bad allocation (memory, wall time)
* Faults in application code

# Decomposition and Configuration

Job scripts always start simple and gradually organically grow. This makes it annoyingly easy for paths to get hardcoded. It helps to break up your scripts into those that set up a worker, such as creating temporary files, copying data to `/scratch`, the script that runs in an environment prepared for it, and something to clean up afterwards and store results. For example, instead of one `runit.sh` that does everything, decompose it:

	#setup.sh
	set -e
	source $CONFIG_FILE
	mkdir -p /fastdata/$USER/inputs
	mkdir -p /fastdata/$USER/output

	#job.sh
	set -e
	source $CONFIG_FILE
	$SETUP_FILE
	mkdir -p $SCRATCH_DIR
	cd $SCRATCH_DIR
	$JAVA $BIN_DIR/software $INPUT $OUTPUT
	$CLEANUP_FILE

	#cleanup.sh
	rm -r $SCRATCH_DIR

	#config.sh
	SCRATCH_DIR=/scratch/$USER
	SETUP_FILE=/home/$USER/iceberg-setup.sh
	CLEANUP_FILE=/home/$USER/iceberg-cleanup.sh

The job script itself (and the software it invokes) should be completely configurable. Putting all the variables in a separate file means you can leave a file on the cluster and you only have to point to it to set the correct directories.

A big problem here is trying to decide what is a generic setup task, and what is so tied to the script that it has to happen at the same time and on the same node. In my case, I have a directory that I just push onto Iceberg that contains everything and scripts to move inputs into place. This happens independently of the job invocations themselves.

# Parameterise Job Script

The job script should take parameters from two places only. Infrequently changing parameters should be set in the configuration. Other parameters should be set as additional environment variables. This makes things nice, because SGE is doing nothing magic for your job script; it doesn't need to do anything.

There is one exception; if you use array jobs, then you need to map `SGE_TASK_ID` to another variable. Your config file could do that:

	#iceberg_config.sh
	JOB_NUMBER=$SGE_TASK_ID
	...

Now, to execute a job it's only necessary to specify one additional parameter, the path to the config values.

    CONFIG_FILE=my_config.sh INPUT=foo OUTPUT=bar ./job.sh

# Fail Fast and Often

When dealing with Iceberg it's not unusual for the filesystem to break. It's worth setting the `-e` flag on your scripts so they terminate as soon as a called process generates a bad return value. This is a good habit to get into.

It's also highly useful if your job script indicates when it fails by setting the right exit code. If everything goes well that's fine, but it's useful for doing the inevitable post-mortem on your jobs to figure out which subset fail and why.

Don't forget to clean up after failures too. You can run your actual job in another script that's wrapped by a script that makes sure cleanup happens no matter what the actual job did.

# Idempotence and Atomicity

Your jobs should tolerate repeated execution. You will inevitably rerun a job so it makes sense to absolve yourself of doubt of your results when that does happen. The easiest way to get this property is to ensure that jobs only operate on copies of data. After processing, a separate part of the script should move the relevant files back to the output directory, overwriting files if necessary.

If you set the `-e` flag in your job script, this guarantees that the only time a result gets moved to the main output directory is when the job didn't fail. If you're diligent when it comes to your application code (or putting checks in the script to set the exit status) then you have a reasonable guarantee that your output directory reflects the results your software generates. This makes your job atomic; it either finishes or it doesn't.

If your code generates multiple files, it's a good idea to move them to a temporary location on the same filesystem then atomically rename that to the desired output. If you can keep this final rename process to a single rename command you know your outputs are always consistent if they exist. This might mean bundling everything under a single output directory for each invocation.

# Post Mortems

The biggest pain when using Iceberg is figuring out which jobs you need to repeat. There are many ways to do this. I tend to default to using the state of the output directory to figure out what files are missing. This requires some more code that can figure out which files would be generated by a given script invocation and which files are missing. You can make this process easier by organising the output directory such that a subdirectory's presence indicates the job executed.

It's also not a terrible idea to record when a job was started by touching a file somewhere, and touching another file when it exits. This way you have a platform-independent record of the jobs that failed or have yet to start.

Using the filesystem is convenient and works independently of the system your code is running on. The disadvantage is the distance at which the reporting is from execution. The code to generate the job script invocation commands from the missing outputs is fairly trivial, but it means there's more code to maintain. Unless your script invocation generator is sophisticated, you'll probably end up with two shell scripts, one to generate the initial job set, and another to generate the missing ones.

SGE also provides a means to interrogate the job result file. `qacct` will tell you the exit status of your jobs, so if your jobs do fail, you can simply query for failed jobs and try to re-run them. The disadvantage with doing this is that you become dependent on SGE and need to write some logic to figure out if a failing job's successors failed. This also means that after submitting your jobs for the first time you become wholly dependent on SGE to tell you what to re-run. Fine if you only use Iceberg, but a problem if you intend to distribute work over multiple machines/clusters.

# Good Citizenship

The process of getting stuff running on Iceberg seems to involve a lot of computational waste. It's hard to mock up the environment, so you'll end up testing in production. It's generally a decent thing to try to avoid doing this too often. You can reduce the impact by making sure your jobs at least run on your own machine first. Furthermore, don't go all-in on the first run, send a smaller batch first.

It makes a lot of sense to avoid nugatory work where you can. If you want to be lazy and resubmit a big block of jobs knowing only one needs to re-run this is expensive and penalises you in the queue. If you really must do this, you should give your jobs a means to check if the output they'll eventually produce already exists.

# Make is not a bad tool

If your jobs themselves have some fairly close dependencies, then you might consider using a Makefile to describe these instead of writing a lot of logic in a shell script. This has a few benefits: jobs will automatically fail-fast when the exit code is wrong and you have to be explicit when you ignore an error code. Moreover, your jobs will skip any unnecessary work if the outputs to be generated don't trigger any rebuilds.

An example of where Make is useful is setting up temporary directories:

	.PHONY := run
	run: $(OUTPUT_DIR)/result.csv $(OUTPUT_DIR)/log.txt
	
	CODE := $(BINDIR)/tool.jar
	
	
	$(OUTPUT_DIR):
		mkdir $@
	
	$(TEMP_DIR):
		mkdir $@
	
	$(OUTPUT_DIR)/result.csv $(OUTPUT_DIR)/log.txt: $(OUTPUT_DIR) $(TEMP_DIR)/result.csv $(TEMP_DIR)/log.txt
		mv $(TEMP_DIR)/result.csv $(OUTPUT_DIR)
		mv $(TEMP_DIR)/log.txt $(OUTPUT_DIR)
	
	$(TEMP_DIR)/result.csv $(TEMP_DIR)/log.txt: $(TEMP_DIR) $(CODE) $(INPUT)
		java -jar $(CODE) $(INPUT) $(TEMP_DIR)

The above example ensures that:

* Execution of the tool happens separately from installing the results
* The tool is only executed when the input or code changes

This extra checking means the job script can be made much simpler. It only needs to set the variables in the Makefile and invoke the target. You may want to set these variables in another Makefile that's loaded from an environment variable to save a lot of work.

# Workflow

The above set of primitives are useful to have in building jobs tolerant of a system like Iceberg. I have only briefly touched on the point of how job scheduling should work. The challenge of figuring out what's finished gets worse when you have larger scale jobs that won't all fit on the queue.

I've found that it's better to not have a live queue and instead use something to manage submitting jobs to Iceberg then spotting the missing results. This is an alternative to using a real queue that remembers which jobs have been submitted and checks on their status. It shouldn't be hard to build such a tool, but when the only communication method you have is an SSH connection it makes it more hassle than it's perhaps worth.

The workflow I recommend is therefore:

1. Write a script to build and push your code to Iceberg
2. Write a script to push inputs to Iceberg
3. Write the setup, job, and cleanup scripts as above (preferably using Makefiles)
4. Run a test batch
5. Write code to generate all the jobs that need executing (emitting `qsub` commands)
6. Execute those jobs by piping them to `ssh iceberg`
7. Write code to figure out which jobs need executing, emitting them as `qsub` commands (the `diff` command can be used to compare the list of remaining jobs with the total set of jobs to execute)

This works fairly well for isolated tasks that involve running a single command on a large batch. It doesn't work well when tasks have dependencies on one another.

# Handling Dependencies Between Tasks

This is an open problem for me. My preference is to use each batch as a synchronisation point. Each distinct phase is written as a separate set of scripts. This involves manual work on the cluster to check that one phase is complete before submitting the next phase's tasks. This isn't great, and it would be nicer to more expressively encode dependencies between tasks ala Hadoop, but while SGE does support dependencies between tasks my preference is to ignore as much of SGE as possible.

# Why ignore SGE?

I want to close with some justification for ignoring the scheduler. The advice above pertains to people who don't care about HPC. For me, I would sooner have a million machines for 30 seconds than 30 machines for a million seconds. I don't depend on MPI or specific CPU architectures. At the very least making jobs highly agnostic of their execution environment means switching to something else is fairly easy.

## Example: Azure Batch Service

I recently migrated some Iceberg code to Azure's Batch job service and found it fairly painless. While with Iceberg it's possible to `rsync` code to the server, the batch service doesn't give you a machine. Instead, you must provide a bundle of your code and data and a script that installs it. If your jobs are configurable (as they would be in this example) it's simply a case of building the script that bundles the application package and sets up a target machine, unpacking data as necessary.

In my case, I actually built the package by pointing my script to provision a worker at my own machine, then zipped up the directory it prepared. The script to install Java and unpack that zip was simple to write too.

Unlike SGE, Azure has a richer API for job status. While using shell scripts means fine-grained control and error recovery is difficult, the exit status is still easily queried using the batch service. With the batch service, a job is just a shell command, which is analogous to the command issued to `qsub`.

As an exercise, it's really useful to try executing workloads on another machine. If nothing else, it gives you confidence you can survive a prolonged outage, such as one that might occur if power fails.
