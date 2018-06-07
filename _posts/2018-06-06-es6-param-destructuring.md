---
layout: post
title: "Use ES6 destructuring with function parameters"
---

# [](#javascript-es6-destructuring)javascript, es6, destructuring

One of the features of es6 that I've had a hard time adopting is destructuring.  I have been trying to incorporate it into my code more and one use case that I couldn't find much info on was using it for function parameters.  Sometimes, especially, when calling apis you can get a giant blob of json back but, most likely, you are only interested in a few properties.  This is where destructuring can come in real handy!

### [](#fun-example)Fun Example

Lets use an api's return but parse out only the data elements we care about using destructuring.

I'm going to use [rickandmortyapi](https://rickandmortyapi.com/documentation#get-a-single-character) because it requires no key and has CORS enabled.  Don't worry if you don't know what that means.  I'm also going to use the native [fetch api](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) sorry IE users.

> Requirements
>
> Display name, origin, image, and first episode morty shows up in rick and morty.

First lets take a look at what the api returns when fetching morty info morty has a character id of 2.

```js
fetch('https://rickandmortyapi.com/api/character/2')
  .then(response => response.json())
  .then(json => console.log(json));

  //{id: 2, name: "Morty Smith", status: "Alive", species: "Human", type: "",
  //image:"https://rickandmortyapi.com/api/character/avatar/2.jpeg",location:{name: "Earth (Replacement Dimension)", url: "https://rickandmortyapi.com/api/location/20"},
  //name:"Morty Smith",origin:{name: "Earth (C-137)", url: "https://rickandmortyapi.com/api/location/1"} ...}
```

If we take a look at the json that is returned there are a bunch of properties that we don't really care about since all we want is name, origin name, image, and the first item in the episodes array.  Here is where destructuring can really help.

Let's write a little code to display this json in a div without destructuring.

```js
fetch('https://rickandmortyapi.com/api/character/2')
  .then(response => response.json())
  .then(char => displayCharacterInfo(char));

function displayCharacterInfo(char) {
  const appDiv = document.getElementById('app');
  appDiv.innerHTML = `
  <h1>${char.name}</h1>
  <p>Origin: ${char.origin.name}</p>
  <p>First Episode: ${char.episode[0].split('/').pop()}
  <div>
    <img src="${char.image}"/>
  </div>
  `;
}
```

This looks fine and will work but there are a lot of nested properties.  So lets re-write this but use destructuring!!

```js
fetch('https://rickandmortyapi.com/api/character/2')
  .then(response => response.json())
  .then(char => displayCharacterInfo(char));

function displayCharacterInfo({name, image, origin: {name: origin}, episode: {[0]: episode}}) {
  const appDiv = document.getElementById('app');
  appDiv.innerHTML = `
  <h1>${name}</h1>
  <p>Origin: ${origin}</p>
  <p>First Episode: ${episode.split('/').pop()}
  <div>
    <img src="${image}"/>
  </div>
  `;
}
```

Since we only care about those 4 properties we can destructure that large json object into the properties we care about.  Destructuring can also be used to rename properties which you can see is how I change the origin.name property to just be called origin in the code.  This cleans up our code and eliminates us diving into the json object which in my opinion makes the code a little more readable.

Another es6 feature we can take advantage of is default params.  What would happen if the api returned null data?  Well currently the app would throw an error.

```js
function displayCharacterInfo({ name, image, origin: { name: origin }, episode: { [0]: episode } } = { name: 'Unknown', image: '', origin: { name: 'Unknown' }, episode: '0' }) {
  const appDiv = document.getElementById('app');
  appDiv.innerHTML = `
  <h1>${name}</h1>
  <p>Origin: ${origin}</p>
  <p>First Episode: ${episode.split('/').pop()}
  <div>
    <img src="${image}"/>
  </div>
  `;
}
```

Now our app will not throw an error if the api starts returning no information and this function will fall back to use the provided defaults.

Code for this example can be found at [stackblitz](https://stackblitz.com/edit/parameter-destructuring)

While destructuring is very powerful I do think if you aren't familiar with it at first glance it can be slightly confusing.  I showed some deeply destructured code to a co-worker and they weren't quite sure what was going on.  Now after I explained it to them they saw the light but keep that in mind when using destructuring!

Happy Coding.
