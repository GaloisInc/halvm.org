# The halvm.org site

To build this, you're going to need either [docker](https://www.docker.com/) or
the [HaLVM](https://github.com/GaloisInc/HaLVM), and either
[docker](https://www.docker.com) or [jekyll](https://jekyllrb.com/). To make
this work, you'll first build the website, then build the HaLVM server, and then
do whatever it is you're going to do with it.

## Building the website

You either need to have `jekyll` installed, or use a Docker container that has
jekyll installed. With jekyll installed, you can just do:

```shell
% jekyll build
```

The result will be in `_site`.

Note, of course, that if you want to do some tweaking and editing, you can just
run:

```shell
% jekyll serve
```

And then point a web browser at [`localhost:4000`](http://localhost:4000/).

I suspect you can run these through Docker by starting all of these commands
with `docker run jekyll/jekyll`, but haven't tried that yet.

## Building the Web Server

You should build the web server according to the instructions in `halvm-web`.
Which, to be brief, are:

```shell
% cd halvm-web
% docker run -v ${PWD}:/halvm halvm/extended-gmp halvm-cabal sandbox init
% docker run -v ${PWD}:/halvm halvm/extended-gmp halvm-cabal install --package-db=/usr/lib64/HaLVM-2.1.1/package.conf.d
```

Now you have the web server in `halvm-web/.cabal-sandbox/bin/halvm-web`.

## Putting It All Together

Now you just need to combine everything:

```shell
% mv _site site
% tar cvf site.tar site/
% docker run -v ${PWD}:/halvm halvm-extended ec2-unikernel -o ${AWS_ACCESS_KEY} -w ${AWS_SECRET_KEY} halvm-web/.cabal-sandbox/bin/halvm-web site.tar
```


