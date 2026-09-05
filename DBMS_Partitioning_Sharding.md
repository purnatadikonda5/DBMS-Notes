# DBMS Architecture: Partitioning, Sharding & Distributed Systems

Welcome to the masterclass on database partitioning and sharding. This guide will walk you through how to break down massive databases to achieve infinite scale, and how to solve the distributed engineering problems that arise when you do.

## ⭐ Part 1: What is Partitioning?

Imagine you have a massive, incredibly heavy textbook. Instead of carrying the entire book just to read one specific chapter, you separate it into smaller, individual booklets. 

That is exactly what **Partitioning** is. It is the technique of dividing a large database—along with its indexes and metrics—into smaller, manageable slices of data.

> [!IMPORTANT]
> **The Single Machine Rule:** In standard partitioning, even though the data is logically split into smaller chunks, **all of those chunks still physically live on the exact same computer (server)**. You are simply organizing the hard drive better so the local CPU can process queries faster without scanning the entire giant database.
