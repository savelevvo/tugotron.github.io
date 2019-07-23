---
layout: post
title: "RDS PostgreSQL: Logical replication issues"
comments: true
description: ""
keywords: "postgresql, aws, rds, logical, replication"
---

## RDS PostgreSQL: Logical replication issues

There are three important limitations you need to know in case you’re going to set up or already have a logical replication system on RDS. 

### Re-create

There is a tricky thing behind max_replication_slots and you might face serious problems after re-creation of the instance: 

```2019-07-05 13:16:18 UTC::@:[6165]:PANIC: could not find free replication state, increase max_replication_slots```

New subscription instance just won’t start. Here is what I found on postgres forums:

_The identifiers aren't currently dropped during node part, which should be changed. It hasn't come up to date because frequent node addition and removal is something to be avoided, and because most deployments configure room for more slots than needed to avoid future restarts._

For example: you’ve created and correctly configured publisher and subscription instances. Now, you need to re-create those instances for some reason (enable encryption or something else). But when you stop instances, they don’t free replication slots they use. And that causes a big issue because newly-created RDS instance uses default configuration in which max_replication_slots == 10. You can change its config only after it started but it can’t do so. Turns out that the new instance get stuck in a loop and at this point you have only two options: configure subscription instance from scratch or ask AWS support for help. 

### Reboot

Also, you can’t just easily restart the subscription instance because of this:

If you reboot subscriber or change instances type/class of the subscriber instance public IP of the subscriber RDS changes and the logical replication no longer works, we have to set it up again.

That means you should drop every single subscription and create it again. It will be populated from scratch then (might take a lot of time and you might reach IOPS limit depending of the publisher table size). Here is what you’ll need to do:

```sql
ALTER SUBSCRIPTION my_table_sub DISABLE;
ALTER SUBSCRIPTION my_table_sub SET (slot_name = NONE);
DROP SUBSCRIPTION my_table_sub;
delete from my_table;
create subscription my_table_sub connection 'dbname=xxx user=xxx password=xxx host=xxx port=xxx' publication my_table_pub;
```

For cases like that, it’s useful to have an automated recovery procedure that would configure the whole replication system for you.

### Change

If publisher table structure changes, replication stops:

```
2019-07-04 12:45:33 UTC::@:[22255]:LOG: logical replication apply worker for subscription "users_locationbyip_sub" has started
2019-07-04 12:45:33 UTC::@:[22255]:ERROR: logical replication target relation "public.users_locationbyip" is missing some replicated columns
2019-07-04 12:45:33 UTC::@:[15988]:LOG: worker process: logical replication worker for subscription 16739 (PID 22255) exited with exit code 1
```

That causes consuming memory (1) by huge number of WAL files that are getting collected and waiting for replication to start. 

**image**

You need to ALTER (2) subscription table to make it identical to publisher. After that, postgres will automatically sync all of the missed data.