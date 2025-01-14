---
title: "[Excerpt] Dynamodb Intro"
date: 2025-01-14T16:24:44+08:00 
lastmod: 2025-01-14T16:24:44+08:00 
categories: 
- AWS
tags:
- AWS
- AWS CDK
summary: "Initial note for DynamoDB"
description: "Initial note for DynamoDB" 
weight: 
slug: ""
draft: false 
hidemeta: false
cover:
    image: "" 
    caption: ""
    alt: ""
    relative: false
---

# Reference

[The What, Why, and When of Single-Table Design with DynamoDB](https://www.alexdebrie.com/posts/dynamodb-single-table/?source=post_page-----c734d8445ae2--------------------------------)

# Excerpt
## Overview
DynamoDB provides many benefits that other databases don't:
- a flexible pricing model
- a stateless connection model that works seamlessly with serverless compute (guaranteed by AWS!)
- consistent response time even as your database scales to enormous size

## Single-table design
### Background on SQL modeling & joins
一般用關連式資料庫時，我們會設計多張表已進行正規化，例如電商服務會有消費者和訂單的關係。

Each Order belongs to a certain Customer, and you use foreign keys to refer from a record in one table to a record in another. These foreign keys act as pointers -- if I need more information about a Customer that placed a particular Order, I can follow the foreign key reference to retrieve items about the Customer.

The SQL language for querying relational databases use joins to follow the pointers. Joins allow you to combine records from two or more tables at read-time.

### The problem of missing joins in DynamoDB
While convenient, SQL joins are also expensive. They require scanning large portions of multiple tables in your relational database, comparing different values, and returning a result set. Also, huge query in codebase is not so maintainable. That's why I usually call APIs for SQL operation.

DynamoDB was built for enormous, high-velocity use cases, such as the Amazon.com shopping cart. These use cases can't tolerate the inconsistency and slowing performance of joins as a dataset scales.

Rather than working to make joins scale better, DynamoDB sidesteps the problem by removing the ability to use joins at all.

In our example above, we want to get both a Customer record and all Orders for the customer. Many developers apply relational design patterns with DynamoDB even though they don't have the relational tools like the join operation. This means they put their items into different tables according to their type. However, since there are no joins in DynamoDB, they'll need to make multiple, serial requests to fetch both the Orders and the Customer record.

This can become a big issue in your application. Network I/O is likely the slowest part of your application, and you're making multiple network requests in a waterfall fashion, where one request provides data that is used for subsequent requests. As your application scales, this pattern gets slower and slower.

### The solution: pre-join your data into item collections
An item collection in DynamoDB refers to all the items in a table or index that share a partition key. In the example below, we have a DynamoDB table that contains actors and the movies in which they have played. The primary key is a composite primary key where the partition key is the actor's name and the sort key is the movie name.  

Note that in AWS, the price of table with just partition key and composite primary key is the same, so we generally setup sort key when creating a table.  
 
{{< figure  src="pre-join.png" title="sol" width="70%" align="center">}}

Use DynamoDB's Query API operation to read multiple items with the same partition key. Thus, if you need to retrieve multiple heterogenous items in a single request, you organize those items so that they are in the same item collection.  

In sum, the single-table design is all about -- tuning your table so that your access patterns can be handled with as few requests to DynamoDB as possible, ideally one. The benefit would be the performance improvement by making a single request to retrive all needed items.

## Downsides of a single-table design
Three main downside of single-table design:
- The steep learning curve to understand single-table design
- The inflexibility of adding new access patterns
- The difficulty of exporting your tables for analytics

### The steep learning curve to understand single-table design
A single, over-loaded DynamoDB table looks really weird compared to the clean, normalized tables of your relational database. It's hard to unlearn all the lessons you've learned over years of relational data modeling.

### The inflexibility of adding new access patterns
However, your table design is narrowly tailored for the exact purpose for which it has been designed. If your access patterns change because you're adding new objects or accessing multiple objects in different ways, you may need to do an ETL process (Extract, transform, load), something like database migration, to scan every item in your table and update with new attributes. This process isn't impossible, but it does add friction to your development process.

### The difficulty of exporting your tables for analytics
DynamoDB is designed for OLTP (Online transaction processing) use cases -- high speed, high velocity data access where you're operating on a few records at a time. But users also have a need for OLAP (Online analysis processing) access patterns -- big, analytical queries over the entire dataset to find popular items, or number of orders by day, or other insights.

DynamoDB is not good at OLAP queries. This is intentional for achieving high performance!!! DynamoDB focuses on being ultra-performant at OLTP queries and wants you to use other, purpose-built databases for OLAP. To do this, you'll need to get your data from DynamoDB into another system.

Forrest Brazeal's excellent walkthrough on single-table design:
```
[A] well-optimized single-table DynamoDB layout looks more like machine code than a simple spreadsheet
```
## When not to use single-table design
The concrete answer is "whenever I need query flexibility and/or easier analytics more than I need blazing fast performance." And I think there are two occasions where this is most likely:

- In new applications where developer agility is more important than application performance: The flexibility outweighs the performance.
- In applications using GraphQL: Because of the way GraphQL's execution works, you're losing most of the benefits of a single-table design while still inheriting all of the costs.
