---
layout: blogpost
title: Introducing Hitchspots.me
subhead: Get Hichwiki spots into MAPS.me
categories: hitchhiking code
date: 2017-10-15 10:00:00 +0300
imgclass: vw
---

I like to go hitchhiking in Europe with my friends. We get together, draw pairs, and race towards a famous spot in a European city. The first team who arrives there by hitchhiking alone is declared winner 🏆. It's very fun, you can check our track record on our [awesome website](https://www.somewherexpress.com/). 😬

We organized one of these trips last September (2017), and a few weeks before, I was starting to think about it.

## Hitchwiki is Awesome

[Hitchwiki](http://hitchwiki.org/en/Main_Page) is the ultimate resource for hitchhikers: there you can find valuable resources about anything hitchhiking related: country and city informations, tips, places to avoid, etc. 

There is also a [map](http://hitchwiki.org/maps/) with crowd sourced, rated hitchhiking spots everywhere in the world. It's extremely helpful, for example in order to know where to start a new trip, to tell a driver where to drop you, or to find the closest good spot if you get dropped in an unexpected place.

![Hitchwiki Spots Map](/img/posts/hitchwiki_map.jpg)

But there is one problem for me with this map: I usually don't have an Internet connection when I'm hitchhiking. Yes, the European Union has [ended roaming charges](http://europa.eu/rapid/press-release_MEMO-17-885_en.htm) but unfortunately, neither my home country Switzerland, nor my residence country Turkey are part of this deal.

Because of that, I cannot access the very helpful Hitchwiki spots whenever I leave Turkey or Switzerland without paying way too much money. That's a bummer. 😞

## MAPS.me is awesome

I use the [MAPS.me](https://maps.me/download) application in my everyday life and I love it. It allows you to download maps and navigate without an Internet connection. I also use it when I go Hitchhiking.

MAPS.me has this Bookmarks feature where you can pin a location and write some personal comments about it. I realized earlier this year that those Bookmarks can in fact be exported, but more importantly imported from a KML file.

And that's when it clicked: I should find a way to get Hitchwiki spots as a MAPS.me Bookmarks collection, that I can import while I have an Internet connection, before I go on a trip. 

## Hitchspots.me brings them together

SPOILER: I made it and it's dead simple to use!

1. Go to [Hitchspots.me](http://www.hitchspots.me/) with your phone (which has MAPS.me installed already)
2. Enter your trip departure and arrival locations and click the Generate button
3. It will download the KML file, open it
4. That's it 🎉

<p style="text-align: center"><img src="/img/posts/hitchspots_me_demo.gif" alt="Hitchspots.me demo" /></p>

It will get all the spots that are approximately 10km on each sides of your route. Bookmarks are colored according to their Hitchability rating (from Green — Very Good, to Red — Very Bad).

If you click on a Bookmark, you will get the spot detail: city name, description by the author, average waiting time, and comments.

Oh, and by the way, if you are unsure of your route, you can also download all the spots for an entire country.

![Hitchspots example](/img/posts/hitchspots_examples.jpg)
<div class="legend">Left: spot detail in Bookmark, Right: all the spots of a country</div>

## Under the hood

<div class="disclaimer">That's the part where it gets technical, so if you are not into coding, you should probably stop reading here, and go enjoy the rest of your day! Happy Hitchhiking! 👍🚗</div>

Let me walk you through my process to create Hitchspots.me. To begin with, all the code is [Open Source](https://github.com/norydev/hitchspots). Hithhiking is all about giving and receiving, so I wanted my code to be Open Source, and use Open data in it.

Ruby is my favorite programming language, so I reached out to it, and started a plain old Ruby project.

### What it needed to do

First, Hitchwiki maps has an API from which we can retrieve spots data. Without it, it would not really have been possible to build this app. I was going to hit it a lot, so I asked [Mikael](https://twitter.com/simison) if it was ok, and he has been very supportive. Shout out to him and the wonderful Hitchwiki community. 🙌

Second, as I said earlier, MAPS.me has a Bookmark import functionality. So data gathered from Hitchwiki can be transformed to fit a MAPS.me file, which is then importable.

Third, we need to develop the part that figures out which spots to add to a file. My main interest is to gather the spots for a defined trip from point A to point B <sup id="sup-1"><a href="#footnotes-1">1</a></sup>.

### Getting the spots from Hitchwiki

Hitchwiki's API <sup id="sup-2"><a href="#footnotes-2">2</a></sup> has basically two modes: Get a collection of spots, or get the detail of one spot. The payload for a collection of spots is like this:

```json
[{"id":"355","lat":"51.2029594207479","lon":"4.38469022512436","rating":"2"},
 {"id":"182","lat":"49.8697605358452","lon":"8.6283928155899","rating":"2"},
 {"id":"362","lat":"48.4196173897275","lon":"10.6816077232361","rating":"4"}]
```

In the end, I would ideally like my Bookmarks to contain more information than that, so I will need the payload from a spot detail. But for the first iteration, let's start with this minimal data. It is enough to pin a Bookmark on a map and color it according to it's rating.

Hitchwiki's API has several modes for retrieving collections; by city, continent, country or coordinates-bounded area. As my main goal is to get the spots for a trip from point A to point B, my only option would be to use the "by area" endpoint.

### Calculating the areas for a trip

After a bit of research, I found [OSRM](http://project-osrm.org/), which allows to calculate routes between two points. It's open source, and so far, I've been pretty satisfied by the results it gave me. 

When I started working on Hitchspots.me, it was possible to hit the API described in their [documentation](http://project-osrm.org/docs/v5.10.0/api/#general-options) but it seems no longer the case <sup id="sup-3"><a href="#footnotes-3">3</a></sup> (renders a 503 for all requests).

[MabBox](https://www.mapbox.com/) is using the OSRM material and even paying for some of it's development ([allegedly](https://github.com/NoryDev/hitchspots/pull/2#issuecomment-324653518)) so I'm happy to use it to calculate routings.

OSRM results are pretty good, because they give enough intermediate points of coordinates, but not too many either.

![OSRM results example](/img/posts/osrm_example.jpg)

From this route, we can calculate a collection of areas around these points, in order to query the Hitchwiki API "spots by area" endpoint. To do so, I found out adding 0.1 degree (or about 10km) on each side of the point is good enough to cover all the useful spots.

![Zones calculation](/img/posts/zones_explained.jpg)

For a 500km trip, it will mean about 25-40 zones. That means, 25-40 hits of the Hitchwiki API. That can be a long time to wait for the user. From the Hitchwiki maps website, we can know that there are about 25'000 spots worldwide. That's not so many. For a first iteration, we could get them all in a JSON file and query the file instead of the real API. We can do that with one hit of the "by area" endpoint passing -90, 90, -180 and 180 as bounds.

### Geolocating departure and arrival locations

OSRM expects lat/lon coordinates for departure and arrival. But I want users to give me a city name, or an address, not a latitude and longitude.

Open Street Map provides the [Nominatim](http://wiki.openstreetmap.org/wiki/Nominatim) tool. Given a string, it will return the details of the n first places it found, ordered by accuracy. The detail about the place includes it's latitude and longitude.

```json
{
  "place_id": "114173",
  "licence": "Data © OpenStreetMap contributors, ODbL 1.0. http://www.openstreetmap.org/copyright",
  "lat": "50.938361",
  "lon": "6.959974",
  "display_name": "Cologne, District de Cologne, Rhénanie-du-Nord-Westphalie, Allemagne",
  "class": "place",
  "type": "city",
  "importance": 0.82233722043841,
  // more data
}
```

Now a user can enter two locations, and we can pipe our different components to get all the Hitchwiki spots for the route between these two locations:

1. Geolocate Departure and Arrival locations with Nominatim
2. Pass these coordinates to MapBox to get a route (list of coordinates)
3. Compute areas around this route
4. Query the Hitchwiki API (or it's data stored in a JSON file) for each computed area to get a collection of spots

What remains is to render them as a MAPS.me Bookmarks collection file.

### Rendering the spots in a MAPS.me Bookmarks file

This [article](https://support.maps.me/hc/en-us/articles/207895029-How-can-I-import-Bookmarks-) explains how to import a Bookmarks collection. We can see that MAPS.me accepts both KML and KMZ (zipped KML files). We will go with KML. 

To figure out the MAPS.me format, I exported one of my personal boomarks collection, and it looks like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<kml xmlns="http://earth.google.com/KML/2.2">
<Document>
  <Style id="placemark-blue">
    <IconStyle>
      <Icon>
        <href>http://mapswith.me/placemarks/placemark-blue.png</href>
      </Icon>
    </IconStyle>
  </Style>
  <!-- More placemark Styles -->
  <name>My places</name>
  <visibility>1</visibility>
  <Placemark>
    <name>A great place I went to</name>
    <description>Contact number
+1-555-33-44-55</description>
    <TimeStamp><when>2017-05-17T13:41:44Z</when></TimeStamp>
    <styleUrl>#placemark-red</styleUrl>
    <Point><coordinates>6.5652005,46.4674717</coordinates></Point>
    <ExtendedData xmlns:mwm="http://mapswith.me">
      <mwm:scale>14</mwm:scale>
    </ExtendedData>
  </Placemark>
  <!-- More Placemarks -->
</Document>
</kml>
```

For each spot obtained from the Hitchwiki API, we can create a `Placemark` with coordinates, and `styleURL` containing a different color tag base on the rating.

### Making the web application

If we pipe all these components together, we have a working first iteration. It is not perfect: if spots are updated on Hitchwiki, we won't get the updated version (as we are still using a JSON file with the data instead of the real API), and we only get limited data for a spot (it's location and rating).

But that's a working alpha version, so I integrated this code in a new [Sinatra](http://sinatrarb.com/) application, made a simple form interface for the user to enter locations and [published](https://twitter.com/norydev/status/899204874054893568) it.

## Second iteration

There are a few necessary improvements from here. First, the user doesn't get any feedback if their locations are real places. Second, it would have more value if the Bookmark description contained more data for the spot.

### Places autocomplete and real time geolocation

Implementing places autocomplete was easier that I thought: I used [this plugin](https://github.com/komoot/typeahead-address-photon) that is doing exactly what I wanted and gets the same Open Street Map data that I used in the first iteration.

Except it does it in the browser and in real time. These coordinates can be sent with the form data directly and the backend doesn't need to call an external API for geolocation anymore.

I kept the backend code as a backup in case the javascript plugin fails for some reasons.

### Add more data to the Bookmark description

The Hitchwiki API provides an endpoint to get the detail of a spot, given it's ID (which we get from the "by area" endpoint). There is no endpoint to retrieve more than one spot detail at a time. A 500km trip in central Europe can quickly mean 100 to 200 spots, so it is not an acceptable solution to hit the Hitchwiki API that many times for a single trip.

As we have seen, the total number of spots is not that high (about 25'000), so with the permission of Mikael from Hitchwiki (thanks again), I decided to harvest the entire dataset from Hitchwiki, store it locally, and then query from this local storage. I would also have a CRON job to refresh the data from Hitchwiki once a day.

I opted for a MongoDB storage, for several reasons:

1. I preferred saving the Hitchwiki API payload in a NoSQL storage, as it may change (there are, in fact, already plans to change it).
2. I could have used a PostgreSQL jsonb column for the raw payload. But it turns out, for my kind of data, a cloud hosting of a MongoDB instance was cheaper.
3. I had no experience with MongoDB except for a few tutorials, so it was the opportunity to learn about it.

### Sanitizing the data

With this done, I quickly added some of this new data to my KML template: description, city name, comments, etc.

But it didn't work. The file would be generated, and MAPS.me would open it, but there would be no Bookmark in it. 🤔

It turns out, the data from Hitchwiki, at least some of it, is not utf-8 encoded and it looks like this breaks MAPS.me. After a few trial and errors, I figured the corrupt data was Windows-1252 encoded. So I passed all of the spots through a sanitizer and now it worked.

## Testing it out

As I said in the introduction, all of this was triggered by the fact that we were going to go on a Hitchhiking trip with my friends. I finished the second iteration of Hitchspots.me just a few days before the trip, ready to be used.

How did it go? Well [pretty good](https://twitter.com/norydev/status/910821946979438593)! 😄🏁🍾🎉

---

[<span id="footnotes-1">1</span>] <a href="#sup-1">↑</a> If you have followed, there is also a feature to get all the spots for a country. I added it after the hitchhiking trip with my friends, for which I needed the "spots on a route" feature. It was, by the way painless to add, so I guess I can be proud of my architecture. 💪

[<span id="footnotes-2">2</span>] <a href="#sup-2">↑</a> There is no url to the documentation, you need to go to the [maps](http://hitchwiki.org/maps/) site and click API in the bottom right corner. 

[<span id="footnotes-3">3</span>] <a href="#sup-3">↑</a> It seems it is coming and going, sometimes it's available sometimes not. I decided to stick with MapBox for now.
