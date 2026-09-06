# DBMS Architecture: Partitioning, Sharding & Distributed Systems

Welcome to the masterclass on database partitioning and sharding. This guide will walk you through how to break down massive databases to achieve infinite scale, and how to solve the distributed engineering problems that arise when you do.

## ⭐ Part 1: What is Partitioning?

Imagine you have a massive, incredibly heavy textbook. Instead of carrying the entire book just to read one specific chapter, you separate it into smaller, individual booklets. 

That is exactly what **Partitioning** is. It is the technique of dividing a large database—along with its indexes and metrics—into smaller, manageable slices of data.

> [!IMPORTANT]
> **The Single Machine Rule:** In standard partitioning, even though the data is logically split into smaller chunks, **all of those chunks still physically live on the exact same computer (server)**. You are simply organizing the hard drive better so the local CPU can process queries faster without scanning the entire giant database.

### When Do We Apply Partitioning?
You don't need to partition every database. We introduce this technique under two specific conditions:
1. **The Dataset is Too Huge:** The sheer volume of data makes backups, indexing, and basic management too tedious and slow.
2. **The Traffic is Too Heavy:** The number of requests hitting the database is so large that a single server's CPU queues up, causing the system's response time to spike.

### The Advantages of Partitioning
* **Performance & Parallelism:** Multiple read/write operations can happen simultaneously across different partitions.
* **Availability:** If one partition is corrupted or goes down, the rest of the database remains accessible.
* **Manageability:** Smaller chunks of data are vastly easier to backup, restore, and maintain.
* **Cost Reduction:** Scaling up a single, massive supercomputer (vertical scaling) is astronomically expensive. Partitioning allows you to use cheaper, standard servers.

## ⭐ Part 2: Vertical vs Horizontal Partitioning

Depending on how your data is accessed, there are two primary ways to slice a database table.

### 1. Vertical Partitioning (Slicing by Columns)
This involves slicing the data relation vertically. 
* Imagine a `Users` table with 50 columns. You might put the `id`, `name`, and `email` on Partition A (accessed frequently for login), and the `bio`, `profile_picture`, and `address` on Partition B (accessed rarely).
* **The Catch:** Because the columns of a single record are split up, the application must join data from different partitions to put a complete row back together.

### 2. Horizontal Partitioning (Slicing by Rows)
This involves slicing the data relation horizontally. 
* In this method, independent chunks of *complete* data rows (tuples) are stored in different partitions. For example, Users A-M go into Partition A, and Users N-Z go into Partition B.
* The structure of the table remains identical across all partitions; only the raw rows are divided.
