---
layout: post
title: Website Refresh
---

It's that time of year again where I change the website design and move it to
a new system. This time round it's a bootstrap based design using jekyll and
github pages.

There's support for code

```c
void
tn_task_run_single(void)
{
    bs_sem_wait(&internal.num_queue_full);
    int index = atomic_fetch_sub(&internal.queue_index, 1);
    tn_task_handle handle = internal.queue[index];
    bs_sem_post(&internal.num_queue_free, 1);

    run_task(handle);
}
```

and MathJax

\begin{aligned}
\dot{x} & = \sigma(y-x) \\
\dot{y} & = \rho x - y - xz \\
\dot{z} & = -\beta z + xy
\end{aligned}

It's all very exciting stuff, no doubt this will lead to countless deeply
informative and insightful posts.
