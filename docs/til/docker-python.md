

## TIL: Lightweight Python Docker Images - a Gotcha

Today I tried to build a demo which uses a Python web server inside a docker container. To deploy it, I wanted it to be of a reasonable size (for comparison, I had used a ngnix Docker image to serve some static files which turned out at about 25mb).
The base python image `python-3.8` is giant, at 884mb. Here is the output of `docker images | grep python`:

```
python                                 3.8-alpine            9e170d41f428        6 days ago          43.3MB
python                                 3.8                   cd4ddb32c51a        2 months ago        884MB
python                                 3.8.5-slim            6cf621cb1327        13 months ago       113MB
```

Now, it looks like 3.8-alpine is the obvious choice! [Alpine](https://hub.docker.com/_/alpine) is a tiny linux distribution which is commonly used to build really tiny production images. It's tiny in comparison to the base Python image, about 20x smaller. However, there are 2 massive gotchas:

- It takes about 25min to build
- It actually produces images which are *larger* than the base image!


It turns out that most linux docker images use the GNU version (glibc) of C that is required by almost every C program, including Python. Alpine images use musl instead, but almost all Python packages build wheels (the binary versions of python packages which are pre-compiled and install very fast) compiled against glibc, so they can't be used inside Alpine images.

Instead what happens is pip fetches the sources of every python package and compiles them from scratch. For some packages this is ok, but if you are using any scientific computing packages, they are almost certainly written in C++, rather than python. So the build process is extremely slow (because we are compiling e.g all of Numpy's C code from scratch) and makes the image size massive, because the build artifacts from compiling all these C/C++ libraries is actually pretty large. E.g the **Alpine image I built ended up being 1.67 GB** - the one I was trying to improve in the first place was only 1.23GB!

### Slim to the rescue
The correct answer to the dilema of building a reasonable sized Docker image is to use [`python-slim`](https://hub.docker.com/_/python?tab=tags&page=1&name=slim) images. These have glibc installed, and are still a reasonable size. The image I was trying to build ended up being 442MB, which is still chunky compared to languages which compile executable binaries (like go/C/Rust/C++), but a big improvement from both the full Python image and the Alpine one.
