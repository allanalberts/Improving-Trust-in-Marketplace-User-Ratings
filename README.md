# Fighting-Fraud-with-Graph-Theory
## Background / Purpose
The purpose of this project is to examine strategies for detecting ratings manipulation that occurs when a fraudster sets up fake accounts in an online marketplace and then uses them to boost their peer to peer ratings score. Specifically, I will demonstrate how effective fraud detection features can be built using graph theory and applied in a machine learning model to predict fraud.

## The Data 
I will be using five years of peer to peer ratings from the Over The Counter (OTC) Bitcoin Marketplace [Over The Counter (OTC) Bitcoin Marketplace!](https://bitcoin-otc.com) between 2011 and 2016.

![](/images/OTC_screenshot.png)

In this online marketplace, where users trade products and services for bitcoin, each party has the option of rating the trustworthiness of the person they transact with. Trust ratings range from -10 representing completely untrustworthy to +10 reflecting a very trusted individual. For the purpose of this analysis, I have categorized these ratings as fraud (-10), untrustworhy (-9 to -1), and trusted (>0). The [dataset](https://snap.stanford.edu/data/soc-sign-bitcoin-otc.html) contains four fields: date, rater, ratee, rating.

## The Fraud Ratings Manipulation Scheme

Fraud is an issue in the OTC marketplace and their websites displays the following warning: 

*"Do not rely on the ratings blindly - since the cost of entry into the web of trust is only one positive rating, it is not impossible for a scammer to infiltrate the system, and then create a bunch of bogus accounts who all inter-rate each other. Talk to people on #bitcoin-otc first, make sure they are familiar with the person you're about to trade with, have traded with him successfully in the past, etc."*

The goal of this project will be to identify these types of fraudulent OTC transactions before they occur. 

In peer to peer (P2P) marketplaces, fraudsters generally need to establish a trusted rating profile in order to convince a victim to engage in a transaction of any significant value. Therefore, the most aggregious frauds will involve a user account with a positive ratings profile and no history of negative ratings. I will test fraud detection techniques on a subset of the marketplace transactions that meet this criteria:
- 3+ prior positive ratings
- no history of negative ratings

Total Ratings in subset: 17,965 
- Negative Ratings: 525 (2%)
- Fraud Ratings: 245 (1%)

## Using Graph Theory to Detect Ratings Manipulation
Graph theory metrics can be used to understand the interactions that a user is having with other accounts and can be used to identify patterns associated with fraudulent ratings boosting. When OTC users rate each other, a network is formed linking them together by the ratings they give each other. I display positive ratings in blue and negative ratings in red. The direction shows who is receiving the rating and width is related to the size of the rating score. 

<img src="/images/example2.png" alt="drawing" width="300"/>

The network surrounding a fraudster who is manipulating his ratings is structured different from the activity that is generated from a normal user. Interactions between users can be quantified and then compared with normal OTC interations to identify anomalies indicative of fraudulently boosted ratings. Graph theory components for this type of analyis can be grouped into **Triads** and **Measures of Centrality**

### Traids
In graph theory, triads are groups of 3 nodes that are interlinked in some way. Triads in our case would be users that are linked by the ratings that they give each other. In a directional graph there are 16 different possible configurations of triads. 

<img src="/images/triads.png" alt="drawing" width="700"/>

Triads use the following nameing convention:
    1st digit: number of bidirectional connections between nodes (users having both rated each other)
    2nd digit: number of single directional connections (only one user has been rated)
    3rd digit: number of open connections (users have not interacted with each other)
    
The triads most interesting for us are the 201 and 030T structures. Prevalence of the **201 traid** is what we would normally expect to see when **legitimate users** interact and both mutually rate each other. 

<img src="/images/201_example.png"  width="200" heigh="300" />

The **030T structure** is often seen when fraudsters are using **fake accounts to boost ratings**. In this scenaro, they don't necessarily provide reciprocal ratings and the users that have been rated tend to rate each other in an interconnected fashion. 

<img src="/images/030T_example.png"  width="300" heigh="300" /> <img src="/images/example_4733.png" width="500" height="300"/>

To normalize the triad metrics for high and low volume users, I divide the number of triads by the number of neighbors (other users) associated with the account. As you can see in the example above, as the user receives additional fake ratings, their 030T metric also rises.

### Measures of Centrality
The **Cluster Coefficient** measures the proportion of raters that also rate each other. Fake users that rate each other will also produce a dense network which can manifest itself with a **low Betweeness value and a high Closeness value**. Betweeness represents the degree to which nodes stand between each other, so if a node acted as a bridge between communities if would have high betweeness. Closenss represents how close the node is to all of the other nodes in the network. A high value indicates that the node would appear visually towards the center of a graph. 

### Feature Creation
To create fraud prediction features, I use the **NetworkX** machine learning library of functions along with the history of prior positive ratings in the marketplace to generate a **Reverse Directed Ego Subgraph** of each users rating connections. An Ego Subgraph is a subnetwork that is centered around the user. I use a reverse directed graph with a radius of 1 so that I pick up the users that have rated the user in question. The ego network will also pick up interactions between any of the nodes in the subgraph. Feature sets are generated for both the rater and ratee of each rating.

### Graph Based Features:
- number of neighbors
- 210_triad / neighbors
- 120_triad / neighbors
- 300_triad / neighbors
- 030T_triad / neighbors
- 201_triad / neighbors
- 111_triad / neighbors
- 102_triad / neighbors
- 021_triad / neighbors
- betweeness
- closeness
- cluster coefficient

### Non Graph Based Features:
In the OTC marketplace, fraudsters are impatient. They rate each other over a very short period of time, possible using automation bots, and they give each other unusually high ratings. Features based on these two traits (days active and avg rating) are included in my model:
- days since last rated
- days since first rated
- average rating
- total ratings

### Features for Differentiating Legitimate Rating Interactions
To predict negative OTC ratings, we need to do more than just identify a user involved in ratings manipulation. We must also identify when they are being rated by one of their co-conspiritor users (positive rating) vs. a legit user who is being victimized (negative rating). I do this by measuring the difference in metric values between the rater and ratee.

### Classifier Model
My fraud detection model utilizes 41 features with a **Random Forest Classifier**. I trained the model on 80 percent of the marketplace ratings using a stratified and shuffled sample and tested performance against the remaining 20% of ratings. I used scikit-learn's RandomizedSearchCV method to define a grid of parameter ranges and randomly sampled from the grid performing 3-fold cross validation. Next I used a grid search with cross validation for final tuning. Tuning resulted in a 29% increase in model performance. The accuracy score of 0.9880 was extemely close to the Out Of Bag score of 0.09877 thus suggesting model validity. My model was able to successfully predict a negative rating 49% of the time with 2% of legitimate users affected with a false positive prediction when using a 0.1 threshold value. 

<img src="/images/results.png" alt="drawing"/>

### Measuring Model Performance
The model predicted 36% of the fraud using only graph theory features. When additional features based on timing and level of rating where added, the performance increased to 49%. However, fraudsters adapt and will change their modus oprandi once their actitity is recognized. The advantage of graph theory is that it is relatively easy for fraudsters to change the length time between ratings and also the rating value given, however, it is much harder for them to manipulate the features developed through graph theory and circumvent detection.  
