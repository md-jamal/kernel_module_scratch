# workqueue

http://www.ibm.com/developerworks/jp/linux/library/l-tasklets/

 * _defer work into a kernel thread_
 * process context
   * 割り込みコンテキストでしか利用出来ない softirq, tasklet と比較するとよい
 * _worker threads_ と呼ばれるカーネルスレッド

```
root      1938  0.0  0.0      0     0 ?        S    14:40   0:00  \_ [workqueue/0]
root      1939  0.0  0.0      0     0 ?        S    14:40   0:00  \_ [workqueue/1]
```

## API

### queue

 * struct workqueue_struct

 ```c
/*
 * The per-CPU workqueue (if single thread, we always use the first
 * possible cpu).
 */
struct cpu_workqueue_struct {

	spinlock_t lock;

	struct list_head worklist;
	wait_queue_head_t more_work;
	struct work_struct *current_work;

	struct workqueue_struct *wq;
	struct task_struct *thread;
} ____cacheline_aligned;
```

 * create_workqueue
   * CPU数分のカーネルスレッドが作成される
   * 1個のカーネルスレッドが欲しいなら create_singlethread_workqueue
   * enqueue した work がどのCPUで実行されるかはどうやって決まる?
     * get_cpu()
 * flush_workqueue
   * キューを一掃する
 * destroy_workqueue

### worker

 * struct work_struct
 * INIT_WORK
 * queue_work

### 遅延ワーカー

 * struct delayed_work
 * INIT_DELAYED_WORK
 * queue_delayed_work
   * 内部で add_timer を使って遅延を実現している

### keventd_wq

 * schedule_work
 * schedule_delayed_work

共有の workqueue ( global workqueue  = keventd_wq ) にタスクを投げる

 * keventd_wq は _events/N_ のカーネルスレッド
```
[vagrant@vagrant-centos65 ~]$ ps aux | grep events
root        19  0.0  0.0      0     0 ?        S    11:17   0:00 [events/0]
root        20  0.0  0.0      0     0 ?        S    11:17   0:00 [events/1]
root        21  0.0  0.0      0     0 ?        S    11:17   0:00 [events/2]
root        22  0.0  0.0      0     0 ?        S    11:17   0:00 [events/3]
```

 * schedule_work の中身。 queue_work に keventd_wq を呼び出すラッパー
```c
static struct workqueue_struct *keventd_wq __read_mostly;

/**
 * schedule_work - put work task in global workqueue
 * @work: job to be done
 *
 * Returns zero if @work was already on the kernel-global workqueue and
 * non-zero otherwise.
 *
 * This puts a job in the kernel-global workqueue if it was not already
 * queued and leaves it in the same position on the kernel-global
 * workqueue otherwise.
 */
int schedule_work(struct work_struct *work)
{
	return queue_work(keventd_wq, work);
}
EXPORT_SYMBOL(schedule_work);
```

 * 自前で workqueue_struct を作らなくていいのでお手軽。
   * 滅多によびだししない work_struct であればメモリの節約になる
 * ただし他のコンポーネントからも呼び出されるので実行時間が長いと副作用が大きい

## queue_work から内部実装を見る

 * queue_work は queue_work_on のラッパー
   * 呼び出したコンテキストで使用されている CPU上に work を enqueue する

```
/**
 * queue_work - queue work on a workqueue
 * @wq: workqueue to use
 * @work: work to queue
 *
 * Returns 0 if @work was already on a queue, non-zero otherwise.
 *
 * We queue the work to the CPU on which it was submitted, but if the CPU dies
 * it can be processed by another CPU.
 */
int queue_work(struct workqueue_struct *wq, struct work_struct *work)
{
	int ret;

	ret = queue_work_on(get_cpu(), wq, work);
	put_cpu();

	return ret;
}
EXPORT_SYMBOL_GPL(queue_work);
```

work が pending されている場合は enqueue されない

```c
/**
 * queue_work_on - queue work on specific cpu
 * @cpu: CPU number to execute work on
 * @wq: workqueue to use
 * @work: work to queue
 *
 * Returns 0 if @work was already on a queue, non-zero otherwise.
 *
 * We queue the work to a specific CPU, the caller must ensure it
 * can't go away.
 */
int
queue_work_on(int cpu, struct workqueue_struct *wq, struct work_struct *work)
{
	int ret = 0;

	if (!test_and_set_bit(WORK_STRUCT_PENDING, work_data_bits(work))) {
		BUG_ON(!list_empty(&work->entry));
		__queue_work(wq_per_cpu(wq, cpu), work);
		ret = 1;
	}
	return ret;
}
EXPORT_SYMBOL_GPL(queue_work_on);
```

cpu_workqueue_struct の spinlock を取って ...

```c
static void __queue_work(struct cpu_workqueue_struct *cwq,
			 struct work_struct *work)
{
	unsigned long flags;

	spin_lock_irqsave(&cwq->lock, flags);
	insert_work(cwq, work, &cwq->worklist);
	spin_unlock_irqrestore(&cwq->lock, flags);
}
```

cpu_workqueue_struct の more_work を起床させている

```
static void insert_work(struct cpu_workqueue_struct *cwq,
			struct work_struct *work, struct list_head *head)
{
	trace_workqueue_insertion(cwq->thread, work);

	set_wq_data(work, cwq);
	/*
	 * Ensure that we get the right work->data if we see the
	 * result of list_add() below, see try_to_grab_pending().
	 */
	smp_wmb();
	list_add_tail(&work->entry, head);
    // ここで起床させている
	wake_up(&cwq->more_work);
}
```

## xfs でも使ってる

  * https://github.com/hiboma/hiboma/blob/master/xfs.md