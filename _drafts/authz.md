---
layout: post
title: "The problem with authorisation"
date: 2024-06-15 00:00:00 +0000
categories: auth
published: true
---

Let's say we're building a backend service for GitHub. The service receives calls over HTTP, and based on the action, it checks required permissions, and decides whether to accept or reject the call.

To demonstrate the problem, let's assume we have a typical CRUD application, and in no means I pretend I know how GitHub is implemented, the example is mainly to demonstrate the problem with the following endpoints:

- `POST /repository` creates an repository content and returns its ID
- `GET /repository/{ID}` returns a repository file and folder structure

Each of these endpoints would have a different authorisation action behind them. Let's take a look at them individually.

### Create a repository record

In a create scenario, the backend application creates a record in the domain database, and updates the authorisation service to associate the record ID with the user ID.

![Create Record](/assets/post240615/create-record.jpg)

It is up to the application to make sure the data store supports transactions, and the write to the authorisation service is part of the atomic operation. Meaning a failure to write to the authorisation service should rollback the write to the database. The application needs to handle atomicity failures correctly.

In this example, we use two-phase commit approach. However, an architecture could use [CDC](https://en.wikipedia.org/wiki/Change_data_capture){:target="_blank"} to capture write events to the domain database, and write tuple relations in the authorisation service. Or, in a [CQRS](https://martinfowler.com/bliki/CQRS.html){:target="_blank"}, the application could send authorisation commands that is then written to the authorisation service. Regardless of the approach, the problem with this article is demonstrating remains the same.

### Reading a repository record

To read a single repository record, the application first checks permissions first by asking the authorisation service whether a given user is on the list of users who can perform an action for a given record.

![Read Single Record](/assets/post240615/read-single-record.jpg)

Again, it is up to the application to make sure it understands and conforms to the response from the authorisation service. In the first call, Alice wanted to read record with ID 1, and the authorisation service allowed that. If the application has a bug in consuming the authorisation decision, there is nothing preventing it from rejecting the call. The same is true in the second example where Alice requests record ID 2. The application could query the database and return the result to the user who has been denied access as far as the authorisation service is concerned.

### Is this secure?

AuthZed has done an amazing job implementing Google Zanzibar-inspired, SpiceDB. I have used it professionally, and solved many of the complex scenarios that we have abstracted from our database. In AuthZed landing page, they demo a spaceship requesting a permission to land on a planet. One of the spaceship is denied landing, and the other one was granted.

![Ornithopter check permission](/assets/post240615/authzed-can-land-authzedia.jpg)

The question whether this design is secure is dependant on the the domain of the problem we are trying to solve. If we are working on a highly regulated product, and keeping the permission checks on the application, and yet it can decide to abide by it or not is definitely insecure.

### What are we guarding?

We are using an authorisation service to keep data secure and accessible to only who have permissions t o read it. In other words, we are guarding against unauthorised access to records. So why are we not putting the boundaries around our data? If our demo GitHub backend application is doing the authorisation checks, and based on the decision it requests the database, the guards are only surrounding the backend application, not the data.

If we decided to create another backend application that has access to the same database, but without the authorisation checks, or with buggy authorisation checks, that means the data is not guarded.

Some databases have Row-Level Security feature built into them. These RLS are the most secure way of protecting against unauthorised access. But they lack the advanced relational style of Zanzibar-inspired products.