---
title: ThreadPoolExecutor队列满时提交任务阻塞
date: 2019-08-15 11:19:15
tags:
	- java
---

项目中有一个需求：给线程池提交任务的时候，如果任务队列已满，需要ThreadPoolExecutor.execute调用阻塞等待。google了相关的资料，记录在这里，供有同样需求的同行参考。

https://stackoverflow.com/questions/4521983/java-executorservice-that-blocks-on-submission-after-a-certain-queue-size/4522411#4522411

ThreadPoolExecutor相关的几个点：
（1）execute提交任务的时候，会调用指定队列的offer方法，如果offer方法返回失败，则表示队列已满。如果此时，corePoolSize < maximumPoolSize 会发起新的线程执行新提交的任务；如果 corePoolSize == maximumPoolSize， 则任务提交失败，会调用RejectedExecutionHandler处理
（2）java提供了四个内置的RejectedExecutionHandler， 如

```
    /**
     * A handler for rejected tasks that throws a
     * {@code RejectedExecutionException}.
     */
    public static class AbortPolicy implements RejectedExecutionHandler {
        /**
         * Creates an {@code AbortPolicy}.
         */
        public AbortPolicy() { }

        /**
         * Always throws RejectedExecutionException.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         * @throws RejectedExecutionException always
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }

    /**
     * A handler for rejected tasks that silently discards the
     * rejected task.
     */
    public static class DiscardPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardPolicy}.
         */
        public DiscardPolicy() { }

        /**
         * Does nothing, which has the effect of discarding task r.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
    }

    /**
     * A handler for rejected tasks that discards the oldest unhandled
     * request and then retries {@code execute}, unless the executor
     * is shut down, in which case the task is discarded.
     */
    public static class DiscardOldestPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardOldestPolicy} for the given executor.
         */
        public DiscardOldestPolicy() { }

        /**
         * Obtains and ignores the next task that the executor
         * would otherwise execute, if one is immediately available,
         * and then retries execution of task r, unless the executor
         * is shut down, in which case task r is instead discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
```



解法1: 重写队列的offer方法，让其变为一个阻塞调用
使用这种方法时，线程数最多只能到corePoolSize个，相当于maximumPoolSize的设置无效；

```
/**
     * 重写offer为阻塞操作
     */
    private static class MyBlockingQueue<T> extends LinkedBlockingQueue<T> {

        public MyBlockingQueue(int size) {
            super(size);
        }

        @Override
        public boolean offer(T t) {
            try {
                put(t);
                return true;
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }

            return false;
        }
    }
    
        /**
     * 固定线程池, block when queue is filled up
     */
    public static ThreadPoolExecutor newFixedThreadPool(int nThreads, int queueSize, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(
                nThreads, nThreads, 0L, TimeUnit.SECONDS, new MyBlockingQueue<>(queueSize), threadFactory);
    }

```


解法2: 基于java信号量

```
public class BoundedExecutor {
    private final Executor exec;
    private final Semaphore semaphore;

    public BoundedExecutor(Executor exec, int bound) {
        this.exec = exec;
        this.semaphore = new Semaphore(bound);
    }

    public void submitTask(final Runnable command)
            throws InterruptedException, RejectedExecutionException {
        semaphore.acquire();
        try {
            exec.execute(new Runnable() {
                public void run() {
                    try {
                        command.run();
                    } finally {
                        semaphore.release();
                    }
                }
            });
        } catch (RejectedExecutionException e) {
            semaphore.release();
            throw e;
        }
    }
}
```