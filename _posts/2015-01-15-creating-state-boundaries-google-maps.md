---
layout: post
header-img: img/default-blog-pic.jpg
author: PGarg
description: 
post_id: 3527
created: 2010/05/24 10:31:25
created_gmt: 2010/05/24 05:31:25
comment_status: open
image: /assets/article_images/2014-07-06-oracle/desktop.jpg

---

# Creating state boundaries on google maps.

Google maps api provides feature to draw polygon on the map for a given sets of polygon points [developer google maps for more info](https://developers.google.com/maps/documentation/javascript/examples/polygon-simple). A polygon point is a geographical point represented by latitude and longitude. This feature can be leveraged to draw the state boundaries by drawing polygon in such a fashion that it consists of polygon points which are nothing but set of lat/lon coordinates representing that state boundary.

Taking specific case for a particular Indian state, If we search for Delhi in google maps then delhi state boundary is highlighted in light red color. If we want something similar in application highlighting an area or maybe completely fill the area inside that boundary, we can't do it by using some out of the box feature provided by google maps (Maybe in future).

To draw the polygon first we need the polygon points that make up the boundary of delhi, Now the problem is from where to get this set of coordinates.

There are few website that provide this data for some cost but there are free options 	also.

* One option is get this data from [Open Street Maps](http://www.openstreetmap.org/). Search for delhi and inspect the network calls in the browser. One of call is going to return the polygon point in xml format. Convert this data in appropriate format to be consumable by google maps using a script/program.

The problem with this data is that the polygon points retured as a reponse are not in a sequential order to draw the polygon in clockwise or anticlock wise starting from a particular point, Which causes google maps api to draw some wierd diagram. There is a small program to fix it in the github repo link provided below.

* Second option is to utilize the [Rest Services](http://wikimapia.org/api) provided by wikimapia to get these coordinates. The useful service is #placegetbyid. This service requires an area id (in our case id of the state). 

* Most easy option is to use data provided by this [DynoGeometry](http://www.dyngeometry.com/web/WorldRegion.aspx)  We can copy this data and then do some cleanup/formatting so that its consumable by google maps. This is how it should look like finally (Only a snippet).

```
[{"lon":77.58635,"lat":18.27177}, {"lon":77.57572,"lat":18.22574}, {"lon":77.57125,"lat":18.16786}, {"lon":77.61761,"lat":18.0048}, {"lon":77.66366,"lat":17.97081}, {"lon":77.6026,"lat":17.86826}, {"lon":77.50312,"lat":17.80861}, {"lon":77.47424,"lat":17.7017}, {"lon":77.46527,"lat":17.60376}, {"lon":77.57575,"lat":17.5477}, {"lon":77.66482,"lat":17.47081}, {"lon":77.60968,"lat":17.45284}, {"lon":77.55151,"lat":17.42597}, {"lon":77.47812,"lat":17.3486}, {"lon":77.43534,"lat":17.29506}, {"lon":77.38955,"lat":17.20887}, {"lon":77.38976,"lat":17.11836}, {"lon":77.42966,"lat":17.09326}, {"lon":77.45603,"lat":16.95684}, {"lon":77.44851,"lat":16.89747}, {"lon":77.45475,"lat":16.84704}, {"lon":77.47634,"lat":16.7937}]

```

So once we have the coordinate, we can package them as we like. One way is to keep them in individual .txt file. Now provide this data to the maps to plot it. This code is fetching data for server for a list of states, creating a polygon and adding it on the map. 

```javascript
var states = {"delhi"};

for ( var state in states) {
		var coords = [];
		$.ajax({
			url : "Data/states/" + state.toLowerCase(), 
			success : function(data) {
				$.each(data, function(c, s) {
					coords.push(new google.maps.LatLng(s.lat, s.lon));
				});
			},
			async : false,
			type : "GET",
			dataType : 'json'
		});

		polygon = new google.maps.Polygon({
			paths : coords,
			strokeColor : "white",
			strokeOpacity : 1.5,
			strokeWeight : 1,
			fillOpacity : 1.0
		});
	
		polygon.setMap(map);

	}
```

A sample application is created to plot crime agaist women from 2001-2012 in India. The boundaries on india states have been created using above mentioned approach.
The crime data has been taken from [gov.data.in](http://www.data.gov.in/keywords/crime-against-women). Here is a visual from the application.
	

![India State Boundaries](/assets/article_images/boundaries.png)

The complete code can be found at the [Github repository](https://github.com/PratikGarg/visuals-crime-india). This repo also contains the coordinates for [All indian states](https://github.com/PratikGarg/visuals-crime-india/tree/master/src/main/webapp/Data). 