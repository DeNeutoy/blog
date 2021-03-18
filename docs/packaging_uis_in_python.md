
# Including Javascript in Python Packages
Recently, i've become much more interested in interactive machine learning, where humans have input into the training, prediction
or evaluation of machine learning models.


## Distributing ML Visualisations
I've been working on several projects surrounding visualisation of embeddings of neural network models for text and annotation tools. As part of this, one major barrier is how to distribute these tools to people that need to use them. For most machine learning tools, the answer to this question is relatively obvious. It *has* to be a pip-installable Python package. Python has dominated as the language for interfacing with powerful machine learning tools and data science tools more generally. It's not going to go away any time soon.


### The standard way - Client Server Architectures
Unfortunately, this poses a bit of a problem for interactive machine learning tools, because to build complex user interfaces with lots of interactivity (buttons, draggable selections, 3D visualisations), it's mostly necessary (or at least, standard) to use Javascript/Typescript. Typically, this means that in development and production, tool developers replicate the client-server model in which two services communicate with each other, one running the frontend code and one running the backend code.

This is great (and actually preferrable) if you want your application to run as a website. But this is a very clunky interface to a command line tool, because it requires a _user_ knowing how to orchestrate the front and back end of your application, which can be quite complicated - involving Docker, proxies and complicated software engineering stuff that you can't expect research/data scientists to necessarily know about.

So the real question is - how do we package a full javascript frontend application *inside* a Python package?

