---
title: Hack your way to your GF's heart
date: 2026-05-04
---

# Hack your way to your GF's heart

At my girlfriend’s workplace, there was a Chrome extension that displayed a daily mantra alongside beautiful landscape images, along with some to-do list features and the date & time. She really liked it and used it often, but for some reason I didn’t connect with it. Some of the mantras felt dramatic to me—for example, phrases like “Let it go” or “Time heals any pain, and you will be out of this soon.”, and I want phrases like "You look gorgeous today".

Hmm...

What if I can control these mantras? \
What if I can display my own mantras/quotes? \
**What if I can hack my way to my girlfriend's heart?**

## I - Hack the extension?

### Getting The Source Code

I need to analyze the code first, so there should be a way to get the source code of the extension by just using another Chrome extension that exports the installed mantra extension into a `.crx` file. From there I can use VSCode to read the source code. Easy.

### Now what...

After analyzing and reading the code, I need a way to make the time in the center of the page clickable, and on a click it should get a quote/mantra that is published from my server through an API call.

### It is not actually hacking

I discovered that it took me a lot of time to reverse engineer the obfuscated JavaScript code and add an API call to my server, while it was relatively easy for me to add the onClick behaviour for the time widget.

Because I could manage to set the onClick behaviour, what if I just create a new `my_script.js` file in the root directory of the extension and then declare a method in it to call my API and make the `onClick` method call it, and upon receiving an API response just `document.getElementById("time").innerHTML = "<api response which is my mantra>"`. Magnificent. 
In addition to that, I want to make the UI more alerting when a new mantra is published, so I decided to add a hook to the extension so that whenever a new tab is opened, an API call is made to my server to check if there is a new mantra. If yes, then the time starts flickering (I did that by sliding back and forth the time HTML element's CSS property `opacity` between 100% and 50%).

### Let's test it on my Chrome browser

It's very easy to do this: just activate developer mode in the extension section and click `load extension`. This will add the extension as if it is a newer version to Chrome.

### The backend and the API

Back then I needed something really quick that just runs, so I developed an ExpressJs app and deployed it to Vercel. I designed two GET endpoints for my service and another one POST.

* GET `/mantra`: Fetches the published mantra.
* GET `/check`: Checks if there is a published mantra.
* POST `/add/<my-mantra-to-publish>`: Stores the mantra in DB.

The flow is not complicated. I wake up every day and publish a sweet mantra using the POST endpoint, and it gets stored in a PostgreSQL database table called - you guessed it - `mantra`. 
The table looks like this:

```SQL
CREATE TABLE IF NOT EXISTS public.mantra
(
    id SERIAL PRIMARY KEY, 
    text character varying(512), -- the actual mantra text
    seen_time character varying(128), -- when was it seen for the first time
    seen_who character varying(10), -- seen by who (will explain this below)
    set_on character varying(128) -- when was it published by me?
);
```

Then my girlfriend logs in to her computer and opens Chrome, the trigger calls the `/check` endpoint and it returns true, the time element starts flickering. She clicks the mantra and an API call to `/mantra` endpoint is made, and the time HTML element changes its text to the mantra content. A cute smile on my girlfriend's face. DONE.

### The `seen_who` and `seen_time`

I can't let the API be publicly accessible because the content is private, obviously, so I came up with a simple trick: when installing the extension on Chrome for the first time, a code will trigger to generate 6 random numbers, e.g., `123456`. Then I store the number MANUALLY (I should have provided CRUD APIs for this, but who cares! She is my girlfriend so we are the only ones who are using this) in a table called `whitelist` that looks like this:

```SQL
create table whitelist (
    pin int primary key not null
); 
```

So whatever API you are calling, you need to specify an HTTP header `pin`. I know that's not really secure and you still can brute force it, I just didn't want to deal with all the crazy auth frameworks. And I wrote a middleware that checks the header:

```Javascript
app.use(async (req, res, next) => {
  let pin = req.headers["pin"]
  if(pin == undefined) {res.send("UNAUTHORIZED"); return}
  if(!(await checkPin(pin))) {res.send("UNAUTHORIZED"); return}

  next()
})
```

where `checkPin()` is:

```Javascript
async function checkPin(pin){
  let list = await db.whitelist();
  for(let i=0; i<=list.length-1; i++){
    if(list[i].pin == pin) return true;
  }
  return false;
}
```

Now that my APIs are private, I can store who has seen the mantra in `seen_who` in the `mantra` table when hitting `/mantra` endpoint, and if `seen_time` is NULL it means the latest published mantra is never seen yet, thus it needs to be updated to the current timestamp.

The `/mantra` API here is always fetching the latest unseen mantra to return it, the one that has the latest `set_on` and `seen_time == NULL`.

This whole thing worked for almost a year... a year of me publishing mantras every single workday. She always liked it and waited for it EVERY DAY, that's why I kept publishing. Then my girlfriend left that job...

## II - Pivoting to a Mobile App

Because my girlfriend left the job for that company, she lost all access to the mantra platform and the Chrome extension along with her work PC, of course. One thing was always there: her love for these mantras I published. She adores it!

And as any wise boyfriend would do, I decided to "officialize" the platform. I mean here by giving up the Chrome extension and its hacky way altogether and actually think about something more robust, clear, easy to use, and right in her pocket...

A mobile app! Yes! Basically the same APIs that were called by the Chrome extension will remain the same but with a few changes. At that time I was learning Flutter and did a few applications for fun, so I decided to do it in Flutter.

### New technical requirements

The app basically should provide:

* A secure way to access the app
* Should work only when she is at work, just like she used to use it at work through the extension
* A clear UI

#### Secure app

This requirement is for privacy more than "security". It basically provides a lock on the whole app by leveraging the `local_auth` Flutter package.

#### Geolocation confirmation

The application will use the `location` package that manages everything related to detecting locations.
Of course I know her new job headquarters, so creating a map circle with a radius of 0.5 km would be more than enough to allow her to see the mantra I published.

```dart
int pin = await sh.getPin();
final response = await http.post(
  Uri.parse('${api}/mantra'),
  headers: {"Content-Type": "application/json",
  "pin": pin.toString()},
  body: jsonEncode({
    "lat": lat,
    "long": long
  })
);

var data = jsonDecode(response.body.toString());
```

If you noticed, this code snippet is now used to call the mantra API from the mobile app, but with a new body and operation POST. The body contains two points: latitude and longitude, or you can consider them as the X,Y of your position on Earth. So the backend code will be:

```javascript
function arePointsNear(checkPoint, centerPoint, km) {
    var ky = 40000 / 360; // km per degree latitude
    var kx = Math.cos(Math.PI * centerPoint.lat / 180.0) * ky; // km per degree longitude at given latitude

    // Calculate the differences in degrees and convert to kilometers
    var dx = Math.abs(centerPoint.lng - checkPoint.lng) * kx; // Difference in longitudes
    var dy = Math.abs(centerPoint.lat - checkPoint.lat) * ky; // Difference in latitudes

    // Calculate the Euclidean distance and compare with the given distance in kilometers
    return Math.sqrt(dx * dx + dy * dy) <= km;
}

function isNearToPlace(lat, long) {
    
    var pointA = { lat: lat, lng: long };
    var pointB = geo;
    var distance = 0.5; // in km
    if(geo.disabled) return true;
    
    let v = arePointsNear(pointA, pointB, distance);
    console.log("lat: " + lat + ", long: " + long + " is close " + v)
    return v
}
```

We implemented the geolocation check successfully! Now what...

## III - At the end of the day she likes it and that's what matters

Technically, this project is the longest running project I have made for more than two years now. 
I really enjoyed programming and developing this project. But seeing her smile every day made me proud of what I did. 
As you can see, everything I have implemented here was relatively easy, so I encourage you too to build something for her or in another way... **Hack your way to your girlfriend's heart.**

Thanks for reading.
