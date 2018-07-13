# Hackathon Graph Recommendations

How we built a graph model for product recommendations in two days.

![](https://s3-us-west-2.amazonaws.com/knowledge-repo/2018-07-05/Hackathon-Graph-Recommendations/b58cda1c0a65bb3c0c633a070be268cc)  
  
Author: Seth Dimick  
The Team: Jim Liu, Aaron Flower, Ola Hungerford, Thomas Fritchman, Fan Tu

## The Idea

Using a [property graph model](https://neo4j.com/developer/guide-build-a-recommendation-engine/) to surface relevant content to users is now common practice for many digital experiences, from social media to retail. While no graph technology solution has been implemented at Nordstrom for personalization (*as of today*), a one-step Markov chain, or transition matrix, was implemented in October 2017 to provide shoppers with a personalized sort of Homepage content on the mobile web experience and [yielded significant conversion lift](https://confluence.nordstrom.net/display/ACE/6-22+NORDACE-1443+Homepage+Redesign+-+MOW+Results). While the transition matrix implementation proved successful in one instance, the approach is incredibly hard to scale or iterate upon using relational data structures.

To expand upon this proven success our [Summer 2018 Hackathon](https://confluence.nordstrom.net/display/HACK/Summer+2018+Hackathon) team, __*Graphathon*__, built a __[neo4j](https://neo4j.com/)__ graph database with our website clicksteam data. The MVP graph model included product view and purchase data connected by shopper interactions for adult men's shoes. Our demonstrated use case for the graph associates similar shoppers via live clickstream data in order to present __*personalized product recommendations*__ of various strategies.

## The Graph

For our project, we designed a graph that leveraged both the well-documented property graph method and the proven Markov chain method from the Homepage. The result is a schema with three types of nodes and three types of relationships.  
  
<p align="center"><b>Basic Schema</b></p>

<p align="center">![](https://s3-us-west-2.amazonaws.com/knowledge-repo/2018-07-05/Hackathon-Graph-Recommendations/ec57944c9dc032ed6796d5c08a386fde)</p> 
  
__Product nodes__ (*green*) represent all adult male shoe Style IDs viewed in the past 30 days. The Product nodes are shared by the property graph of shopper views and purchases and the Markov chain of shoppers' next product view transitions.  
  
The property graph functionality is enabled by relating Product nodes to the __ShopperSession nodes__ (*red*). ShopperSession nodes represent browsing sessions determined by our existing clickstream logic, and they are related to Product nodes by __VIEWED__ and __PURCHASED__ relationships, or edges.  
  
A Markov chain model was then created by relating Product nodes to each other with an intermediary __NextView node__ (*purple*). The NextView nodes have shopper and session information and connect to Product nodes with __NEXT__ edges. Counting relevant NextView nodes provides the "probabilities" of a shopper transitioning from one product view to another. (*The intermediary NextView nodes are not theoretically necessary, as shopper and session information could be attributed to the NEXT edges, but the nodes were added to the schema to increase query performance when filtering the Markov chain by shopper attributes and counting the number of relationships.*)  
  
<p align="center"><b>Schema with Data</b></p>

<p align="center">![](https://s3-us-west-2.amazonaws.com/knowledge-repo/2018-07-05/Hackathon-Graph-Recommendations/0749e57fcb316326e875d9f33b9ea9d0)</p>

## The Data

Our graph model for the hackathon was derived from two tables of snowplow clickstream data in the [Customer Analytics Redshift](https://confluence.nordstrom.net/display/TDS/CA-+Redshift+%3A+Data+Sets+Documentation) database: `clk_strm_sp.sp_product_views` and `clk_strm_sp.sp_order_items`. To transform these relational tables into a directional graph, the Product Views table is joined to itself, by shopper and session, to create a relationship between each of shoppers' page views and next sequential page view for all adult men's shoes. The data was transformed and saved to s3 with some simple [ETL code](https://gitlab.nordstrom.com/hack/Summer2018/graphathon/tree/master/etl), and then loaded into neo4j by the cypher (*graph query language*) [load script](https://gitlab.nordstrom.com/hack/Summer2018/graphathon/blob/master/cypher/dataload.cypher).  
  
The data source for our recommendation queries is also notable. If taken to production, graph recommendations will be the first consumer of Personalization's __*real-time*__ serverless streaming data pipeline that was built to populate the recently viewed products tray. Our graph-based recommendation strategies (*cypher queries*) are parameterized by this real-time data to provide personalized recommendations based on shoppers' __*in-session clickstream events*__ leading up to and including the current product page view.

## The Strategies

In the short time of the hackathon we developed three recommendation strategies based on the graph model. The first strategy, __*Previous to Next*__, is a Markov-centric approach that considers two transitions, or three sequential product views. The second two strategies were designed as competitors for the existing strategies __*People Also Viewed*__ and __*People Also Bought*__.

### Previous Product to Next Product

The __*Previous to Next*__ strategy was designed to leverage information about the shopper's in-session journey in addition to information about the current product view.  
  
![](https://s3-us-west-2.amazonaws.com/knowledge-repo/2018-07-05/Hackathon-Graph-Recommendations/86647e8aaf7c4ba9103d596921b68dba)  
  
In the above graph visualization of the strategy, the Product node (*green*) in the center represents a current product being viewed. The Product node on the left is the product viewed immediately before the current product. A portion of the NextView nodes (*purple*) between the previous and current Product nodes represent the shoppers who followed the same transition as our current shopper. The remaining NextView nodes in the graph show us where those same shoppers went for their next product view immediately following the current product view (which includes the possibility of going back to the previous product). These NextView nodes can be traced to all the Product nodes around the perimeter and counted for each associated Style ID, which gives a sorted list of recommended products to view next through the cypher query below.  
  
```cypher
MATCH (prev_product:Product {styleid:$style2})
MATCH (curr_product:Product {styleid:$style1})
MATCH (prev_product)-->(user_nv1:NextView)-->(curr_product)-->(user_nv2:NextView)-->(rec:Product)
  WHERE user_nv1.shopper_id = user_nv2.shopper_id
RETURN rec.styleid as styleid, count(user_nv2) AS count
  ORDER BY count DESC
  LIMIT 10;
```

### People Also [ Bought | Viewed ]

The __*People Also Viewed*__ and __*People Also Bought*__ strategies were designed to compete with existing strategies while utilizing the advantages of the graph model.  
  
![](https://s3-us-west-2.amazonaws.com/knowledge-repo/2018-07-05/Hackathon-Graph-Recommendations/f7c95dcc62d17566c9d3b91897008e79)  
  
Similar to the previous strategy's graph representation, in the above graph the central Product node (*green*) represents the current product a shopper is viewing. Unlike the previous strategy, we don't consider what previous product this shopper is coming from, but only consider all shoppers who have viewed the same product and return which products they viewed next, represented by the surrounding NextView (*purple*) and Product nodes. From the products viewed next, we then attain a measurement of popularity to sort our recommended products by counting the number of ShopperSession nodes (*red*) connected to our Product nodes with a PURCHASED or VIEWED edge. The simple cypher query below accomplishes this logic.   
  
```cypher
MATCH (curr_product:Product {styleid:$style1})-[:NEXT]->(rec:Product)
MATCH (rec)<-[:PURCHASED]-(sessions:ShopperSession)
RETURN rec.styleid, count(sessions) as buyers
  ORDER BY buyers DESC
  LIMIT 10;
```

## The Implementation

### Architecture

During the hackathon we were able to implement our new recommendations strategies utilizing the existing development environments for Recbot (Personalization's recommendation service) and Product Page. After populating the graph, we added added functionality to Recbot that calls the neo4j database with our three strategy queries. The recommendations are triggered by the existing call from Product Page to Recbot with the Shopper ID and the current product's Style ID. Recbot uses the Shopper ID to query it's streaming data for the shopper's recently viewed products' Style IDs, and uses the recently viewed Style IDs along with the current Style ID from the Product Page as parameters for the queries. Recbot then returns the recommended styles from the neo4j response along with offer service data in the same format as it's existing recommendation strategies to the Product Page for rendering.  
  
![](https://s3-us-west-2.amazonaws.com/knowledge-repo/2018-07-05/Hackathon-Graph-Recommendations/759857aa225fa7f17f646d6ce0ec1884)  
  
### Development Experience

The new recommendations were surfaced in the Product Page development environment through the tried and true recommendation treatments. Below you can see a rendered adult male shoe page with the new __*Previous to Next*__ recommendation strategy on the right-hand side of the page.  
  
![](https://s3-us-west-2.amazonaws.com/knowledge-repo/2018-07-05/Hackathon-Graph-Recommendations/6df32e54c23eeb27b9ff62fb4fd594bd)

### Outcomes

While our project did not manage to take the podium in the hackathon judging, we managed to build a near production ready, customer facing product on an entirely new data structure in two days. Through the process we found that working with graph provides incredible agility to iterate on both the query strategies and the graph schema itself and that graph models can provide incredibly personalized experiences for shoppers which can optimized for various outcomes. I was glad to hear Ola and Personalization plan to build off our work to test graph strategies in production post-anniversary.  
  
![](https://s3-us-west-2.amazonaws.com/knowledge-repo/2018-07-05/Hackathon-Graph-Recommendations/fc6067e30f11026ee9fb8f7d9c504542)  
  
Working with graph visualizations and the cypher query language also sparked countless ideas on how to leverage our existing data while working on our hack, and I look forward to utilizing graph technology across many use cases with many data sets in the future.  
  
![](https://s3-us-west-2.amazonaws.com/knowledge-repo/2018-07-05/Hackathon-Graph-Recommendations/6aa92d9a3d14552c87f367808c7d224d)
