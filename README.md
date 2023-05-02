Download Link: https://assignmentchef.com/product/solved-cos418-assignment-1-part-2-and-3-sequential-map-reduce
<br>
<h2><a id="user-content-introduction" class="anchor" href="https://github.com/theoliao1998/Distributed-Systems/blob/master/1-2%2C3%20Sequential%20%26%20Distributed%20MapReduce/README1.md#introduction" aria-hidden="true"></a>Introduction</h2>

In parts 2 and 3 of the first assignment, you will build a Map/Reduce library as a way to learn the Go programming language and as a way to learn about fault tolerance in distributed systems. For part 2, you will work with a sequential Map/Reduce implementation and write a sample program that uses it.

The interface to the library is similar to the one described in the original <a href="http://research.google.com/archive/mapreduce-osdi04.pdf" rel="nofollow">MapReduce paper</a>.

<h2><a id="user-content-software" class="anchor" href="https://github.com/theoliao1998/Distributed-Systems/blob/master/1-2%2C3%20Sequential%20%26%20Distributed%20MapReduce/README1.md#software" aria-hidden="true"></a>Software</h2>

You’ll implement this assignment (and all the assignments) in <a href="https://www.golang.org/" rel="nofollow">Go</a>. The Go web site contains lots of tutorial information which you may want to look at.

For the next two parts of this assignment, we will provide you with a significant amount of scaffolding code to get started. The relevant code is under this directory. We will ensure that all the code we supply works on the CS servers (cycles.cs.princeton.edu). We expect that it is likely to work on your own development environment that supports Go.

In this assignment, we supply you with parts of a flexible MapReduce implementation. It has support for two modes of operation, <em>sequential</em> and <em>distributed</em>. Part 2 deals with the former. The map and reduce tasks are all executed in serial: the first map task is executed to completion, then the second, then the third, etc. When all the map tasks have finished, the first reduce task is run, then the second, etc. This mode, while not very fast, can be very useful for debugging, since it removes much of the noise seen in a parallel execution. The sequential mode also simplifies or eliminates various corner cases of a distributed system.

<h3><a id="user-content-getting-familiar-with-the-source" class="anchor" href="https://github.com/theoliao1998/Distributed-Systems/blob/master/1-2%2C3%20Sequential%20%26%20Distributed%20MapReduce/README1.md#getting-familiar-with-the-source" aria-hidden="true"></a>Getting familiar with the source</h3>

The mapreduce package (located at <tt>$GOPATH/src/mapreduce</tt>) provides a simple Map/Reduce library with a sequential implementation. Applications would normally call <tt>Distributed()</tt> — located in <tt>mapreduce/master.go</tt> — to start a job, but may instead call <tt>Sequential()</tt> — also in <tt>mapreduce/master.go</tt> — to get a sequential execution, which will be our approach in this assignment.

The flow of the mapreduce implementation is as follows:

<ol>

 <li>The application provides a number of input files, a map function, a reduce function, and the number of reduce tasks (<tt>nReduce</tt>).</li>

 <li>A master is created with this knowledge. It spins up an RPC server (see <tt>mapreduce/master_rpc.go</tt>), and waits for workers to register (using the RPC call <tt>Register()</tt> defined in <tt>mapreduce/master.go</tt>). As tasks become available, <tt>schedule()</tt> — located in <tt>mapreduce/schedule.go</tt> — decides how to assign those tasks to workers, and how to handle worker failures.</li>

 <li>The master considers each input file one map task, and makes a call to <tt>doMap()</tt> in <tt>mapreduce/common_map.go</tt> at least once for each task. It does so either directly (when using <tt>Sequential()</tt>) or by issuing the <tt>DoTask</tt> RPC — located in <tt>mapreduce/worker.go</tt> — on a worker. Each call to <tt>doMap()</tt> reads the appropriate file, calls the map function on that file’s contents, and produces <tt>nReduce</tt> files for each map file. Thus, after all map tasks are done, the total number of files will be the product of the number of files given to map (<tt>nIn</tt>) and <tt>nReduce</tt>.<pre>f0-0, ...,  f0-[nReduce-1],...f[nIn-1]-0, ..., f[nIn-1]-[nReduce-1].</pre></li>

 <li>The master next makes a call to <tt>doReduce()</tt> in <tt>mapreduce/common_reduce.go</tt> at least once for each reduce task. As with <tt>doMap()</tt>, it does so either directly or through a worker. <tt>doReduce()</tt> collects corresponding files from each map result (e.g. <tt>f0-i, f1-i, ... f[nIn-1]-i</tt>), and runs the reduce function on each collection. This process produces <tt>nReduce</tt> result files.</li>

 <li>The master calls <tt>mr.merge()</tt> in <tt>mapreduce/master_splitmerge.go</tt>, which merges all the <tt>nReduce</tt> files produced by the previous step into a single output.</li>

 <li>The master sends a Shutdown RPC to each of its workers, and then shuts down its own RPC server.</li>

