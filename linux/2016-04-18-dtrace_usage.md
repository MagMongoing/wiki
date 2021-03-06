* Not just user-level software, including applications, databases and webservers, but also the operating system kernel and device drivers. The name is short for Dynamic Tracing: an instrumentation technique pioneered by DTrace which dynamically patches live running instructions with instrumentation code. The DTrace facility also supports Static Tracing: where user-friendly trace points are added to code and compiled-in before deployment.

* DTrace provides a language, 'D', for writing DTrace scripts and one-liners. The language is like C and awk, and provides powerful ways to filter and summarize data in-kernel before passing to user-land. This is an important feature that enables DTrace to be used in performance-sensitive production environments, as it can greatly reduce the overhead of gathering and presenting data.

	```
	dtrace -l -n tcp::entry or dtrace -l -m tcp
	dtrace -lv -n fbt:tcp:inet_info:entry
	```

* A D script consists of a probe description, a predicate, and actions as shown
    below:
    
    ```
    probe description
    /predicate/
    {
    	actions
    }
    ```

* When the D script is executed the probes described in the probe description are enabled.

* DTrace inserts the interrupt dynamically.

* DTrace uses the following format to identify a probe:
    
	```
    provider:module:function:name
	```

    Or, from the command line:
    
    ```
    -P     provider
    -m     module
    -f     function
    -n     name 
    -s     script
    -p     pid
    -c     run command
    -l     list
    -v     verbose
    ```

    So a simple DTrace script would look like this:
    
    ```
    syscall:::entry                     // probe description
    /pid == $1/                        // predicate
    {
    	@[probefunc] = count();    // action
    }
    ```

    In the preceding script, syscall is the provider. The module designation is
    omitted so the script matches on any module. The function designation is
    likewise omitted, so the script matches on any function. In this example, we are interested only in the entry to the function as opposed to the return. The
    predicate `( /pid == $target/)` determines if the action `( @[probefunc] = count();)` is performed. The action occurs only when the pid (process id) matches the first parameter passed on the command line.

    The action is an aggregation (designated by the @) that contains the count by function name of the system calls executed. By default, the contents of the
    aggregation are printed upon exit from the script.

* Use `dtrace -l -P pid<pid_of_backend> -m postgres` to list all the probes of a backend process;

* $target is used with -p <PID> in provider, while in predicate, we can directly
    use keyword pid, same as cpu, execname, ppid, arg0;

* For action part, serverl common used actions are: `printf()`, `ustack()`, `trace`; if action is missed, then the name of the probe fired is printed;

* The types of information application developers are particularly interested in
    can be obtained by using the following providers: syscall, proc, pid, sdt,
    vminfo.

* You may have noticed that the output from scripts went by too fast to be useful. D language has a wonderful construct called aggregate to collect all the detail in memory and print out a summary. Aggregations allow you to collect tables of information in memory. Aggregations have the following construct:

	```
    @name[table index(es)] =aggregate_function()
    ```

    For example:
    
    ```
    @count_table[probefunc] = count() ;
    ```
    This aggregation will collect information into a table with name of the function (probefunc is a built-in variable that has the name of the function). The aggregation function count() keeps track of the number of times the function was called. The other popular aggregation functions are average, min, max, and sum.

    Once you stop the script, DTrace will print out the table of information. Notice that
    you do not need to write any code to print the table. The probes are disabled as soon
    as the script is stopped. You do not need to do anything special for disabling these
    probes. In addition, DTrace will automatically clean up any memory it allocated.
    You do need to write any code for cleanup.

* DTrace allows you to monitor a process from the time it starts until it ends using the \$target variable and the -c option. This script will count the number of times libc functions are called from a given application.

	```
    -- libc_func.d
    #!/usr/sbin/dtrace -s
    pid$target:libc::entry
    {
    	@[probefunc]=count();
    }
    ```
    
    then run the target binary using:
    
    ```
    libc_func.d -c "postgres > /tmp/log 2>&1 &"
    ```

* proc provider fires at processes and thread creation and termination as well as signals.

