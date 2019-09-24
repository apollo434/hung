#### The key function

**overview**

![Alt text](/pic/hung.jpg)


**Init**
```
static int __init hung_task_init(void)
{
	atomic_notifier_chain_register(&panic_notifier_list, &panic_block);
	watchdog_task = kthread_run(watchdog, NULL, "khungtaskd");

	return 0;
}
subsys_initcall(hung_task_init);

```

```
static struct notifier_block panic_block = {
	.notifier_call = hung_task_panic,
};

```

```
/*
 * kthread which checks for tasks stuck in D state
 */
static int watchdog(void *dummy)
{
	set_user_nice(current, 0);

	for ( ; ; ) {
		unsigned long timeout = sysctl_hung_task_timeout_secs;

		while (schedule_timeout_interruptible(timeout_jiffies(timeout)))
			timeout = sysctl_hung_task_timeout_secs;

		if (atomic_xchg(&reset_hung_task, 0))
			continue;

		check_hung_uninterruptible_tasks(timeout);
	}

	return 0;
}

```

```
void reset_hung_task_detector(void)
{
	atomic_set(&reset_hung_task, 1);
}
EXPORT_SYMBOL_GPL(reset_hung_task_detector);

```

**How to check the hung**

```
/*
 * Check whether a TASK_UNINTERRUPTIBLE does not get woken up for
 * a really long time (120 seconds). If that happens, print out
 * a warning.
 */
static void check_hung_uninterruptible_tasks(unsigned long timeout)
{
	int max_count = sysctl_hung_task_check_count;
	int batch_count = HUNG_TASK_BATCHING;
	struct task_struct *g, *t;

	/*
	 * If the system crashed already then all bets are off,
	 * do not report extra hung tasks:
	 */
	if (test_taint(TAINT_DIE) || did_panic)
		return;

	rcu_read_lock();
	for_each_process_thread(g, t) {
		if (!max_count--)
			goto unlock;
		if (!--batch_count) {
			batch_count = HUNG_TASK_BATCHING;
			if (!rcu_lock_break(g, t))
				goto unlock;
		}
		/* use "==" to skip the TASK_KILLABLE tasks waiting on NFS */
		if (t->state == TASK_UNINTERRUPTIBLE)
			check_hung_task(t, timeout);
	}
 unlock:
	rcu_read_unlock();
}

```

```
static int
hung_task_panic(struct notifier_block *this, unsigned long event, void *ptr)
{
	did_panic = 1;

	return NOTIFY_DONE;
}

```
**the key function of checking the hung**

```
static void check_hung_task(struct task_struct *t, unsigned long timeout)
{
	unsigned long switch_count = t->nvcsw + t->nivcsw;

	/*
	 * Ensure the task is not frozen.
	 * Also, skip vfork and any other user process that freezer should skip.
	 */
	if (unlikely(t->flags & (PF_FROZEN | PF_FREEZER_SKIP)))
	    return;

	/*
	 * When a freshly created task is scheduled once, changes its state to
	 * TASK_UNINTERRUPTIBLE without having ever been switched out once, it
	 * musn't be checked.
	 */
	if (unlikely(!switch_count))
		return;

	if (switch_count != t->last_switch_count) {
		t->last_switch_count = switch_count;
		return;
	}

	trace_sched_process_hang(t);

	if (!sysctl_hung_task_warnings)
		return;

	if (sysctl_hung_task_warnings > 0)
		sysctl_hung_task_warnings--;

```

```
	/*
	 * Ok, the task did not get scheduled for more than 2 minutes,
	 * complain:
	 */
	pr_err("INFO: task %s:%d blocked for more than %ld seconds.\n",
		t->comm, t->pid, timeout);
	pr_err("      %s %s %.*s\n",
		print_tainted(), init_utsname()->release,
		(int)strcspn(init_utsname()->version, " "),
		init_utsname()->version);
	pr_err("\"echo 0 > /proc/sys/kernel/hung_task_timeout_secs\""
		" disables this message.\n");
	sched_show_task(t);
	debug_show_held_locks(t);

	touch_nmi_watchdog();

```

```

	if (sysctl_hung_task_panic) {
		trigger_all_cpu_backtrace();
		panic("hung_task: blocked tasks");
	}

```