</ol>

You should look through the files in the MapReduce implementation, as reading them might be useful to understand how the other methods fit into the overall architecture of the system hierarchy. However, for this assignment, you will write/modify <strong>only</strong> <tt>doMap</tt> in <tt>mapreduce/common_map.go</tt>, <tt>doReduce</tt> in <tt>mapreduce/common_reduce.go</tt>, and <tt>mapF</tt> and <tt>reduceF</tt> in <tt>main/wc.go</tt>. You will not be able to submit other files or modules.

<h2><a id="user-content-part-i-mapreduce-input-and-output" class="anchor" href="https://github.com/theoliao1998/Distributed-Systems/blob/master/1-2%2C3%20Sequential%20%26%20Distributed%20MapReduce/README1.md#part-i-mapreduce-input-and-output" aria-hidden="true"></a>Part I: Map/Reduce input and output</h2>

The Map/Reduce implementation you are given is missing some pieces. Before you can write your first Map/Reduce function pair, you will need to fix the sequential implementation. In particular, the code we give you is missing two crucial pieces: the function that divides up the output of a map task, and the function that gathers all the inputs for a reduce task. These tasks are carried out by the <tt>doMap()</tt> function in <tt>mapreduce/common_map.go</tt>, and the <tt>doReduce()</tt> function in <tt>mapreduce/common_reduce.go</tt> respectively. The comments in those files should point you in the right direction.

To help you determine if you have correctly implemented <tt>doMap()</tt> and <tt>doReduce()</tt>, we have provided you with a Go test suite that checks the correctness of your implementation. These tests are implemented in the file <tt>test_test.go</tt>. To run the tests for the sequential implementation that you have now fixed, follow this (or a non-<tt>bash</tt> equivalent) sequence of shell commands, starting from the <tt>418/assignment1-2</tt> directory:

<pre># Go needs $GOPATH to be set to the directory containing "src"$ cd 418/assignment1-2$ lsREADME.md src$ export GOPATH="$PWD"$ cd src$ go test -run Sequential mapreduce/...ok  mapreduce 4.515s</pre>

If the output did not show <em>ok</em> next to the tests, your implementation has a bug in it. To give more verbose output, set <tt>debugEnabled = true</tt> in <tt>mapreduce/common.go</tt>, and add <tt>-v</tt> to the test command above. You will get much more output along the lines of:

<pre>$ go test -v -run Sequential=== RUN   TestSequentialSinglemaster: Starting Map/Reduce task testMerge: read mrtmp.test-res-0master: Map/Reduce task completed--- PASS: TestSequentialSingle (2.30s)=== RUN   TestSequentialManymaster: Starting Map/Reduce task testMerge: read mrtmp.test-res-0Merge: read mrtmp.test-res-1Merge: read mrtmp.test-res-2master: Map/Reduce task completed--- PASS: TestSequentialMany (2.32s)PASSok  mapreduce4.635s</pre>

<h2><a id="user-content-part-ii-single-worker-word-count" class="anchor" href="https://github.com/theoliao1998/Distributed-Systems/blob/master/1-2%2C3%20Sequential%20%26%20Distributed%20MapReduce/README1.md#part-ii-single-worker-word-count" aria-hidden="true"></a>Part II: Single-worker word count</h2>

Now that the map and reduce tasks are connected, we can start implementing some interesting Map/Reduce operations. For this assignment, we will be implementing word count — a simple and classic Map/Reduce example. Specifically, your task is to modify <tt>mapF</tt> and <tt>reduceF</tt> within <tt>main/wc.go</tt> so that the application reports the number of occurrences of each word. A word is any contiguous sequence of letters, as determined by <a href="https://golang.org/pkg/unicode/#IsLetter" rel="nofollow"><tt>unicode.IsLetter</tt></a>.

