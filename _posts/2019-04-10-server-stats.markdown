---
layout: post
title:  "How much memory do you need?"
date:   2019-04-10
categories: analysis 
excerpt: "Probably much less than we think.
          A look into resource utilization on the Hoffman2 high performance
          cluster at UCLA."

---

> Highlights of this post:
> * [Jobs are using a small proportion of the resources they request](#the-problem)
> * [Monitor your own usage to optimize Hoffman2 throughput](#a-fix)


<br>

UCLA's own [Hoffman2] cluster gives researchers access to huge amounts of
computational resources (Over 20,000 cores and 50TB of memory!). There are
currently over 3,000 registered users that keep the server quite busy,
often near 100% according to the [reported statistics] at the time of writing.

So how do we use the cluster? A typical workflow would be:

1. Write a script to do something
2. Submit the script to the queueing system

When we submit the script, we have to specify how much time and memory we
*think* the job will need. Hoffman2 uses the Univa Grid Engine for job
scheduling, so a job submission command might look like:

{% highlight bash %}
qsub -N job_name -l h_data=8G,h_rt=24:00:00 my_script.sh
{% endhighlight %}

Then you wait... When the job runs, the
scheduler will reserve the amount of resources you requested for your script
to run. So in the above example, we get 1 core and 8 GB RAM for 24 hours all
to ourselves.

Your job will be mercilessly killed if you exceed the memory or time you
requested, so it's common practice to overestimate how much resources you need.
Overestimation of time is not a huge deal since resources will be freed when
your job finishes, but it may cause your job to sit on the queue for longer.
On the other hand, overestimation of memory (or even worse, cores) makes these
resources unavailable for other tasks for the duration of your job.

<br>
*Excessive overestimation increases wait times and decreases computation throughput.*
<br>
<br>

--- 

<br>
<h3 id="the-problem">How much are we overestimating?</h3>

I pulled accounting data from Hoffman2 (which is read-accessible to all users) that documents every job run on the cluster, separated by month. We'll take a look into March of 2019.

The accounting data provides [quite a few metrics] for each job. We can see the amount of time and memory requested. We can also see how much time and memory was actually used by the job. Let's take a look at the differences in requested and utilized resources across all the jobs.

I calculated the memory overestimation of a job by subtracting the peak memory usage from the total memory requested (sometimes this is negative, indicating that a job went over the limit, but we only looked at positive values here). For this visualization, I also removed around 1,800 outlier jobs that had over 50 GB of unused memory (a few had 256 GB overestimated). 

<br>
<div class="parent">
  <div class="container">
  <center><img src="/_img/posts/server-stats/hist_memory_overestimate_all_jobs_march19.png" width="450" height="400" ></center>
  </div>
</div>
<br>

Overall, we can observe a mean of roughly 5 GB overestimated per job and mode of 2 GB after rounding. For reference, jobs use 2.6 GB of memory on average. It should be noted that failed jobs can exaggerate the impact of memory overestimation, since they typically die fairly quickly and do not hold up the resources for that long. Roughly three percent of jobs had non-zero exit status, which usually indicates that the job failed. When I remove these jobs in these analyses, the above findings do not change.

Jobs vary significantly in how much memory they need. If a job requires 100GB of memory and 1GB is unused, it is much less of a problem than if a job only needing a few megabytes requests 1GB. With this is mind, I calculated the proportion of requested memory each job is actually using. I restricted the data to jobs that had zero-exit status and had memory usage less than or equal to the amount requested.  

<br>
<div class="parent">
  <div class="container">
<center><img src="/_img/posts/server-stats/hist_memory_proportion_all_jobs_march19.png" width="450" height="400" ></center>
  </div>
</div>
<br>

Most jobs use *zero* percent of the memory they are requesting. On average (which is less meaningful for skewed distributions such as these), jobs use around 25% of the memory they have reserved. Proper use of the queuing system would give us a mirrored image of this histogram. In this ideal world, jobs would tend to use most of the memory they request.

*Update* - While looking at my own jobs, I noticed that actual resource usage for interactive nodes is reported as zero for both time and memory. I'm not sure if this problem is limited to interactive nodes, but even if we remove the all the jobs that report zero percent memory usage, most jobs still only use around 10% of their requested memory (average around 32%). Hopefully this bug is fixed in newer versions of UGE (at time time of writing, Hoffman2 is running UGE 8.0.1 and the latest available is 8.5.X). 

<br>
**Jobs tend to use a small fraction of the memory they are allocated. How are we doing on time?**
<br>
<br>

Similarly, I calculated time ovestimation as the difference between the requested and actual duration of each job. Again, I restricted the data to jobs that successfully ran (non-zero exit statuses) and I did not show about one percent of outlier jobs that have a difference over 100 hours.

<br>
<div class="parent">
  <div class="container">
<center><img src="/_img/posts/server-stats/hist_time_overestimate_all_jobs_march19.png" width="450" height="400" ></center>
  </div>
</div>
<br>

Here, we can see a bimodal distribution. There is a peak around zero indicating that many jobs are great at estimating how much time they need, even when I removed jobs that are killed for running too long. The second peak at 23 hours (indicated by the dotted gray line) indicates that many jobs request about a day too much. For reference, the average actual runtime of a job was around 1.5 hours. This behavior is a consequence of the resource limits on shared resources in Hoffman2 (you can request a maximum of one day if you are using a node available to all users). As stated previously, time overestimation does not place a huge burden on the server, since jobs will exit whenever they actually complete. 

<br>

---

<br>


**To summarize, we find:**
* On average, jobs are given **four times** as much memory as they actually need.
* Though not as harmful, short jobs are often submitted with 24-hour requests.

<br>

---

<br>
<h3 id="a-fix">Resource overestimation is excessive. What can we do?</h3>

Keep track of what your jobs need. You can check your jobs in the current month with the following command:
{% highlight bash %}
qacct -o USER -j PATTERN
{% endhighlight %}
where USER is your server username and PATTERN is a Regex to filter for job names. If you leave PATTERN empty (but still with the `-j` flag) you'll see all jobs submitted this month. You'll get a lot of information (not the requested resource amounts for some reason), so you can filter for just the name, runtime, and peak memory usage with the following command:

{% highlight bash %}
qacct -o USER -j PATTERN | grep -E "^jobname|^ru_wallclock|^maxvmem" | awk 'ORS=NR%3?RS:RS RS'
{% endhighlight %}

To summarize these stats, I wrote a [script] that will describe memory requests and usage for a particular type of job over a specified month (previous month by default and currently does not work for current month). The output might look like this:

{% highlight console %}
$ ./check_my_usage -p rrstar*
Parsing accounting file (/u/systems/UGE8.0.1vm/hoffman2/common/accounting-2019-03)
for jobs belonging to bjew
Filtering for jobs with name containing pattern: rrstar*
Found 45 matching jobs.
On average, these jobs:
        -request 32.00 GB of memory.
        -will have a peak usage of 13.81 GB of memory
        -leave 18.19 GB unused
        -use 43.16% of requested memory
The highest observed memory usage among these jobs was 13.81 GB.
{% endhighlight %}

Next time I run STAR, I'll probably request 15 GB of memory.

There's no perfect answer to how much extra memory you should give your job to avoid getting it killed, but if you see that at most your jobs are using 13 GB of RAM, you might not need to request 32GB. Using 85% of the requested memory seems like a reasonable guideline, but anything above the current trends we're seeing around zero percent would make Hoffman2 more efficient. Your jobs will probably spend less time in the queue!

[Hoffman2]: http://www.idre.ucla.edu/
[reported statistics]: https://idre.ucla.edu/resources/hpc/h2stats
[quite a few metrics]: http://gridscheduler.sourceforge.net/htmlman/htmlman5/accounting.html
[script]: https://github.com/brandonjew/server-stats/blob/master/check_my_usage