* pid provider is unstable, and is the provider you're most likely to
    shoot yourself in the foot with. -- Brendan Gregg

	Since the pid provider is exposing functions from software, which may be
	changed by the programmer at any time (software updates, patches, new
    releases), this is an "unstable" interface. There is no guarantee that
	these probes will work on any software version other than the one they
	were written for.

	POSIX functions can provide reliable pid provider probes.
	Public API functions make for decent pid probes too.

	If it is a static function and is only called locally in a file, then
	the likelihood of it changing is high. Really try to avoid. (The compiler
    may not even keep the symbol, such as by inlining the function, which would make
	the function entry invisible to DTrace anyway.)

	Thus, try to avoid tracing random local functions.

	The potential brittleness of the pid provider is not a flaw of DTrace, but a
    reflection of the underlying code. DTrace has a feature to address the
    situation: User Statically Defined Tracing (USDT).

    This allows developers to define stable probe points in their code. By doing so,
	you can create new providers which are explicitly defined to be stable, and
	export stable probe names. Only reach for the pid provider if a USDT
	provider doesn’t exist, or can't be introduced.

* My test result shows that, if the compiler reserve the symbol information of
    a binary, then dtrace can find the probes of each function symbol.

* **IMPORTANT TIPS** -- Not a big problem for internal test usage

	While you can try to pick safer probes, there is always the possibility that
	something will break across software versions. How this happens to function
	names includes the following:
	* The function is renamed.
	* The function is deleted, and the code moved elsewhere.
	* A new function is added to perform this task, and this function is left behind
	  in the source as legacy (or as a slow path).
	* The function is rewritten to perform a different action.

	If it's (1) or (2), you are lucky, dtrace would report "failed to compile script ./mysql\_pid\_latency.d: line 18: probe description. pid23423::\*parse_command*:entry
	does not match any probes".

	Then you can immediately start work fixing the script: which can include
	examining the soure code (if available) for the software version the script
	is written for and the new version, to see what became of that function.

	If it's (3) or (4), your script will execute, but the results will not be
	what you expect. You could describe this as a silent failure. If you are
	lucky here, you'll have a good estimate of what the results should be, and
	what you get is so far off that you'll begin to investigate why. If you are
	unlucky, the output will seem plausible (but is wrong) and you'll continue
	your investigation elsewhere based on bad data.

	On a different software version, the same pid probes may appear to work, but
	the results are wrong.

	It's a good habit of documenting the software version in pid provider DTrace
	scripts, with a block comment at the top, to help the future user debug a broken script

* DTrace probes execute in the kernel address space. Probes use the copyin() or
	copyinstr() subroutines to copy user process data into the kernel's address
	space. This is critical for ptr args.

	Consider the following write() system call:
	
	```
		ssize_t write(int fd, const void *buf, size_t nbytes);
	```

	The following D program illustrates an incorrect attempt to print the contents of a string that is passed to the write system call:

	```
	syscall::write:entry
	{
		    printf("%s", stringof(arg1)); /* incorrect use of arg1 */
	}
	```

	When you run this script, DTrace produces error messages similar to the
	following example.

	```
	dtrace: error on enabled probe ID 1 (ID 37: syscall::write:entry): \
		    invalid address (0x10038a000) in action #1
	```

	The arg1 variable is an address that refers to memory in the process
	that is executing the system call. Use the copyinstr() subroutine
	to read the string at that address. Record the result with the
	printf() action:

	```
	syscall::write:entry
	{
	    printf("%s", copyinstr(arg1)); /* correct use of arg1 */
	}
	```
	The output of this script shows all of the strings that are passed to
	the write system call.

	Another dilemma here is: copyin() and copyinstr() subroutines cannot read
	from user addresses which have not yet been touched. A valid address might
	cause an error if the page that contains that address has not been faulted
	in by an access attempt. Consider the following example:
	
	```
	dtrace -n syscall::open:entry'{ trace(copyinstr(arg0)); }'
	```

	```
	dtrace: description 'syscall::open:entry' matched 1 probe
	CPU     ID                    FUNCTION:NAME
	dtrace: error on enabled probe ID 2 (ID 50: syscall::open:entry): invalid address
	(0x9af1b) in action #1 at DIF offset 52
	```

	In the output from the previous example, the application was functioning properly
	and the address in arg0 was valid. However, the address in arg0 referred to a page
	that the corresponding process had not accessed. To resolve this issue, wait for the
	kernel or application to use the data before tracing the data. For example, you might
	wait until the system call returns to apply copyinstr(), as shown in the following example:
	```
		dtrace -n syscall::open:entry'{ self->file = arg0; }' \ 
		-n syscall::open:return'{ trace(copyinstr(self->file)); self->file = 0; }'
	```

* The ustack() action records program counter (PC) values for the stack. The
	dtrace command resolves those PC values to symbol names by looking though the
	process's symbol tables. The dtrace command prints out PC values that cannot be
	resolved as hexadecimal integers.