There are some input files with pathnames of the form <tt>pg-*.txt</tt> in the <tt>main</tt> directory, downloaded from <a href="https://www.gutenberg.org/ebooks/search/%3Fsort_order%3Ddownloads" rel="nofollow">Project Gutenberg</a>. This is the result when you initially try to compile the code we provide you and run it:

<pre>$ cd "$GOPATH/src/main"$ go run wc.go master sequential pg-*.txt# command-line-arguments./wc.go:14: missing return at end of function./wc.go:21: missing return at end of function</pre>

The compilation fails because we haven’t written a complete map function (<tt>mapF()</tt>) nor a complete reduce function (<tt>reduceF()</tt>) in <tt>wc.go</tt> yet. Before you start coding read Section 2 of the <a href="http://research.google.com/archive/mapreduce-osdi04.pdf" rel="nofollow">MapReduce paper</a>. Your <tt>mapF()</tt> and <tt>reduceF()</tt> functions will differ a bit from those in the paper’s Section 2.1. Your <tt>mapF()</tt> will be passed the name of a file, as well as that file’s contents; it should split it into words, and return a Go slice of key/value pairs, of type <tt>mapreduce.KeyValue</tt>. Your <tt>reduceF()</tt> will be called once for each key, with a slice of all the values generated by <tt>mapF()</tt> for that key; it should return a single output value.

You can test your solution using:

<pre>$ cd "$GOPATH/src/main"$ go run wc.go master sequential pg-*.txtmaster: Starting Map/Reduce task wcseqMerge: read mrtmp.wcseq-res-0Merge: read mrtmp.wcseq-res-1Merge: read mrtmp.wcseq-res-2master: Map/Reduce task completed</pre>

The output will be in the file <tt>mrtmp.wcseq</tt>. We will test your implementation’s correctness with the following command, which should produce the following top 10 words:

<pre>$ sort -n -k2 mrtmp.wcseq | tail -10he: 34077was: 37044that: 37495I: 44502in: 46092a: 60558to: 74357of: 79727and: 93990the: 154024</pre>

(this sample result is also found in main/mr-testout.txt)

You can remove the output file and all intermediate files with:

<pre>$ rm mrtmp.*</pre>

To make testing easy for you, from the <tt>$GOPATH/src/main</tt> directory, run:

<pre>$ sh ./test-wc.sh</pre>

and it will report if your solution is correct or not.

<h2><a id="user-content-resources-and-advice" class="anchor" href="https://github.com/theoliao1998/Distributed-Systems/blob/master/1-2%2C3%20Sequential%20%26%20Distributed%20MapReduce/README1.md#resources-and-advice" aria-hidden="true"></a>Resources and Advice</h2>

