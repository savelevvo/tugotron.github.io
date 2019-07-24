---
layout: post
title: "PostgreSQL: Important logical replication issues on Amazon RDS"
comments: true
description: "There are three important limitations you need to know in case you’re going to set up or already have a logical replication system on RDS. They are related to re-creation, reboot and change of the subscription or publisher instances and might lead to awful consequences if you are not aware of them."
keywords: "postgresql, aws, rds, logical, replication, amazon"
---

There are three important limitations you need to know in case you’re going to set up or already have a logical replication system on RDS. They are related to re-creation, reboot and change of the subscription or publisher instances and might lead to awful consequences if you are not aware of them.

<br>

### Re-create

There is a tricky thing behind `max_replication_slots` and you might face serious problems after re-creation of the instance: 

```2019-07-05 13:16:18 UTC::@:[6165]:PANIC: could not find free replication state, increase max_replication_slots```

New subscription instance just won’t start. Here is what I [found](https://www.postgresql.org/message-id/CAMsr%2BYGA%3Drb2VD7FPR8gXSBhTVYwmXZHM8TR3qtq2Lkpq%2BCkrg%40mail.gmail.com) on postgresql forums:

> The identifiers aren't currently dropped during node part, which should be changed. It hasn't come up to date because frequent node addition and removal is something to be avoided, and because most deployments configure room for more slots than needed to avoid future restarts.

For example: you’ve created and correctly configured publisher and subscription instances. Now, you need to re-create those instances for some reason (enable [encryption](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.Encryption.html) or something else). But when you stop instances, they don’t free replication slots they use. And that causes a big issue because newly-created RDS instance uses default configuration in which `max_replication_slots == 10`. You can change its config only after it started but it can’t do so. Turns out that the new instance get stuck in a loop and at this point you have only two options: configure subscription instance from scratch or ask AWS support for help. 

<br>

### Reboot

Also, you can’t just easily restart the subscription instance because of [this](https://forums.aws.amazon.com/thread.jspa?messageID=848707):

> If you reboot subscriber or change instances type/class of the subscriber instance public IP of the subscriber RDS changes and the logical replication no longer works, we have to set it up again.

That means you should drop every single subscription and create it again. It will be populated from scratch then (might take a lot of time and you might reach IOPS limit depending of the publisher table size). Here is what you’ll need to do:

<script src="https://gist.github.com/tugotron/71050d8679fc9c8be9b411807264eebf.js"></script>

For cases like that, it’s useful to have an automated recovery procedure that would configure the whole replication system for you.

<br>

### Change

If publisher table structure changes, [replication stops](https://www.postgresql.org/docs/10/logical-replication-restrictions.html):

```
2019-07-04 12:45:33 UTC::@:[22255]:LOG: logical replication apply worker for subscription "users_locationbyip_sub" has started
2019-07-04 12:45:33 UTC::@:[22255]:ERROR: logical replication target relation "public.users_locationbyip" is missing some replicated columns
2019-07-04 12:45:33 UTC::@:[15988]:LOG: worker process: logical replication worker for subscription 16739 (PID 22255) exited with exit code 1
```

That causes consuming memory (1) by huge number of WAL files that are getting collected and waiting for replication to start. 

![](/assets/images/TshCbKaiW4.png)

You need to ALTER (2) subscription table to make it identical to publisher. After that, postgres will automatically sync all of the missed data.