---
layout: post
title: How We Handle Transactions across Locations
---

## Prelude

So, I've done some work for a convenience store chain that's semi-popular in the Southern US. I won't name names, but we've done fairly well and have a few dozen locations spread out across various states.
<br>
I'm gonna talk about some of the challenges we face in the locations we have and how we overcome them when it comes to aggregating our transaction data to power our reporting tools and business decisions.
<br>
## Hardware Setup

Our POS system in any given location talks to a central server in the respective location. This server pushes the prices for our items to the POS system, aggregates the transactions from the location's various POS's, as well as a host of other non-transaction related functions.

For the purpose of this post, it's just a windows machine with a Microsoft SQL Server database installed on it. (Gross. I know.)

Our machines are a bit dated. The vendor that provides our POS equipment has many new offerings (the latest of which we're in the process of migrating to), but our current setup relies on these servers which are a bit old.

You know what that means, right? They fail constantly, and some of them have limited resources by today's standards.
<br>
## Locations

On top of those problems that most of our locations are in the Southern US and some have questionable internet access at best, and we have many printed reports, homemade data visualization tools and automated alerts that are used to make important business decisions in our central office. 

Some of those tools only lag behind at most 2 minutes from any given transaction happening in any given location, and our central office is more than a day of constant driving away from some of our locations. So if any of the hardware fails in a given store, our essential tools stop working and our business leaders know very quickly.
<br>
## Transaction Programs

Each location's database only holds transactions for ~20 days. So, it is necessary to collect them if we want to store them for longer durations of time.
 
### Ant

The first program we've developed that's extremely helpful with managing this data is Ant. Ant is a golang program that sits on the server in a given location. This program uses a config file that is stored locally on-site and there is a separate task that fetches a new version of the config file every hour from an AWS S3 bucket. This pattern allows Ant to keep running on the local machine if the remote config file is unavailable, poorly formatted, or worse, if AWS is down.

The config file holds lists of queries to run against the local SQL Server database. For each of the queries the following json format is defined:

```js
[
	{
	"ID": "refundalert",
		"Driver": "mssql",
		"Src": "<Connection String>",
		"Dest": "<AWS Lambda Endpoint",
		"SQLQuery": "<Query String>",
		"UsesParameter": 1, // Boolean. 0 - Does not use local parameter table. 1 - Uses local parameter table.

		"ParamUpdateSQL": "", // Query to update local parameter table.
		"Format": {
			// Key-Value of sql column to data type
			//  to send to lambda
		},
		"MaxRows": 0, // Maximum number of rows to query
		"Frequency": "1m" // Frequency to run query
	},

	// More Queries...
]
```

*Notes:*
  - The Ant executable runs every 1 minute but not all queries need to fetch data every minute. So we list a frequency for each query.
  - Some queries return an obscene amount of data, so we may limit the number of rows to return per query.

*Parameters*
Each Ant install creates a local parameter table. This table may or may not be used in any given `SQLQuery`. Any lambda that data is sent to may return an array of key-value pairs called "Parameters". These will be stored in the local parameter table to be used in future queries.

An example of where we use parameters is for refund alerts. We will query any given refund in the local database and send the data to a special lambda to be processed.

This lambda will find any refund in the data that is greater than some dollar amount (say $10). If it finds any, it will send a text alert to the location manager (and in some cases the district manager) of a given location to notify them of this refund.

The lambda sends back the last timestamp of the latest transaction that it has processed, and And will store it in the parameters table. The next time Ant runs this query, it will only query transactions greater than this timestamp.

We use this pattern in many of our transaction queries because often the internet access at a given location might go down for hours, or in some cases weeks (we have several locations that were affected by hurricanes in the past). After any given length of time, when the location comes back online, it will always start processing at the last time it knows a lambda has received data, and will catch up over a few minutes or hours and send any transaction that occurred while it was not connected to the internet.
<br>
### Database Backups

Occasionally, the server at any given location will have some critical hardware failure. To mitigate this, we have a program that backs up the entire database for a location and send it to an S3 bucket every night. A separate service runs as part of the initial setup of any new server hardware for a location to pull and restore the latest backup of the database from that location.

<p></p>

## Final Thoughts and Future Plans

Using backups and the above process for aggregating and doing work on transaction data, we lose very little if any data about any given transaction that occurs in the face of hardware failures or internet outages at our locations, and most of our business-critical applications built on top of Ant show actionable data in near-realtime to everyone in our organization from our location managers to the CEO.

We have currently put in place programs to aggregate raw transaction data to an S3 bucket using similar methods in a project we call the Transaction Journal in an effort to never loose a piece of any given transaction data. The data will be uploaded and have various asynchronous processes run against it within a minute of any given transaction occurring. A beta version is already pushed out to all locations and collecting a lot of data. We've also included configurable limitations on the number of transactions to process and send across the internet to preserve some of the precious bandwidth and system resources at some locations.