<ul>

 <li>a good read on what strings are in Go is the <a href="https://blog.golang.org/strings" rel="nofollow">Go Blog on strings</a>.</li>

 <li>you can use <a href="https://golang.org/pkg/strings/#FieldsFunc" rel="nofollow"><tt>strings.FieldsFunc</tt></a> to split a string into components.</li>

 <li>the strconv package (<a href="https://golang.org/pkg/strconv/" rel="nofollow">http://golang.org/pkg/strconv/</a>) is handy to convert strings to integers etc.</li>

</ul>

<h2><a id="user-content-submitting-assignment" class="anchor" href="https://github.com/theoliao1998/Distributed-Systems/blob/master/1-2%2C3%20Sequential%20%26%20Distributed%20MapReduce/README1.md#submitting-assignment" aria-hidden="true"></a>Submitting Assignment</h2>

You hand in your assignment exactly as you’ve been letting us know your progress:

You should verify that you are able to see your final commit and your a12-handin tag on the Github page in your repository for this assignment.

You will receive full credit if your software passes the Sequential tests in <tt>test_test.go</tt> and <tt>test-wc.sh</tt>.

We will use the timestamp of your <strong>last</strong> tag for the purpose of calculating late days, and we will only grade that version of the code. (We’ll also know if you backdate the tag, don’t do that.)

Our test script is not a substitute for doing your own testing on the full go test cases detailed above. Before submitting, please run the full tests given above for both parts one final time. <b>You</b> are responsible for making sure your code works.

You will receive full credit for Part I if your software passes the Sequential tests (as run by the <tt>go test</tt> commands above) on the CS servers. You will receive full credit for Part II if your Map/Reduce word count output matches the correct output for the sequential execution above when run on the CS servers.

The final portion of your credit is determined by code quality tests, using the standard tools <tt>gofmt</tt> and <tt>go vet</tt>. You will receive full credit for this portion if all files submitted conform to the style standards set by <tt>gofmt</tt> and the report from <tt>go vet</tt> is clean for your mapreduce package (that is, produces no errors). If your code does not pass the <tt>gofmt</tt> test, you should reformat your code using the tool. You can also use the <a href="https://github.com/qiniu/checkstyle">Go Checkstyle</a> tool for advice to improve your code’s style, if applicable. Additionally, though not part of the graded cheks, it would also be advisable to produce code that complies with <a href="https://github.com/golang/lint">Golint</a> where possible.




<h2>PART 3</h2>

<h1>COS418 Assignment 1 (Part 3): Distributed Map/Reduce</h1>

<h2><a id="user-content-introduction" class="anchor" href="https://github.com/theoliao1998/Distributed-Systems/blob/master/1-2%2C3%20Sequential%20%26%20Distributed%20MapReduce/README2.md#introduction" aria-hidden="true"></a>Introduction</h2>

Part c continues the work from assignment 1.2 — building a Map/Reduce library as a way to learn the Go programming language and as a way to learn about fault tolerance in distributed systems. In this part of the assignment, you will tackle a distributed version of the Map/Reduce library, writing code for a master that hands out tasks to multiple workers and handles failures in workers. The interface to the library and the approach to fault tolerance is similar to the one described in the original <a href="http://research.google.com/archive/mapreduce-osdi04.pdf" rel="nofollow">MapReduce paper</a>. As with the previous part of this assignment, you will also complete a sample Map/Reduce application.

<h2><a id="user-content-software" class="anchor" href="https://github.com/theoliao1998/Distributed-Systems/blob/master/1-2%2C3%20Sequential%20%26%20Distributed%20MapReduce/README2.md#software" aria-hidden="true"></a>Software</h2>

You will use the same mapreduce package as in part b, focusing this time on the distributed mode.

Over the course of this assignment, you will have to modify <tt>schedule</tt> from <tt>schedule.go</tt>, as well as <tt>mapF</tt> and <tt>reduceF</tt> in <tt>main/ii.go</tt>.

As with the previous part of this assignment, you should not need to modify any other files, but reading them might be useful in order to understand how the other methods fit into the overall architecture of the system.

To get start, copy all source files from <code>assignment1-2/src</code> to <code>assignment1-3/src</code>

<pre># start from your 418 GitHub repo$ cd 418$ lsREADME.md     assignment1-1 assignment1-2 assignment1-3 assignment2   assignment3   assignment4   assignment5   setup.md$ cp -r assignment1-2/src/* assignment1-3/src/$ ls assignment1-3/srcmain      mapreduce</pre>

<h2><a id="user-content-part-i-distributing-mapreduce-tasks" class="anchor" href="https://github.com/theoliao1998/Distributed-Systems/blob/master/1-2%2C3%20Sequential%20%26%20Distributed%20MapReduce/README2.md#part-i-distributing-mapreduce-tasks" aria-hidden="true"></a>Part I: Distributing MapReduce tasks</h2>

One of Map/Reduce’s biggest selling points is that the developer should not need to be aware that their code is running in parallel on many machines. In theory, we should be able to take the word count code you wrote in Part II of assignment 1.2, and automatically parallelize it!

Our current implementation runs all the map and reduce tasks one after another on the master. While this is conceptually simple, it is not great for performance. In this part of the assignment, you will complete a version of MapReduce that splits the work up over a set of worker threads, in order to exploit multiple cores. Computing the map tasks in parallel and then the reduce tasks can result in much faster completion, but is also harder to implement and debug. Note that for this part of the assignment, the work is not distributed across multiple machines as in “real” Map/Reduce deployments, your implementation will be using RPC and channels to simulate a truly distributed computation.

To coordinate the parallel execution of tasks, we will use a special master thread, which hands out work to the workers and waits for them to finish. To make the assignment more realistic, the master should only communicate with the workers via RPC. We give you the worker code (<tt>mapreduce/worker.go</tt>), the code that starts the workers, and code to deal with RPC messages (<tt>mapreduce/common_rpc.go</tt>).

Your job is to complete <tt>schedule.go</tt> in the <tt>mapreduce</tt> package. In particular, you should modify <tt>schedule()</tt> in <tt>schedule.go</tt> to hand out the map and reduce tasks to workers, and return only when all the tasks have finished.

Look at <tt>run()</tt> in <tt>master.go</tt>. It calls your <tt>schedule()</tt> to run the map and reduce tasks, then calls <tt>merge()</tt> to assemble the per-reduce-task outputs into a single output file. <tt>schedule</tt> only needs to tell the workers the name of the original input file (<tt>mr.files[task]</tt>) and the task <tt>task</tt>; each worker knows from which files to read its input and to which files to write its output. The master tells the worker about a new task by sending it the RPC call <tt>Worker.DoTask</tt>, giving a <tt>DoTaskArgs</tt> object as the RPC argument.

When a worker starts, it sends a Register RPC to the master. <tt>master.go</tt> already implements the master’s <tt>Master.Register</tt> RPC handler for you, and passes the new worker’s information to <tt>mr.registerChannel</tt>. Your <tt>schedule</tt> should process new worker registrations by reading from this channel.

Information about the currently running job is in the <tt>Master</tt> struct, defined in <tt>master.go</tt>. Note that the master does not need to know which Map or Reduce functions are being used for the job; the workers will take care of executing the right code for Map or Reduce (the correct functions are given to them when they are started by <tt>main/wc.go</tt>).

To test your solution, you should use the same Go test suite as you did in Part I of assignment 1.2, except swapping out <tt>-run Sequential</tt> with <tt>-run TestBasic</tt>. This will execute the distributed test case without worker failures instead of the sequential ones we were running before:

<pre>$ go test -run TestBasic mapreduce/...</pre>

As before, you can get more verbose output for debugging if you set <tt>debugEnabled = true</tt> in <tt>mapreduce/common.go</tt>, and add <tt>-v</tt> to the test command above. You will get much more output along the lines of:

<pre>$ go test -v -run TestBasic mapreduce/...=== RUN   TestBasic/var/tmp/824-32311/mr8665-master: Starting Map/Reduce task testSchedule: 100 Map tasks (50 I/Os)/var/tmp/824-32311/mr8665-worker0: given Map task #0 on file 824-mrinput-0.txt (nios: 50)/var/tmp/824-32311/mr8665-worker1: given Map task #11 on file 824-mrinput-11.txt (nios: 50)/var/tmp/824-32311/mr8665-worker0: Map task #0 done/var/tmp/824-32311/mr8665-worker0: given Map task #1 on file 824-mrinput-1.txt (nios: 50)/var/tmp/824-32311/mr8665-worker1: Map task #11 done/var/tmp/824-32311/mr8665-worker1: given Map task #2 on file 824-mrinput-2.txt (nios: 50)/var/tmp/824-32311/mr8665-worker0: Map task #1 done/var/tmp/824-32311/mr8665-worker0: given Map task #3 on file 824-mrinput-3.txt (nios: 50)/var/tmp/824-32311/mr8665-worker1: Map task #2 done...Schedule: Map phase doneSchedule: 50 Reduce tasks (100 I/Os)/var/tmp/824-32311/mr8665-worker1: given Reduce task #49 on file 824-mrinput-49.txt (nios: 100)/var/tmp/824-32311/mr8665-worker0: given Reduce task #4 on file 824-mrinput-4.txt (nios: 100)/var/tmp/824-32311/mr8665-worker1: Reduce task #49 done/var/tmp/824-32311/mr8665-worker1: given Reduce task #1 on file 824-mrinput-1.txt (nios: 100)/var/tmp/824-32311/mr8665-worker0: Reduce task #4 done/var/tmp/824-32311/mr8665-worker0: given Reduce task #0 on file 824-mrinput-0.txt (nios: 100)/var/tmp/824-32311/mr8665-worker1: Reduce task #1 done/var/tmp/824-32311/mr8665-worker1: given Reduce task #26 on file 824-mrinput-26.txt (nios: 100)/var/tmp/824-32311/mr8665-worker0: Reduce task #0 done...Schedule: Reduce phase doneMerge: read mrtmp.test-res-0Merge: read mrtmp.test-res-1...Merge: read mrtmp.test-res-49/var/tmp/824-32311/mr8665-master: Map/Reduce task completed--- PASS: TestBasic (25.60s)PASSok  mapreduce25.613s</pre>

<h2><a id="user-content-part-ii-handling-worker-failures" class="anchor" href="https://github.com/theoliao1998/Distributed-Systems/blob/master/1-2%2C3%20Sequential%20%26%20Distributed%20MapReduce/README2.md#part-ii-handling-worker-failures" aria-hidden="true"></a>Part II: Handling worker failures</h2>

In this part you will make the master handle failed workers. MapReduce makes this relatively easy because workers don’t have persistent state. If a worker fails, any RPCs that the master issued to that worker will fail (e.g., due to a timeout). Thus, if the master’s RPC to the worker fails, the master should re-assign the task given to the failed worker to another worker.

An RPC failure doesn’t necessarily mean that the worker failed; the worker may just be unreachable but still computing. Thus, it may happen that two workers receive the same task and compute it. However, because tasks are idempotent, it doesn’t matter if the same task is computed twice — both times it will generate the same output. So, you don’t have to do anything special for this case. (Our tests never fail workers in the middle of task, so you don’t even have to worry about several workers writing to the same output file.)

You don’t have to handle failures of the master; we will assume it won’t fail. Making the master fault-tolerant is more difficult because it keeps persistent state that would have to be recovered in order to resume operations after a master failure. Much of the rest of this course is devoted to this challenge.

Your implementation must pass the two remaining test cases in <tt>test_test.go</tt>. The first case tests the failure of one worker, while the second test case tests handling of many failures of workers. Periodically, the test cases start new workers that the master can use to make forward progress, but these workers fail after handling a few tasks. To run these tests:

<pre>$ go test -run Failure mapreduce/...</pre>

<h2><a id="user-content-part-iii-inverted-index-generation" class="anchor" href="https://github.com/theoliao1998/Distributed-Systems/blob/master/1-2%2C3%20Sequential%20%26%20Distributed%20MapReduce/README2.md#part-iii-inverted-index-generation" aria-hidden="true"></a>Part III: Inverted index generation</h2>

Word count is a classical example of a Map/Reduce application, but it is not an application that many large consumers of Map/Reduce use. It is simply not very often you need to count the words in a really large dataset. For this application exercise, we will instead have you build Map and Reduce functions for generating an <em>inverted index</em>.

Inverted indices are widely used in computer science, and are particularly useful in document searching. Broadly speaking, an inverted index is a map from interesting facts about the underlying data, to the original location of that data. For example, in the context of search, it might be a map from keywords to documents that contain those words.

We have created a second binary in <tt>main/ii.go</tt> that is very similar to the <tt>wc.go</tt> you built earlier. You should modify <tt>mapF</tt> and <tt>reduceF</tt> in <tt>main/ii.go</tt> so that they together produce an inverted index. Running <tt>ii.go</tt> should output a list of tuples, one per line, in the following format:

<pre>$ go run ii.go master sequential pg-*.txt$ head -n5 mrtmp.iiseqA: 16 pg-being_ernest.txt,pg-dorian_gray.txt,pg-dracula.txt,pg-emma.txt,pg-frankenstein.txt,pg-great_expectations.txt,pg-grimm.txt,pg-huckleberry_finn.txt,pg-les_miserables.txt,pg-metamorphosis.txt,pg-moby_dick.txt,pg-sherlock_holmes.txt,pg-tale_of_two_cities.txt,pg-tom_sawyer.txt,pg-ulysses.txt,pg-war_and_peace.txtABC: 2 pg-les_miserables.txt,pg-war_and_peace.txtABOUT: 2 pg-moby_dick.txt,pg-tom_sawyer.txtABRAHAM: 1 pg-dracula.txtABSOLUTE: 1 pg-les_miserables.txt</pre>

If it is not clear from the listing above, the format is:

<pre>word: #documents documents,sorted,and,separated,by,commas</pre>

We will test your implementation’s correctness with the following command, which should produce these resulting last 10 items in the index:

<pre>$ sort -k1,1 mrtmp.iiseq | sort -snk2,2 mrtmp.iiseq | grep -v '16' | tail -10women: 15 pg-being_ernest.txt,pg-dorian_gray.txt,pg-dracula.txt,pg-emma.txt,pg-frankenstein.txt,pg-great_expectations.txt,pg-huckleberry_finn.txt,pg-les_miserables.txt,pg-metamorphosis.txt,pg-moby_dick.txt,pg-sherlock_holmes.txt,pg-tale_of_two_cities.txt,pg-tom_sawyer.txt,pg-ulysses.txt,pg-war_and_peace.txtwon: 15 pg-being_ernest.txt,pg-dorian_gray.txt,pg-dracula.txt,pg-frankenstein.txt,pg-great_expectations.txt,pg-grimm.txt,pg-huckleberry_finn.txt,pg-les_miserables.txt,pg-metamorphosis.txt,pg-moby_dick.txt,pg-sherlock_holmes.txt,pg-tale_of_two_cities.txt,pg-tom_sawyer.txt,pg-ulysses.txt,pg-war_and_peace.txtwonderful: 15 pg-being_ernest.txt,pg-dorian_gray.txt,pg-dracula.txt,pg-emma.txt,pg-frankenstein.txt,pg-great_expectations.txt,pg-grimm.txt,pg-huckleberry_finn.txt,pg-les_miserables.txt,pg-moby_dick.txt,pg-sherlock_holmes.txt,pg-tale_of_two_cities.txt,pg-tom_sawyer.txt,pg-ulysses.txt,pg-war_and_peace.txtwords: 15 pg-dorian_gray.txt,pg-dracula.txt,pg-emma.txt,pg-frankenstein.txt,pg-great_expectations.txt,pg-grimm.txt,pg-huckleberry_finn.txt,pg-les_miserables.txt,pg-metamorphosis.txt,pg-moby_dick.txt,pg-sherlock_holmes.txt,pg-tale_of_two_cities.txt,pg-tom_sawyer.txt,pg-ulysses.txt,pg-war_and_peace.txtworked: 15 pg-dorian_gray.txt,pg-dracula.txt,pg-emma.txt,pg-frankenstein.txt,pg-great_expectations.txt,pg-grimm.txt,pg-huckleberry_finn.txt,pg-les_miserables.txt,pg-metamorphosis.txt,pg-moby_dick.txt,pg-sherlock_holmes.txt,pg-tale_of_two_cities.txt,pg-tom_sawyer.txt,pg-ulysses.txt,pg-war_and_peace.txtworse: 15 pg-being_ernest.txt,pg-dorian_gray.txt,pg-dracula.txt,pg-emma.txt,pg-frankenstein.txt,pg-great_expectations.txt,pg-grimm.txt,pg-huckleberry_finn.txt,pg-les_miserables.txt,pg-moby_dick.txt,pg-sherlock_holmes.txt,pg-tale_of_two_cities.txt,pg-tom_sawyer.txt,pg-ulysses.txt,pg-war_and_peace.txtwounded: 15 pg-being_ernest.txt,pg-dorian_gray.txt,pg-dracula.txt,pg-emma.txt,pg-frankenstein.txt,pg-great_expectations.txt,pg-grimm.txt,pg-huckleberry_finn.txt,pg-les_miserables.txt,pg-moby_dick.txt,pg-sherlock_holmes.txt,pg-tale_of_two_cities.txt,pg-tom_sawyer.txt,pg-ulysses.txt,pg-war_and_peace.txtyes: 15 pg-being_ernest.txt,pg-dorian_gray.txt,pg-dracula.txt,pg-emma.txt,pg-great_expectations.txt,pg-grimm.txt,pg-huckleberry_finn.txt,pg-les_miserables.txt,pg-metamorphosis.txt,pg-moby_dick.txt,pg-sherlock_holmes.txt,pg-tale_of_two_cities.txt,pg-tom_sawyer.txt,pg-ulysses.txt,pg-war_and_peace.txtyounger: 15 pg-being_ernest.txt,pg-dorian_gray.txt,pg-dracula.txt,pg-emma.txt,pg-frankenstein.txt,pg-great_expectations.txt,pg-grimm.txt,pg-huckleberry_finn.txt,pg-les_miserables.txt,pg-moby_dick.txt,pg-sherlock_holmes.txt,pg-tale_of_two_cities.txt,pg-tom_sawyer.txt,pg-ulysses.txt,pg-war_and_peace.txtyours: 15 pg-being_ernest.txt,pg-dorian_gray.txt,pg-dracula.txt,pg-emma.txt,pg-frankenstein.txt,pg-great_expectations.txt,pg-grimm.txt,pg-huckleberry_finn.txt,pg-les_miserables.txt,pg-moby_dick.txt,pg-sherlock_holmes.txt,pg-tale_of_two_cities.txt,pg-tom_sawyer.txt,pg-ulysses.txt,pg-war_and_peace.txt</pre>

(this sample result is also found in main/mr-challenge.txt)

To make testing easy for you, from the <tt>$GOPATH/src/main</tt> directory, run:

<pre>$ sh ./test-ii.sh</pre>

and it will report if your solution is correct or not.

<h2><a id="user-content-resources-and-advice" class="anchor" href="https://github.com/theoliao1998/Distributed-Systems/blob/master/1-2%2C3%20Sequential%20%26%20Distributed%20MapReduce/README2.md#resources-and-advice" aria-hidden="true"></a>Resources and Advice</h2>

<ul>

 <li>The master should send RPCs to the workers in parallel so that the workers can work on tasks concurrently. You will find the <tt>go</tt> statement useful for this purpose and the <a href="https://golang.org/pkg/net/rpc/" rel="nofollow">Go RPC documentation</a>.</li>

 <li>The master may have to wait for a worker to finish before it can hand out more tasks. You may find channels useful to synchronize threads that are waiting for reply with the master once the reply arrives. Channels are explained in the document on <a href="https://golang.org/doc/effective_go.html#concurrency" rel="nofollow">Concurrency in Go</a>.</li>

 <li>The code we give you runs the workers as threads within a single UNIX process, and can exploit multiple cores on a single machine. Some modifications would be needed in order to run the workers on multiple machines communicating over a network. The RPCs would have to use TCP rather than UNIX-domain sockets; there would need to be a way to start worker processes on all the machines; and all the machines would have to share storage through some kind of network file system.</li>

 <li>The easiest way to track down bugs is to insert <tt>debug()</tt> statements, set <tt>debugEngabled = true</tt> in <tt>mapreduce/common.go</tt>, collect the output in a file with, e.g., <tt>go test -run TestBasic mapreduce/... &gt; out</tt>, and then think about whether the output matches your understanding of how your code should behave. The last step (thinking) is the most important.</li>

 <li>When you run your code, you may receive many errors like <tt>method has wrong number of ins</tt>. You can ignore all of these as long as your tests pass.</li>

</ul>

<h2><a id="user-content-submission" class="anchor" href="https://github.com/theoliao1998/Distributed-Systems/blob/master/1-2%2C3%20Sequential%20%26%20Distributed%20MapReduce/README2.md#submission" aria-hidden="true"></a>Submission</h2>

You hand in your assignment as before.

You should verify that you are able to see your final commit and tags on the Github page of your repository for this assignment.

You will receive full credit for Part I if your software passes <tt>TestBasic</tt> from <tt>test_test.go</tt> (the test given in Part I) on the CS servers. You will receive full credit for Part II if your software passes the tests with worker failures (the <tt>Failure</tt> pattern to <tt>go test</tt> given in Part II) on the CS servers. You will receive full credit for Part II if your index output matches the correct output when run on the CS servers.

The final portion of your credit is determined by code quality tests, using the standard tools <tt>gofmt</tt> and <tt>go vet</tt>. You will receive full credit for this portion if all files submitted conform to the style standards set by <tt>gofmt</tt> and the report from <tt>go vet</tt> is clean for your mapreduce package (that is, produces no errors). If your code does not pass the <tt>gofmt</tt> test, you should reformat your code using the tool. You can also use the <a href="https://github.com/qiniu/checkstyle">Go Checkstyle</a> tool for advice to improve your code’s style, if applicable. Additionally, though not part of the graded cheks, it would also be advisable to produce code that complies with <a href="https://github.com/golang/lint">Golint</a> where possible.

5/5 - (2 votes)

<pre>$ git commit -am <span class="pl-s"><span class="pl-pds">"</span>[you fill me in]<span class="pl-pds">"</span></span>$ git tag -a -m <span class="pl-s"><span class="pl-pds">"</span>i finished assignment 1-2<span class="pl-pds">"</span></span> a12-handin$ git push origin master$ git push origin a12-handin$</pre>