*Note - this concept certainly does exist already (notably in SpaCy's [Prodigy annotation tool](https://prodi.gy/), which is distributed in exactly this way), it's just not very common, and requires some cross-over knowledge to get working properly.*


### The various pieces we need
To achieve this we need 4 things:

- A pip installable python package with a CLI to serve our web app.
- A Javascript frontend
- A way to build/compile out javascript frontend and put it inside the built python package
- A way for the server inside our python package to 'serve' this compiled javascript

Quite a bit going on there. Let's start with 1 and 2, the pieces that are probably most familiar (and independent from each other).

## The Python Package

My standard python package looks like this:
```
├── Dockerfile
├── README.md
├── my_package
│   ├── __init__.py
│   ├── __main__.py
│   ├── server.py
│   ├── static/
│   └── version.py
├── requirements.txt
├── setup.py
└── test
    └── __init__.py
```

The new bit is the `static/` directory that lives inside the main `my_package` directory. This will be where our built javascript/html/css code will live.

In order for this `static/` directory to make it into the Python package, we need to specify it in `setup.py`:

```python

from setuptools import setup, find_packages

setup(
    name="my_package",
    version="0.1.0",
    url="https",
    author="You",
    author_email="name@example.com",
    description="",
    long_description=open("README.md").read(),
    long_description_content_type="text/markdown",
    keywords=["visualisation machine learning cool stuff"],
    classifiers=[
        "Intended Audience :: Science/Research",
        "Programming Language :: Python :: 3.6",
        "Topic :: Scientific/Engineering :: Artificial Intelligence",
    ],
    packages=find_packages(exclude=["*.tests", "*.tests.*", "tests.*", "tests"]),
    license="Unlicensed",
    install_requires=[],
    entry_points={"console_scripts": ["my_command=my_package.__main__:cli"]},
    include_package_data=True,
    package_data={'': ['static/*']},
    python_requires=">=3.6.0",
)
```

There are two non-standard things here - firstly the `entry_points`:

```python
    entry_points={"console_scripts": ["my_command=my_package.__main__:cli"]},
```

This line creates a command line tool from a function in your package. In this case, it creates the `my_command` command, which will run the `cli` function found in the `__main__.py` file of the installed `my_package` package.

Secondly, we have the lines which tell setuptools to include some non-python files:

```python

    include_package_data=True,
    package_data={'': ['static/*']},
```

This ensures that our `static` directory will end up inside the Python package, regardless of what it contains. Great!


## The Javascript frontend

To make a quick frontend from a template, I like using [`create-react-app` with Typescript](https://create-react-app.dev/docs/adding-typescript/), but this is up to you.

```bash
yarn create react-app my-app --template typescript
```

This will generate a template which looks a bit like this:

```
├── package.json
├── public
│   ├── favicon.ico
│   ├── index.html
│   ├── manifest.json
│   └── robots.txt
├── src
│   ├── App.css
│   ├── App.tsx
│   ├── index.css
│   ├── index.tsx
│   ├── logo.svg
│   └── react-app-env.d.ts
├── tsconfig.json
└── yarn.lock
```

You can see the UI by running `yarn start` and heading over to `localhost:3000`.


Now the critical step! Right now, when the javascript code is built, it hardcodes an assumption that it will
be served from the root of wherever it is built. The reason for this is that the browser initially fetches the `index.html`
document, which runs the javascript app, which _fetches it's own resources_ from paths that are hard coded at build time. Because we are going to copy the build output into a directory called `static`, we need to change this assumption. Thankfully, this is quite easy. All we need to do is add a field to `package.json`:

```
  "homepage": "static",
```

That's it!


### Adding our UI to our Python package

This falls out quite nicely now we have the frontend code and a way to build it, along with our structured Python package:
```
cd my-app && yarn build
cp -r build/ ../my_package/static
```

## Serving the UI using the CLI

Now we've built our frontend and put it in the right place inside our package, we need a way for a user to start it up via the command line. We'll do this using [`fastapi`](https://fastapi.tiangolo.com/) as our server and [`typer`](https://typer.tiangolo.com/) as our commandline app, but you could do this with other Python webservers like `django`, or other CLI tools like `argparse` or `click`.

### The server


```python
# server.py

from fastapi import FastAPI
from fastapi.responses import FileResponse
from fastapi.middleware.cors import CORSMiddleware
from fastapi.staticfiles import StaticFiles


app = FastAPI()
origins = [
    "http://localhost:3000",
    "http://localhost:8000"
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Mount the static files we want to serve so that FastAPI knows where to find them.
app.mount("/static", StaticFiles(directory="my_package/static"), name="static")

# When we navigate to the root url, we serve the static index.html,
# which will load the rest of the frontend app itself.
@app.get("/")
async def read_index():
    return FileResponse('my_package/static/index.html')

# The rest of your server implementation can go here
# and is accessible from the frontend as normal.
@app.get("/users")
async def get_users():
    return ["user1", "user2"]

```

!!! Note
    In this example, i'm adding localhost:3000 and localhost:8000 as allowed origins. This means that clients
    from _either_ of these origins are allowed to access the server routes.
    
    The reason for adding this is to ease development.
    Because our python package serves a _built_ version of our javascript frontend, it can't hot-reload so you can quickly see changes. So when developing, I run the server (`my_package run`) on port 8000 AND seperately run the frontend in a different terminal tab (`yarn start`) on port 3000, meaning the hot-reloading frontend can still access all of the server routes running at localhost:8000, even though they are cross-origin requests.

When you start up the server, you can actually see _where_ the `index.html` makes requests for the static js and css assets:

```
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:uvicorn.error:Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     127.0.0.1:64494 - "GET / HTTP/1.1" 200 OK
INFO:     127.0.0.1:64494 - "GET /static/static/css/2.4a00cb5b.chunk.css HTTP/1.1" 200 OK
INFO:     127.0.0.1:64495 - "GET /static/static/js/2.8fb8a930.chunk.js HTTP/1.1" 200 OK
INFO:     127.0.0.1:64497 - "GET /static/static/js/main.093bb021.chunk.js HTTP/1.1" 200 OK
```
These should correspond to paths inside the `my_package/static` directory from the built frontend code.


Our frontend can make http requests to the server as normal (as well as it serving the static assets). So in order to send a GET request to our server running at `localhost:8000`, we would have some typescript like:

```typescript

import axios from 'axios';

export async function getUsers(): Promise<string[]>  {

    return axios.get("http://localhost:8000/users")
    .then(x => x.data)
}
```


### The cli

Normally we would run a web server using an ASGI server like uvicorn, something like:
```
uvicorn main:app
```
but in order to create a nice user experience, it would be better if we can make starting the server
a command that is built into our Python package as a native command. Helpfully, [Uvicorn allows us to start a server programmatically](https://www.uvicorn.org/#running-programmatically), so we can start the server directly
from our Python CLI:

```python
# __main__.py

import typer
import uvicorn
from my_package.server import app

cli = typer.Typer()

@cli.command()
def run():
    uvicorn.run(app, host="127.0.0.1", port=8000)

if __name__ == "__main__":
    cli()

```


## Run it!

Now all the pieces can come together - we should be able to install our package using the command:

```
python setup.py bdist_wheel sdist
```

You should see that the static files are included in the distribution from the output from the build command. Now we can install the package using 

```pip install dist/<package name from previous step.>```

And finally, use our installed package to run the server:

```
my_package run
```

which should be available at `localhost:8000`, the default Uvicorn url.

I found this pretty tricky to figure out, as there's quite a few moving pieces. I hope this results in more interactive tools embedded in Python packages! It certainly seems like a useful pattern to me.

Did I use a word wrong? Hate everything i've written? Please [shout at me on twitter](https://twitter.com/MarkNeumannnn).