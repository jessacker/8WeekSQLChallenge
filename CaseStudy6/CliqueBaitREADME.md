## Case Study #6 - Clique Bait

### Introduction
Clique Bait is not like your regular online seafood store - the founder and CEO Danny, was also a part of a digital data analytics team and wanted to expand his knowledge into the seafood industry!

In this case study - you are required to support Danny’s vision and analyse his dataset and come up with creative solutions to calculate funnel fallout rates for the Clique Bait online store.

### Available Data
For this case study there is a total of 5 datasets which you will need to combine to solve all of the questions.
* `users`
* `events`
* `event_identifier`
* `campaign_identifier`
* `page_hieracrchy`

### Entity Relationship Diagram
For this case study, the creation of an ER Diagram was the first challenge to tackle. The following diagram was created using [dbdiagram](https://dbdiagram.io/d) and the schema from the datasets.

![6CliqueBait_ERdiagram](https://user-images.githubusercontent.com/116126763/225055242-50811dd9-8edb-4404-9809-ac77c5af507e.PNG)

**Table 1:** `users`  
Customers who visit the Clique Bait website are tagged via their `cookie_id`.

**Table 2:** `events`  
Customer visits are logged in this events table at a `cookie_id` level and the `event_type` and `page_id` values can be used to join onto relevant satellite tables to obtain further information about each event.

The `sequence_number` is used to order the events within each visit.

**Table 3:** `event_identifier`  
The `event_identifier` table shows the types of events which are captured by Clique Bait’s digital data systems.

**Table 4:** `campaign_identifier`  
This table shows information for the 3 campaigns that Clique Bait has ran on their website so far in 2020.

**Table 5:** `page_hieracrchy`  
This table lists all of the pages on the Clique Bait website which are tagged and have data passing through from user interaction events.
