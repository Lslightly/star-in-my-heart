---
tags:
  - Go
  - Profiling
---

# [conbon/pprof2csv](https://github.com/conbon/pprof2csv)

go1.24.2

在通过 `go test -cpuprofile ...` 获得 CPU Profile 之后，通过 `go tool pprof -text cpu-100-default.out` 得到如下输出。

```bash
File: actor.test
Build ID: 48676b8e95e7c6e7a73f8f1f7fba2e545eb023eb
Type: cpu
Time: 2026-01-16 12:46:03 CST
Duration: 1.40s, Total samples = 2.32s (165.29%)
Showing nodes accounting for 2.27s, 97.84% of 2.32s total
Dropped 12 nodes (cum <= 0.01s)
      flat  flat%   sum%        cum   cum%
     0.57s 24.57% 24.57%      0.65s 28.02%  github.com/asynkron/protoactor-go/actor.(*actorContext).defaultReceive
     0.46s 19.83% 44.40%      0.46s 19.83%  runtime.(*gcWork).tryGetFast (inline)
     0.45s 19.40% 63.79%      1.10s 47.41%  github.com/asynkron/protoactor-go/actor.(*actorContext).processMessage
     0.18s  7.76% 71.55%      0.22s  9.48%  runtime.(*lfstack).pop (inline)
     0.11s  4.74% 76.29%      1.21s 52.16%  github.com/asynkron/protoactor-go/actor.BenchmarkActorContext_ProcessMessageNoMiddleware
     0.09s  3.88% 80.17%      0.09s  3.88%  runtime.(*lfstack).push
     0.07s  3.02% 83.19%      0.20s  8.62%  runtime.greyobject
     0.06s  2.59% 85.78%      0.07s  3.02%  runtime.(*gcBits).bitp (inline)
     0.06s  2.59% 88.36%      0.06s  2.59%  runtime.markBits.setMarked (inline)
     0.04s  1.72% 90.09%      0.04s  1.72%  runtime.taggedPointer.pointer (inline)
     0.03s  1.29% 91.38%      0.06s  2.59%  github.com/asynkron/protoactor-go/actor.ReceiveFunc.Receive
     0.03s  1.29% 92.67%      0.03s  1.29%  github.com/asynkron/protoactor-go/actor.init.func3
     0.03s  1.29% 93.97%      1.05s 45.26%  runtime.gcDrain
     0.03s  1.29% 95.26%      0.25s 10.78%  runtime.scanobject
     0.02s  0.86% 96.12%      0.02s  0.86%  github.com/asynkron/protoactor-go/actor.UnwrapEnvelopeMessage (inline)
     0.02s  0.86% 96.98%      0.02s  0.86%  runtime.gcMarkWorkAvailable (inline)
     0.01s  0.43% 97.41%      0.02s  0.86%  runtime.(*mspan).heapBitsSmallForAddr
     0.01s  0.43% 97.84%      0.13s  5.60%  runtime.getempty
         0     0% 97.84%      0.02s  0.86%  github.com/asynkron/protoactor-go/actor.(*actorContext).Message (inline)
         0     0% 97.84%      0.02s  0.86%  runtime.(*gcControllerState).findRunnableGCWorker
         0     0% 97.84%      0.18s  7.76%  runtime.(*gcWork).balance
         0     0% 97.84%      0.12s  5.17%  runtime.(*gcWork).tryGet
         0     0% 97.84%      0.07s  3.02%  runtime.(*mspan).markBitsForIndex (inline)
         0     0% 97.84%      0.02s  0.86%  runtime.(*mspan).typePointersOfUnchecked
         0     0% 97.84%      0.02s  0.86%  runtime.findRunnable
         0     0% 97.84%      1.05s 45.26%  runtime.gcBgMarkWorker
         0     0% 97.84%      0.02s  0.86%  runtime.gcBgMarkWorker.func1
         0     0% 97.84%      1.05s 45.26%  runtime.gcBgMarkWorker.func2
         0     0% 97.84%      0.38s 16.38%  runtime.gcDrainMarkWorkerDedicated (inline)
         0     0% 97.84%      0.67s 28.88%  runtime.gcDrainMarkWorkerIdle (inline)
         0     0% 97.84%      0.18s  7.76%  runtime.handoff
         0     0% 97.84%      0.04s  1.72%  runtime.lfstackUnpack (inline)
         0     0% 97.84%      0.05s  2.16%  runtime.mcall
         0     0% 97.84%      0.05s  2.16%  runtime.park_m
         0     0% 97.84%      0.02s  0.86%  runtime.putempty
         0     0% 97.84%      0.05s  2.16%  runtime.putfull
         0     0% 97.84%      0.02s  0.86%  runtime.schedule
         0     0% 97.84%      1.06s 45.69%  runtime.systemstack
         0     0% 97.84%      0.10s  4.31%  runtime.trygetfull
         0     0% 97.84%      1.22s 52.59%  testing.(*B).launch
         0     0% 97.84%      1.22s 52.59%  testing.(*B).runN
```

保存到 `test.txt` 中，然后通过 `pprof2csv -input=test.txt -output=test.csv`, `test.csv` 只有如下内容：

```
flat,flat%,sum%,cum,cum%,function
```

使用 `-top` flag 类似。
