JDK1.8 JUC源码分析(3) AbstractQueuedSynchronizer
=====

##功能介绍

* AbstractQueuedSynchronizer(AQA)是JUC提供的基础同步类，JUC中其他同步机制(Lock、CountDownLatch)都依靠AQA进行实现

* AQA提供2中模式独占模式(用于实现Lock等),共享模式(用于实现Semaphore等),AQA内部包含一个FIFO等待队列用于管理没有获取控制权的线程

* AQA内部还提供了类似于Object类的wait，nofity，nofityAll 

* AQA内部有一个int(AbstractQueuedSynchronizer下是long)变量用于表示状态，各个同步组件可灵活运用这个变量如果Lock用这个变量表示递归锁当前的重入次数，CountDownLatch中用于表示闭锁还多少次可打开

* AQA的使用方式为作为同步组件的内部类提供同步机制

代码分析
=====

