# CTF-Frontend

# How to build a cool 3D globe in pure js?

<p>In this blog we will see how to make a cool looking globe like one shown in banner photo with different flavours and cutomizations for same</p>

## Introduction

Github is one of the most used tools in developer community, they recently did a revamp of their homepage involving a sleek looking 3d-globe, which wasn't just a photo but a complete globe rendered alongwith live information about actions like PR, merges, etc. being live previewed, racing though the rendered globe as beams, developers liked the globe concept a lot and so did I, next thing I knew I wanted to replicate it for our CTF. And the result is the globe shown in banner, we had different types of challenges shown in different colors, depicting various categories of challenges. Side menu could be used to have specific category to only get rendered. Each pole depicts a question and there are similar beams between various poles. Overall dark mode, lighting scheme and racing beams made the globe look super sleek, globe was appreciated all over the platforms, so today I am going to share how I quickly spun up an awesome globe, and also teach you how you can make your own version for the same.

## Some behind the scenes work understanding

Web-browsers have advanced a lot, and we have cool websites like https://blobmixer.14islands.com/, https://bruno-simon.com/, etc., complex animations in three dimensions have become much easier from before, thanks to libraries like three.js for being on lead for the same.

Three.js works by using WebGL renderer, it's a pretty heavy dependency due to all the heavy lifting it does behind curtains for rendering beautiful experiences. There are no restrictions with what you can make with three.js, for just a gist, three.js's official websites features an extensive gallery of interesting projects made using it. Check them out [here](https://threejs.org/ "threejs.org"). Three.js in itself is an excellent abstraction, but we have libraries which make the abstraction much wider between complex object rendering like this globe and simple APIs to customize it.

For this globe we are going to use: [react-globe.gl](https://www.npmjs.com/package/react-globe.gl), it has a very easy to use api which can render very complex globes with much ease, the first thing one notices on its home npm page is the photo of beautiful six globes, these are made with minimal config and can be found under examples in its official repo [here](https://github.com/vasturiano/react-globe.gl). One can also see a lot of examples, listed down in the README.md, its the best to go through all the examples to understand the API properly. There are predefined shapes which can be rendered on globe, or one can make a totally custom polygon using three.js APIs for rendering on the globe, a good example for the same can be found [here](https://vasturiano.github.io/react-globe.gl/example/countries-population/), with its source [here](https://github.com/vasturiano/react-globe.gl/blob/master/example/countries-population/index.html).

One can see though complex shapes are being rendered it is relatively easy to interface with APIs and create the same. For our globe we are going to stick with standard APIs for simplicity but same level of awesomeness.

## Globe Basics

Lets get started, first lets render a simple globe.

Import the library using:

```javascript
import Globe from "react-globe.gl";
```

And then render it using:

```javascript
ReactDOM.render(<Globe />, myDOMElement);
```

and thats it, you have a globe(more like a black spehere but okay) ready, suprised? Yes, thats how easy it can be with the power of abstraction. But meh, this isnt good enough, and we are still away from our final product. So lets dig into customizing it.

Firstly, we wanna give a nightly look to our globe, so lets change our background, we looked up different backgrounds and at last settled with this [one](https://raw.githubusercontent.com/mayankshah1607/Cle-Air/master/earth-planet-night.jpg). Now how to plug it in with our globe, it's as easy as making the globe itself, we have a direct prop under Globe named globeImageUrl.

Next lets change the default background of globe(i.e. our space), we will use [this](https://raw.githubusercontent.com/IEEE-VIT/CTF-Frontend/master/src/assets/night-sky.png) and the property to be changed is backgroundImageUrl.

How about an atmosphere floating with some clouds and making the globe take a single spin when loaded for the first time on users screen, this can be changed using showAtmosphere and animateIn properties, so for now our code should look something like this:

```javascript
ReactDOM.render(
  <Globe
    globeImageUrl={
      "https://raw.githubusercontent.com/mayankshah1607/Cle-Air/master/earth-planet-night.jpg"
    }
    backgroundImageUrl={
      "https://raw.githubusercontent.com/IEEE-VIT/CTF-Frontend/master/src/assets/night-sky.png"
    }
    showAtmosphere={true}
    animateIn={true}
  />,
  myDOMElement
);
```

Thats it for the basic lookover, our globe has starting to take shape. Next lets look how to get those standing pillars and racing arcs across globes.

## Pillars time

So let's erect our colorful pillars on various parts of globe, there are some things we need before starting, for a point we need pointsData, an array of points, and for each point we need a point label, a point color, its altitude, radius and a few other miscellaneous properties which we will setup later.

So first of all we need to get points data, lets make a dummy data for now, we will use this array of json objects for our example, this json object or data can be dynamic too, as was the case with our CTF questions, this example is almost identical to the dynamic content we had for CTF:

```json
[
  {
    "title": "Point A",
    "coordinates": [21.0, 0.21],
    "color": "blue"
  },
  {
    "title": "Point B",
    "coordinates": [101.0, 10.194],
    "color": "red"
  }
]
```

So we have our dummy data ready, its pretty simple to interpret, we have a title for our point, then we have our coordinates array with first entry as latitude and second one as longitude.

Time to hook them up in our code, now our code should look something like this:

```javascript
ReactDOM.render(
  <Globe
    globeImageUrl={
      "https://raw.githubusercontent.com/mayankshah1607/Cle-Air/master/earth-planet-night.jpg"
    }
    pointsData={data}
    backgroundImageUrl={
      "https://raw.githubusercontent.com/IEEE-VIT/CTF-Frontend/master/src/assets/night-sky.png"
    }
    pointLabel={(point) => point.title}
    pointLat={(point) => point.coordinates[0]}
    pointLng={(point) => point.coordinates[1]}
    pointColor={(point) => point.color}
    pointAltitude={0.2}
    pointRadius={1}
    pointsMerge={false}
    showAtmosphere={true}
    animateIn={true}
  />,
  document.getElementById("root")
);
```

Lets understand whats going on, package itself loops over various array elements and hence we get each point in our callback in pointLng, pointLat, pointColor and pointLabel, where we just access necessary properties of given objects.

Remember we talked about some extra properties? Yeah they are pointAltitude, pointRadius and pointsMerge. They are pretty simple to interpret from their name, first tells how long they should be, second talks about their radius measure and lastly wether to merge same coordinate points or not.

## Arc time

We are progressing soundly, now is the time for the racing beams between out points through globe, for that similar to points we need some processing and properties to understand, it's a bit more complicated than points, but still relatively simple. We need label for the arc, a color, dash in animate timing, and its altitude, then we need the starting and ending latitude and longitude respectively. DashAnimate property is the one due to which arcs start moving otherwise they are stationary arcs over globe, adding them converts them into cool looking racing beams.

One does not need to be specifically sticked to points for arcs it can be run from any arbitary coordinate to any other arbitary coordinate. Lets make a sample for that too:

```json
let arcsdata = [
  {
    "arcLabel": "Point A to B",
    "arcStartLat": 21.0,
    "arcStartLng": 0.21,
    "arcEndLat": 101.0,
    "arcEndLng": 10.194,
    "arcColor": "yellow",
    "arcDashAnimateTime": (Math.random() + 1) * 2000,
    "arcAltitude": Math.random() / 1.4,
  },
  {
    "arcLabel": "Somewhere to nowhere",
    "arcStartLat": 165.0,
    "arcStartLng": 0.0,
    "arcEndLat": 0.134,
    "arcEndLng": 23.254,
    "arcColor": "purple",
    "arcDashAnimateTime": (Math.random() + 1) * 2000,
    "arcAltitude": Math.random() / 1.4,
  },
];
```

Here we have included two examples, one between points, and other from somewhere to nowhere on the globe, now lets jump onto integrate them, our code should look something like this now:

```javascript
ReactDOM.render(
  <Globe
    globeImageUrl={
      "https://raw.githubusercontent.com/mayankshah1607/Cle-Air/master/earth-planet-night.jpg"
    }
    backgroundImageUrl={
      "https://raw.githubusercontent.com/IEEE-VIT/CTF-Frontend/master/src/assets/night-sky.png"
    }
    showAtmosphere={true}
    animateIn={true}
    pointsData={data}
    pointLabel={(point) => point.title}
    pointLat={(point) => point.coordinates[0]}
    pointLng={(point) => point.coordinates[1]}
    pointColor={(point) => point.color}
    pointAltitude={0.2}
    pointRadius={1}
    pointsMerge={false}
    arcsData={arcsdata}
    arcLabel={(loc) => `${loc.arcLabel}`}
    arcStartLat={(loc) => loc.arcStartLat}
    arcStartLng={(loc) => loc.arcStartLng}
    arcEndLat={(loc) => loc.arcEndLat}
    arcEndLng={(loc) => loc.arcEndLng}
    arcColor={(loc) => loc.arcColor}
    arcDashLength={1}
    arcDashGap={1}
    arcDashAnimateTime={(loc) => loc.arcDashAnimateTime}
    arcsTransitionDuration={0}
    arcAltitude={(loc) => loc.arcAltitude}
  />,
  document.getElementById("root")
);
```

And wow thats all done, it may look a bit simple now, but try it with enough data and it looks as awesome as the given globe we had in CTF, isnt that awesome? If you like the CTF looks and are curious how it was built, checkout our repo [here](https://github.com/IEEE-VIT/CTF-Frontend). A big shoutout to package's maintainer [vasturiano](https://github.com/vasturiano), if you like his and our work with CTF frontend and globe, do leave a star on both of the repos.

