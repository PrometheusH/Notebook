#### Spark RDD 容错的四大核心要点

1. Stage输出失败，上层调度器DAGScheduler重试

   DAGScheduler.scala

   ```scala
   /**
      * Resubmit any failed stages. Ordinarily called after a small amount of time has passed since
      * the last fetch failure.
      */
     private[scheduler] def resubmitFailedStages() {
         //判断是否存在失败的Stages
       if (failedStages.size > 0) {
         // Failed stages may be removed by job cancellation, so failed might be empty even if
         // the ResubmitFailedStages event has been scheduled.
         // 失败的阶段可以通过作业取消删除，如果ResubmitFailedStages事件已调度，失败将是空值
         logInfo("Resubmitting failed stages")
         clearCacheLocs()
         // 获取所有失败Stage的列表
         val failedStagesCopy = failedStages.toArray
         //清空failedStages
         failedStages.clear()
         // 对之前获取的所有失败的Stage，根据JobId排序后逐一重试
         for (stage <- failedStagesCopy.sortBy(_.firstJobId)) {
           submitStage(stage)
         }
       }
     }
   ```

2. 计算过程中，Task内部任务失败，底层调度器重试

   TaskSetManager.scala

   ```scala
   /**
      * Marks the task as failed, re-adds it to the list of pending tasks, and notifies the
      * DAG Scheduler.
      * 标记task为失败，再次添加到未解决tasks列表，并通知DAGScheduler
      */
     def handleFailedTask(tid: Long, state: TaskState, reason: TaskFailedReason) {
       val info = taskInfos(tid)
       if (info.failed || info.killed) {
         return
       }
       removeRunningTask(tid)
       info.markFinished(state, clock.getTimeMillis())
       val index = info.index
       copiesRunning(index) -= 1
       var accumUpdates: Seq[AccumulatorV2[_, _]] = Seq.empty
       val failureReason = s"Lost task ${info.id} in stage ${taskSet.id} (TID $tid, ${info.host}," +
         s" executor ${info.executorId}): ${reason.toErrorString}"
       val failureException: Option[Throwable] = reason match {
         case fetchFailed: FetchFailed =>
           logWarning(failureReason)
           if (!successful(index)) {
             successful(index) = true
             tasksSuccessful += 1
           }
           isZombie = true
   
           if (fetchFailed.bmAddress != null) {
             blacklistTracker.foreach(_.updateBlacklistForFetchFailure(
               fetchFailed.bmAddress.host, fetchFailed.bmAddress.executorId))
           }
   
           None
   
         case ef: ExceptionFailure =>
           // ExceptionFailure's might have accumulator updates
           accumUpdates = ef.accums
           if (ef.className == classOf[NotSerializableException].getName) {
             // If the task result wasn't serializable, there's no point in trying to re-execute it.
             logError("Task %s in stage %s (TID %d) had a not serializable result: %s; not retrying"
               .format(info.id, taskSet.id, tid, ef.description))
             abort("Task %s in stage %s (TID %d) had a not serializable result: %s".format(
               info.id, taskSet.id, tid, ef.description))
             return
           }
           val key = ef.description
           val now = clock.getTimeMillis()
           val (printFull, dupCount) = {
             if (recentExceptions.contains(key)) {
               val (dupCount, printTime) = recentExceptions(key)
               if (now - printTime > EXCEPTION_PRINT_INTERVAL) {
                 recentExceptions(key) = (0, now)
                 (true, 0)
               } else {
                 recentExceptions(key) = (dupCount + 1, printTime)
                 (false, dupCount + 1)
               }
             } else {
               recentExceptions(key) = (0, now)
               (true, 0)
             }
           }
           if (printFull) {
             logWarning(failureReason)
           } else {
             logInfo(
               s"Lost task ${info.id} in stage ${taskSet.id} (TID $tid) on ${info.host}, executor" +
                 s" ${info.executorId}: ${ef.className} (${ef.description}) [duplicate $dupCount]")
           }
           ef.exception
   
         case e: ExecutorLostFailure if !e.exitCausedByApp =>
           logInfo(s"Task $tid failed because while it was being computed, its executor " +
             "exited for a reason unrelated to the task. Not counting this failure towards the " +
             "maximum number of failures for the task.")
           None
   
         case e: TaskFailedReason =>  // TaskResultLost, TaskKilled, and others
           logWarning(failureReason)
           None
       }
   
       sched.dagScheduler.taskEnded(tasks(index), reason, null, accumUpdates, info)
   
       if (!isZombie && reason.countTowardsTaskFailures) {
         assert (null != failureReason)
         taskSetBlacklistHelperOpt.foreach(_.updateBlacklistForFailedTask(
           info.host, info.executorId, index, failureReason))
         // 对失败Task的numFailures加一
         numFailures(index) += 1
         // 判断失败的Task计数是否大于最大的失败次数，如果大于，则输出日志，并不再重试
         if (numFailures(index) >= maxTaskFailures) {
           logError("Task %d in stage %s failed %d times; aborting job".format(
             index, taskSet.id, maxTaskFailures))
           abort("Task %d in stage %s failed %d times, most recent failure: %s\nDriver stacktrace:"
             .format(index, taskSet.id, maxTaskFailures, failureReason), failureException)
           return
         }
       }
   
       if (successful(index)) {
         logInfo(s"Task ${info.id} in stage ${taskSet.id} (TID $tid) failed, but the task will not" +
           s" be re-executed (either because the task failed with a shuffle data fetch failure," +
           s" so the previous stage needs to be re-run, or because a different copy of the task" +
           s" has already succeeded).")
       } else {
         addPendingTask(index)
       }
   	// 如果运行的Task为0时，则完成Task步骤
       maybeFinishTaskSet()
     }
   ```

3. RDD Lineage容错

   > 在窄依赖中，一个子RDD的分区中的数据只来自一个父RDD，所以重算时不会存在**冗余计算**。
   >
   > 在宽依赖中，丢失一个子RDD分区，重新计算父RDD的所有分区数据，存在**冗余计算**。

4. checkpoint容错

   通过将RDD写入Disk缓存

   适用场景：

   1. Lineage过长时，如果重算开销太大，需要在中间设定检查点。
   2. 适合在宽依赖完成时做checkpoint，避免为Lineage重新计算而带来的冗余计算。





































































