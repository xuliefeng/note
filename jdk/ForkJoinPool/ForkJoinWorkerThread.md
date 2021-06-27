# 线程简介

ForkJoinPool 池使用的线程是自定义过的线程对象，里面关联了需要工作的线程池和任务队列

## 线程创建

ForkJoinPool 的线程池中通过 ForkJoinWorkerThreadFactory 的模式来创建，默认使用 DefaultForkJoinWorkerThreadFactory 来创建，该线程为后台运行线程

ForkJoinPool 创建时，通过将 ForkJoinPool 对象传入则两者关联起来，然后通过 ForkJoinPool 对象来注册获取对应的 WorkQueue 对象。该创建的线程关联的 WorkQueue 的位置处于任务队列数组的奇数位上，并且该 WorkQueue 的 hint 值为递增的 SEED_INCREMENT 值，scanState 的值初始化为 对应的 WorkQueue 任务队列数组下标。

## 线程的启动

由于该线程创建时默认会创建一个新的任务队列，所以在启动时判断任务队列中的任务数组是否为空。如果不为空则不执行
