# bashJobQueue

A simple job queue system written in Bash.  Jobs can be submitted to
a queue.  A specific number of jobs will run concurrently.  When jobs
run the output is logged.  The status of the job will be reported, or
the output of a specific job can be reported.

This simple scheduler is written to allow for several large jobs to be
submitted at once and the jobs will slowly run once other jobs have finished.
When I think of something to run with the arguments, the job is submitted.
Jobs can be queued even if the scheduler is not running.  The system is simple
enough that it can quickly be deployed and adapted for different purposes.

## The scheduler

The scheduler will run up to `n` jobs concurrently.  The scheduler does not
take the size of the job into consideration, it only limits the number of concurrent
jobs.

To launch the scheduler, use the `start_scheduler` script.  This command has the following
arguments:

`-n, --processes [n]`, the number of active processes the scheduler should run.
`--queue-pending`, when starting the scheduler, queue all jobs that are in the `pending` state
`--queue-running`, when starting the scheduler, queue all jobs in the `running` state.  These jobs might have started but due to the system shutting down they are no longer running.  This will requeue the jobs so they will restart.

Once the scheduler starts it will monitor a named pipe, which acts as the job queue.

### Example

Start the scheduler, running up to 4 jobs simultaneously.  Requeue all jobs that
have not finished.

    start_scheduler -n 4 --queue-pending --queue-running
## submitting jobs

Jobs are submitted using the `submit` command.  The submit command saves
the current environment and creates a wrapper script to run the command
submitted.  Since the environment is saved, the command to be executed can use
environment variables for settings.  The current working directory is also saved
and will be restored when the job starts.

The script will redirect the output of the job to a file.

Once the wrapper script is created, it submits the name of the wrapper script to
the scheduler via a named pipe to queue.

### Example

Submit a command to calculate the first 10000 digits of pi.

    submit python calculate_pi.py -n 10000

## Job status

Status of the job queue and specific jobs is read using `status`.

When `status` is run without arguments, report the status of all jobs.
This will print a table of all job submitted, the job index for each job, and
the command submitted.

To get the log for a specific job, run `status <job index>`.  This will print
the output log for that specific job.  Logs are only available for jobs that are
either in the running or finished state.

### Examples

Get a list of all jobs

    status

Get the log for job with index 4

    status 4

## Internals

### The job queue

The job queue is a named pipe located in the same folder as the output.  The length
of the queue before blocking occurs is dependent on the system where this is running.
However, as items are queued using `submit` they are pulled out of the pipe using `tail -f`.  Tail buffers, so writing to the queue will not block.

`xargs` is used to run jobs.  This command was used because it provides a standard
solution that is often available on most systems.  Since jobs submitted to the queue
is the name of the script, `xargs` can run the command as is.

### Job files

When a job is submitted, the command to be run is added to a file called `job_list`.  The
line where the command is found in this file is the job index.  The current environment
is packaged, along with the command to run, and written to a file `<job index>_script.sh`.
This is the wrapper script to run by the scheduler.

When the job starts the file `<job index>_out` is written to the same directory as the
script file.  This contains the `stdout` and `stderr` outputs from the script.  Both
are dumped to the same file.  Having both in the same place is useful in case context
is needed to understand what happened before stderr is written to.  If separate output
is needed for `stdout` and `stderr`, edit the submit script to change how redirection
is done.

When the job finishes the file `<job index>_finished` is written.  The contents of this
file is the return value from the command.  The contents of this file can be used to
know if the job finished successfully.

When reporting the status of jobs, three states exist:

state   | Meaning
--------|----------
Pending | A script file exists, but the output or `_finished` files do not exist.
Running | A script file exists and an output file exists, but no `_finished` file exists.  If the job was running and was terminated for some reason (such as system crash), a job might be Running when it is not really running.
Finished| A script file, output file, and `_finished` file exists for the job.