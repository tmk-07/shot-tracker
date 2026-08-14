# Shot Tracker

A mobile tool to log shots in a basketball shooting session. Visualize efficiency from each zone and identify weak spots.

## Access Site

[https://shooters.tkimify.com/](https://shooters.tkimify.com/)

## Features

* Tap an exact location on an interactive half court, or switch into persistent free-throw mode
* Keep logging from the selected spot until a new location is chosen
* Undo attempts and persist the current session locally
* Review total and distance-based shooting percentages after ending a session
* Review a jittered make/miss attempt map and a Bucks-style zone-efficiency map
* Download any chart or a shareable summary as a PNG

## Tech Stack

* HTML and CSS
* SVG and Canvas
* JavaScript
* Browser `localStorage` for saving shot data on the user's device

Deployment:

* Deployed using Cloudflare Pages
* Static frontend hosted directly from the project files
* No backend server or database required
