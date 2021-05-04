# Retro Gaming in Docker

Its been nearly 30 years since the height of the DOS era, and how far we have come! Computers have advanced to the point that we can emulate entire systems from the era, down to the processor, in our _web browsers_. All while also checking email, or watching a YouTube video. However, with all of these advancements, our old software has a tendency to fall by the wayside. Particularly games, which often relied on incompatible tricks to eke out every ounce of performance from the machines they ran on.

Many projects have sprung up in the wake of these advancements to help preserve this heritage. Projects like [DOSBox]() provide a compatible environment for old games. Additionally, projects like the [Internet Archive]() provide a fantastic library of games. Through the magic of modern web development, Internet Archive is even able to provide a playable experience in your browser. In my experience, they play pretty well!

However, I still miss those less-connected days of the early 90's. I used to ride my bike to the store and pick up a shareware copy of some game for $1. I then had a physical disk, with the game on it, ready to install on my (or a friends) computer. The self-contained nature of it all was somewhat magical. Nowadays, many _web pages_ are bigger than the shareware games I used to buy.

This got me thinking: "Floppies can be imaged. That sounds kind of like a Docker image. I wonder if I can make DOS shareware Docker images?"

Turns out, you can do this by leveraging much of the same technology that the Internet Archive uses for their playable experience!

## What You Will Need

* [JSDOS 6.22]()
* A copy of a Shareware game
* [NodeJS]()
* [Docker]()
* [Earthly]()

## Putting It Together

First, we will need to acquire JSDOS. You can get the latest versions of the files [here](). Download and place these files into your projects directory. Here is how I am doing it in Earthly:

```Dockerfile
jsdos:
    FROM node:16-alpine

    WORKDIR site
    RUN wget https://js-dos.com/6.22/current/js-dos.js && \
        wget https://js-dos.com/6.22/current/wdosbox.js && \
        wget https://js-dos.com/6.22/current/wdosbox.wasm.js
    RUN npm install -g serve    # Another dependency, here for caching
```

JSDOS is nothing without a game to play in it. For our example purposes, we will use one of my aboslute favorites: [Secret Agent](). A couple things to check when you are looking for games:

* Make sure that the game you choose is _actually_ shareware. [Don't copy that floppy!]()
* Make sure that it is the actual game files, not just the installer.
    * While you _can_ install and _then_ run the game via DOSBox, you'll have to install it on every visit to the webpage. The installation is also fairly slow.
* Make sure that it is a `zip` file. This is what JSDOS wants to load for us.

Heres how I add the game using Earthly:

```Dockerfile
game:
    FROM +jsdos

    ARG GAME_URL
    
    RUN wget -O game.zip $GAME_URL
```

At this point, we have a Docker image with JSDOS, and a Zip file containing our beloved game. Now we just need a way to play it! To this end, we need a webpage which loads JSDOS, adds our game, and provides a canvas for JSDOS to render its output to. Here is my webpage:

```html
<html>
  <style type="text/css" media="screen">
      canvas {
          width: 1280px; 
          height: 640px;
      }
  </style>
  <head>
    <title>DOS Game!</title>
    <script src="js-dos.js"></script>
  </head>
  <body>
    <canvas id="jsdos" width="640" height="320" ></canvas>
    <script>
      Dos(document.getElementById("jsdos"), {
      }).ready((fs, main) => {
        fs.extract("game.zip").then(() => {
          main(["-c", GAME_ARGS])
        });
      });
    </script>
  </body>
</html>
```

! Alert re: name, GAME_ARGS

Copy this into the same directory as your JSDOS files and your game. Now, all we need to do is start a server within our Docker container to serve this webpage. For this, I used (`serve`)[], because it was quick and easy to script (you may have noticed intalling this dependency alongside JSDOS). Heres how I add the server to the container:

```Dockerfile
web:
    FROM +jsdos

    ARG GAME_ARGS

    COPY index.html .
    RUN sed -i s/GAME_ARGS/$GAME_ARGS/ index.html

    ENTRYPOINT npx serve -l tcp://0.0.0.0:8000
```

And now we have a shareable Docker container, with playable shareware! Our example also contains an Earthly target that launches the game for you, and save the image at the same time.

```Dockerfile
play:
    LOCALLY

    ARG GAME_TAG

    WITH DOCKER --load jsdos:$GAME_TAG=+web
        RUN docker inspect jsdos:$GAME_TAG > /dev/null && \ #Using side-effect to save image locally too
            docker run --rm -p 127.0.0.1:8000:8000 jsdos:$GAME_TAG
    END
```

Additionally, we have a couple pre-made targets that wrap this for you.