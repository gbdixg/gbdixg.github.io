---
id: 17
title: '''Potential SQL performance issue caused by change tracking retention period'''
date: 2019-02-23-2T19:28:11+00:00
author: Geoff
description: Resolving the SCCM CB update warning 'Potential SQL performance issue caused by change tracking retention period'
layout: post
permalink: /2019/02/23/sql-change-tracking-retention
image: assets/images/thumb-sql-change-tracking-retention.png
comments: true
featured: false
hidden: false
categories: [ SCCM, SQL, Level200 ]
tags:
  - sccm
  - sql
  - Level200
---
A warning was added when upgrading to SCCM current branch 1810 or later that I saw for the first time today - `Potential SQL performance issue caused by change tracking retention period.`
This post covers the steps to investigate and resolve the warning.

#### Running the pre-requisite check

SCCM Current Branch updates are initiated in the `Administration > Updates and Servicing` node of the CM console. The `pre-requisiste check` option will highlight any errors and warnings before the upgrade starts. In this case:

```cmd
The prerequisite check passed with 1 warning
```

#### Identifying the warning

Switch to the `Monitoring` tab of the CM console and click on `Updates and Servicing Status`. The most recent warning will be displayed at the top of the right pane.
Double-click on the warning to see more information.

![Warning](/assets/images/sql-change-tracking-retention1.png){:class="img-responsive"}

The warning description provided the following additional information:

```cmd
The site database has a backlog of SQL change tracking data. For more information, see https://go.microsoft.com/fwlink/?linkid=2027576
```

#### Applying the fix

The Microsoft article in the above description provides SQL commands that must be run over a `diagnostic or administrator connection`.

###### Enable SQL remote admin connection

If you try to create an Administrator connection to the SQL server using SQL Management Studio (SMSS), you will receive an error that SMSS can't be used. This is misleading.

>SMSS can be used to create a dedicated administrator connection to SQL, using a 'database engine query'

Open SMSS and connect to the SQL server or instance hosting the CM database when prompted.
Start a new SMSS query window and enable remote admin connections using the following commands:

```SQL
EXEC sp_configure 'remote admin connections', 1;
 GO
 RECONFIGURE
 GO
 ```

Disconnect the SMSS connection so that an Admin connection can be made:

* Click on the server/instance in the Object Explorer pane
* File > Disconnect

###### Make an admin connection to the SQL database

It isn't possible to make an admin connection using the SQL Object Explorer. The Object explorer tries to make multiple connections whereas an Admin Connection requires a single connection. Instead, click on the option for a `database engine query:`

![DatabaseEngineQuery](/assets/images/sql-change-tracking-retention2.png){:class="img-responsive"}

When prompted for the connection information, prefix the server or sql instance with `admin:`

![DatabaseEngineQuery](/assets/images/sql-change-tracking-retention3.png){:class="img-responsive"}

###### Clean-up the change tracking retention

Once connected, follow the steps in the [Microsoft KB article above](https://docs.microsoft.com/en-us/sccm/core/servers/deploy/install/list-of-prerequisite-checks#bkmk_changetracking) i.e. run the following commands:

```SQL
USE <ConfigMgr database name>
EXEC spDiagChangeTracking
```

If `CT_Days_Old` is greater than 7 in the output, run the following command:

```SQL
EXEC spDiagChangeTracking @CleanupChangeTracking = 1
```

Monitor the clean-up as follows:

```SQL
SELECT * FROM vLogs WHERE ProcedureName = 'spDiagChangeTracking'
```

When the fix is complete, switch back to the Configuration Manager Console, go to the Administration tab, click on Updates and Servicing and re-run the pre-requisite checker to ensure there are no other warnings.
