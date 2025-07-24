---
title: Consistent Hashing
date: 2025-07-24 12:00:00 +600
categories: [technical]
tags: [java, cache, system design]
---

## What is Consistent Hashing?

Consistent Hashing is a specialized hashing algorithm designed to minimize data reorganization when you add or remove nodes in a distributed system.

Why Do We Need It?
Imagine you have a distributed cache with multiple servers (nodes) storing data. What happens when you want to add a new server to handle more traffic, or if one of your existing servers fails and needs to be removed?

With traditional hashing algorithms, adding or removing a node usually means:

Updating the Hashing Logic: The entire hashing function often needs to be reconfigured to account for the new number of nodes.

Massive Data Remapping: A significant portion of your data might need to be moved from existing nodes to new ones, or redistributed among the remaining nodes.

This remapping process can be incredibly inefficient, causing system slowdowns and increasing network traffic. This is precisely where consistent hashing provides a much more elegant solution.

### How Consistent Hashing Works
Consistent hashing visualizes the entire hash range as a ring or circle (from a minimum to a maximum hash value). Here's how it works:

Node Placement: First, each node in your system is hashed and placed at a specific point on this ring.

Data Placement: When you want to store a piece of data (or retrieve it), its key is also hashed and placed on the same ring.

Clockwise Rule: To determine which node stores the data, you move clockwise around the ring from the data's hash position until you encounter the first node. That's where the data is stored or retrieved from.

What Happens When a Node is Added or Removed?
This ring-based approach offers significant advantages:

Adding a Node: When a new node is added, it's simply placed on the ring. Only the data that was previously mapped to the node immediately clockwise to the new node needs to be remapped to the new node. The vast majority of other data remains untouched on its original nodes.

Removing a Node: If a node is removed, the data it contained is simply remapped to the next node clockwise on the ring. Again, only a small portion of data is affected.

This minimal remapping makes consistent hashing incredibly efficient for dynamic distributed systems like distributed caches, where nodes are frequently added or removed.

The consistent hashing approach, while powerful, can sometimes lead to an uneven distribution of data if the physical nodes are not well-scattered around the hash ring. Imagine if your hashing algorithm happened to place all your nodes very close to each other on one side of the circle; in such a scenario, the first node encountered clockwise from most data points would end up handling a disproportionately large amount of traffic. This creates a hotspot and defeats the purpose of distributed load balancing.

To mitigate this problem, we introduce the concept of virtual nodes (also known as "vnodes"). Instead of placing only the physical nodes on the ring, we create multiple virtual representations of each physical node and distribute these virtual nodes across the ring. Each virtual node then points back to its actual physical node.

By adding a significant number of virtual nodes (e.g., 100-200 virtual nodes per physical node), we achieve a much smoother and more uniform distribution of both nodes and data on the ring. This greatly increases the likelihood that data will be evenly spread across all physical servers, preventing any single node from becoming overloaded.
