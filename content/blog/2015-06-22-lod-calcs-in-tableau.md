+++
date = "2015-06-22"
title = "The Power of Level of Detail Calculations in Tableau"
type = "post"
tags = ["Tableau"]
+++
Just a quick one, we've been playing a lot with level of detail calculations lately in Tableau 9 and I have to say, they're _incredible_, especially when combined with context filtering. A few useful ones so far:

###Customer Lifetime Value
Presuming you have a transactions table:

```
First transaction: {FIXED [customer_id] : min([transaction_date])}
Tenure month: DATEDIFF('month',[First transaction], [transaction_date])
```


###RFM (Recency, Frequency, Lifetime Value) Modeling

```
Recency: DATEDIFF('month',{FIXED [customer_id] : max([transaction_date])})
Frequency: {FIXED [customer_id] : countd([order_id])}
Lifetime Value: {FIXED [customer_id] : sum([order_value])}
```


###Salesperson effectiveness (% index vs team average)

```
Size of Team: {FIXED [team_id] : countd([account_mgr_name])}
Team Revenues: {FIXED [team_id] : sum([order_value])}
Salesperson Index: sum([order_value])/avg([Team Revenues]/[Size of Team])
```


Those are a few examples, but these are really a game-changer with Tableau; now we can skip entire subqueries and ETL processes, massively accelerating analytics and dashboard development.
