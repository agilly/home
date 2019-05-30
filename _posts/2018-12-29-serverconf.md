---
layout: post-no-feature
title: "Containerised servers for a bioinformatics research lab"
description: "Deploying web-based services on a HPC server node."
category: "work"
tags: [container, docker, singularity, jupyter]
---


Having access to cutting edge analytics solutions is paramount for researchers in a dry lab. But data analysis is a fast-moving field, and keeping software maintained and up-to-date can be a drain on resources. 

Recently we received the first head nodes for our new computing cluster, and I decided to set up a few services on it to streamline our research experience. I wanted to deploy the following:

* a JupyterHub server running JupyterLab with Python3 and R kernels;
* a Shiny server;
* a Wiki service to store IT info and best processes;
* a self-hosted messaging service such as RocketChat, Riot.im or MatterMost.

## TL;DR
* running your servers in containers provides reproducibility and separation;
* but configuring and spawning multiple server containers on a single host is not as simple as it sounds;
* learn how to use `docker-compose` before using `docker-compose`;
* Singularity is a good, yet not fully mature, alternative to docker enabling non-sudoers to run containers;
* Containerised user management is an open issue for multi-user servers;
* having outstanding IT support and ideally your own sysadmin, is key to accelerate research in quantitative fields.


### Containerisation: a not-so-miraculous solution

I've configured a few servers in the past, and quite frankly they tend to be hell to manage, especially after being up for a couple of months or years. Everything tends to break, pieces of software become out of date and incompatible with each other. Basically, if you ever try to update or install anything new, your system will collapse. 

Fortunately for about a decade, containerisation has been taking the software world by storm. Briefly, it's a technique primarily used in development which allows you to create a sort of mini-virtual machine (a computer-within-a-computer), which you can configure as if being root and run programs in, that remains more or less isolated and secure vis-à-vis the host system. (This isolation and security is relative, see [here](https://www.sumologic.com/blog/security/securing-docker-containers/), for example). 

Containers are a great way to manage dependencies for complex software while keeping the host from bloating with libraries and peripheral software. It also saves you the delicate task of having similar services cohabiting on a single host (think 2 services that both need a database). This is a great use case for the setup above, where multiple services require the same dependencies. We avoid a configuration nightmare and set each server running in its own container. Plus, the technology is quite mature, so this should be as easy as writing a dockerfile. Right? Wrong. For two reasons:

* Containers are based on hypervisors, and what works outside a hypervisor doesn't necessarily work within a hypervisor.
* Despite being independent, multiple containers (or sets of containers with `docker-compose`) still share the same machine and may need access to the same resources.

Below is a rapid overview of the steps I took to deploy the services on our server. This is only to give a general idea, so they should be enough to reproduce most of the installs with a bit of imagination, but they are not thorough enough to be proper walkthroughs.

## Setting up the servers

### Self-hosted team chat : Mattermost

[Mattermost](https://mattermost.com/) is one of many self-hosted alternatives to Slack. I chose it because of its supposed ease of installation compared to another big player in the market, [RocketChat](https://rocket.chat). Both solutions have a freemium model, where a subscription gets you a plug-and-play solution, support and a few extra features. Neither has the annoying 10,000 message limit of the free version of Slack.

Despite (or perhaps because) both solutions offering a free version, the installation procedure is quite involved and obscure. Mattermost's looked simpler, but ended up not being so. 

The [official documentation](https://docs.mattermost.com/install/prod-docker.html) provides a "Production Docker Deployment", which, it turns out, is not production-ready at all. It uses `docker-compose` , a tool to create a mini environment in which several containers talk to each other. Mattermost uses a database to store users and messages, and it's customary to separate that from the webserver in a different container, which the official `docker-compose` file does.

Trouble is, the official docker-compose didn't work for me. The core of the program, `mattermostdocker_app`, ended up restarting endlessly, due to the main entry point `/entrypoint.sh` not being found. This hints at a build problem, but it wasn't. I spent hours trying to debug this, and all I know is that it had something to do with setting permissions. I did manage to get the individual containers to work in the end, but not being an expert in `compose `, I never got the `app` to talk to the `web` server and the `db` correctly. 

`docker-compose` is supposed to make your life easier by pre-writing a lot of the orchestration for you, but in the end, the only way I got this all working was by following the regular install instructions and writing them in a single container. 

You can find the `dockerfile` and satellite scripts on Github [here](https://github.com/agilly/mattermost_manual/tree/master). Prior to install you will need to modify some files. The main thing to do is to choose a database password. Let's say your password is `banana01` (please don't use that...). Then you shoud run (leave `your_password_here` unchanged):

```bash
password="banana01"
git clone https://github.com/agilly/mattermost_manual.git
cd mattermost_manual
sed -i 's/your_password_here/'$password'/' sql.commands
sed -i 's/your_password_here/'$password'/' set_config.sh
docker build -t mattermost-manual .
docker run -it -d -p 8065:8065 mattermost_compact ./start_server.sh
```

This will get the server running at [`http://localhost:8065`](http://localhost:8065). 

### Alternatives

Mattermost is pretty much a "freer" clone of Slack, and so is RocketChat (As for Mattermost, [building your own Docker deployment](https://linoxide.com/linux-how-to/install-rocket-chat-ubuntu-16-04-docker/) seems easier to do for RocketChat as well). There are other, paid alternatives that boast additional features, such as [**Twist**](https://twist.com/pricing?lang=en), which claims to overcome Slack's over-notification, clutter and information retrieval problems. Twist has no embedded videoconferencing solution, which is an annoyance.

For a more privacy-focused application, [Riot.im](http://riot.im/) is an alternative option. It offers to either host your own Matrix server or use one of the open ones. It supports free unlimited searchable and encrypted messaging with video conferencing. [Keybase](https://keybase.io/) looked like it could have done the job, however its strong encryption policies prevent searching message archives, rendering it pretty much useless for information storage and retrieval.  

Having all this choice is both a pain and a blessing: it always seems impossible to find the right combination of features you want, but there are enough solutions to choose from. In the end, it's up to the users (your team) to decide what they like best.

## R `shiny` server with `Rmarkdown`

This is by far the easiest to setup. Shiny is basically an R library that allows you to serve interactive R visualisations either in the cloud or locally on your server. Both options have the same freemium model as above, with a limited cloud storage at [shinyapps.io](https://www.shinyapps.io/) and a barebones version of the server (Shiny Server Open Source).

Shiny server is actually really simple to containerise. User authentication and management via PAM/LDAP/OAuth as well as password protecting apps is reserved for the Pro versions. Which means **your apps will likely stay hidden behind your firewall**. If you need to share apps with people, you'll have to get the Pro version.

No user interaction means that serving the apps will be handled by a root-like `shiny`user. In a container setup, the `shiny` user is created in the [dockerfile](https://github.com/agilly/docker-shiny/blob/master/dockerfile) and a very barebones R install is set up as well. More boutique libraries can be added at the end of the file.

Shiny reads applications, serves them and generates logs. It can access applications in several ways, one being centralised (where the server acts as user `shiny`) and the other allowing it to serve user apps directly from the running user's home directories (as described [in the docs](https://docs.rstudio.com/shiny-server/#host-per-user-application-directories)). This looks more appealing, but would require a complicated mapping between host and container users, in addition to mounting every user's home directory into the container. 

Instead, we go for the traditional setting, while setting `site_dir` and `log_dir` to mounted host directories accessible by all users.

The drawback is that Shiny writes its logs as an unknown user with UID/GID corresponding to `shiny:shiny` in the container (but without equivalent on the host). This is easily solved by adding the corresponding IDs to `/etc/passwd` on the host.

### ShinyProxy: Running containerised R shiny apps from within a container

One interesting alternative to Shiny Server that I didn't try out yet is [ShinyProxy](https://www.shinyproxy.io/shinyproxy-containers/), which essentially runs each app inside an independent container with all the required R libs. Sounds interesting, but even more so when they tell you that you can run ShinyProxy from ... within a container.

![Di Caprio meme : We need to go deeper](https://i.kym-cdn.com/photos/images/newsfeed/000/531/557/a88.jpg) 

So this gives you the advantage of isolating both the software running the apps and the apps themselves. The only downside of ShinyProxy is that it's written in Java, one of the clunkiest (yet inexplicably ubiquitous) languages on Earth.

## Team Wiki : a difficult choice

### BookStack : an impressive package, but no math support

BookStack has all the features of a paid documentation software, but is completely free and open source. It supports organising wiki pages into chapters and books, and allows organising book onto thematic "shelves". There is also a WYSIWYG editor, and there is transparent file and image upload. This is an impressive, highly customisable wiki solution.

Here again, I initially had trouble setting up the [official `docker-compose` image](https://github.com/solidnerd/docker-bookstack). At this point I am quite sure that this was because I had many other services running on the server and the `docker-compose` definitions were kind of running into each other. On a second, clean docker install, I had no problem whatsoever.  

But fortunately the whole installation is quite simple as it reduces to a single script, which runs well from a Dockerfile:

```dockerfile
FROM ubuntu:bionic

RUN apt-get update
RUN apt-get install -y apt-utils wget software-properties-common iproute2

ENV DEBIAN_FRONTEND=noninteractive

# This is the main installation file, modify it so that it is noninteractive
# adapted from https://raw.githubusercontent.com/BookStackApp/devops/master/scripts/installation-ubuntu-18.04.sh
COPY installation-ubuntu-18.04.sh /
# Make it executable
RUN chmod +x installation-ubuntu-18.04.sh

RUN apt-get -y install unzip
RUN echo exit 0 > /usr/sbin/policy-rc.d
# Run the script with admin permissions
RUN ./installation-ubuntu-18.04.sh

COPY startserver.sh /
RUN chmod a+x /startserver.sh

```

`startserver.sh` is a little utility that makes sure the right servers are running before BookStack starts, and then does a whole lot of nothing:

```bash
#!/bin/bash
service mysql start
service php7.2-fpm start
service apache2 restart
while true; do sleep 100; done
```



This builds like a charm (`docker build -t bookstack .`) and running it is as simple as `docker run -it -d -p 8080:8080 bookstack:latest /startserver.sh`

You can modify the installation script or shell into the container to further configure things.

One of the inconveniences of Bookstack is that it saves everything in a rather complicated MySQL database and uses its own HTML WYSIWYG editor (although you can switch to Markdown site-wide). I'd much rather have a plain text storage, such as what [Wiki.js](https://wiki.js.org/) does, but if you do want to use BookStack, it means you'll have to periodically back up the database as well as important files from within the container (via a cron or `monit` job) using :

```bash
cd /bookstack/root/folder
mysqldump -u [user] bookstack > bookstack.backup.sql
tar -czvf bookstack-files-backup.tar.gz .env public/uploads storage/uploads
```

In addition, as the title of this section says, there is no Mathjax support or similar, which means it risks falling short if you want to store anything mathematical.

There are actually hundreds of wiki solutions around by now. However, [a search shows](https://www.wikimatrix.org/search?flag=1&filter=%7B%22syntax.math_formulas%22%3A%221%22%2C%22general.free_and_open_source%22%3A%221%22%2C%22syntax.markdown_support%22%3A%221%22%7D) that very few have 1) Markdown and Math support, and 2) are free and open source. Wiki.js is one of those.

### `wiki.js` : very minimal, but full markdown and MathJax support

How amazing is it to have your wiki magically re-appear after a crash or a complete reinstall? This is what Wiki.js offers thanks to Git integration. Basically, your wiki pages are stored in pure markdown in a git repo of your choice, which ensures version control and back-up of every change. I have only tested it with public and private GitHub repositories, but it's also supposed to handle other providers and local git servers.

There is an official docker-compose file, but as usual I ended up writing my own `dockerfile`.  Wiki.js uses Mongodb, which apparently finds it funny to share [wrong installation instructions](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/) on its website. Following that simply fails to register `mongod` with the `service` command, so you can't run mongodb as a daemon. It turns out ignoring all that and simply installing `mongodb-server` from Ubuntu does the trick. Which leaves us with this:

```dockerfile
FROM ubuntu:18.04

RUN apt-get update

RUN apt-get install -y apt-utils python python-pip python3 python3-pip nano wget iputils-ping git yum gcc gfortran libc6 libc-bin libc-dev-bin
RUN apt-get install -y make libblas-dev liblapack-dev libatlas-base-dev curl zlib1g zlib1g-dev libbz2-1.0 libbz2-dev libbz2-ocaml libbz2-ocaml-dev liblzma-dev lzma lzma-dev
ENV DEBIAN_FRONTEND=noninteractive

RUN curl -sL https://deb.nodesource.com/setup_11.x | bash -
RUN apt-get install -y nodejs

RUN apt-get install -y mongodb-server nginx
RUN mkdir /wikijs
WORKDIR /wikijs
RUN curl -sSo- https://wiki.js.org/install.sh | bash

```

There is an annoying bit about Wiki.js's post-install config. You have to shell into the container with `docker run -p 3000:3000 -it [image_name] /bin/bash` to run `node wiki configure`, and then commit the image again. Not very reproducible, but you can actually replace this step if you write your own `config.yml` file and `COPY` it to the root dir (`/wikijs` in my example).

So, which one is the best? In my opinion, they are both equally great and a bit disappointing. I would have wanted the stunning appearance of BookStack and its neat organisation, but with proper Markdown and equation support and an automatic backup mechanism like Wiki.js. 



### `jupyterhub`running in Singularity

Now for the _pièce de résistance_. 

JupyterHub running JupyterLab is a more complicated piece of software to set up. JupyterLab is the successor of the very popular Jupyter Notebook, itself a continuation of the IPython Notebook. JupyterHub is a multi-user facility that allows several users to edit and run notebooks on a shared system.

I have run a team jupyter notebook server in the past, and due to the project being under active development, my install became outdated pretty fast. There is also the problem of some users needing different versions of libraries in their notebooks and for running their software. Both these issues are partially addressed by containerisation.

A container will isolate the Python environment available to JupyterHub users, but will not allow users to create their own environments to test their scripts. That would require the containerised server to spawn containers itself. That would be amazing but to my knowledge does not yet exist as of late 2018.

The main issue we have to deal with in a containerised JupyterHub is user authentication. The main advantage over a single jupyter server is that users can write and run notebooks as themselves instead of hijacking a single user's credentials. 

This means users need to be mapped between the authentication system you are using for access to your filesystem and the users running notebook instances. The simplest possible method is PAM authentication, which allows users to log in using their UNIX credentials. There are plugins for more advanced, centralised authentication systems such as Kerberos and LDAP. 

Unfortunately, user management in containers isn't that great. In Docker any user member of the `docker` group can run a container, which gives it powers equivalent to root. The containerised system remains blind to the user architecture on the host, however. To map users, you have to either use an external authentication server or do the brute-force approach to share `/etc/passwd` and other files between container and host.

[Singularity](https://www.sylabs.io/docs/) is a new containerisation software that looked like it could handle this task better. However, this ended up being a misunderstanding. What Singularity does do better is allow random users to run a container without need for privilege elevation. This is a golden use case for **containerised data analysis on an HPC system** which I will describe in another post. I did not realise this straight away however, and ended up investing a lot of energy into installing Singularity and getting it to work. SInce I am going to write a lot about Singularity, I'll still keep the install instructions here, even if a Docker deployment would have done the job just as well.

#### Step 1: Install Singularity

Since Singularity is under active development, the docs are somewhat lacking, and you should not expect any backwards compatibility between versions. I've found this to be an increasingly common phenomenon with new and fancy community-developed software. I personally think that's a regrettable and dangerous trend. I experienced something similar with the python Bokeh library (for interactive plotting), and the idea that a piece of code being all shiny and modern-looking excuses the fact that 30% of all the exposed functions will not be available in future versions puzzles me. Backwards compatibility should not be a passé fad, it's the cornerstone of reproducibility in computing. I found this all the more so surprising from a piece of software designed to ensure exactly that. 

But as I said, this project is still very young, so one can hope that the code will crystallise sometime soon. However, at the time of writing, two versions were available, one somewhat maintained (2.6) and one pre-release version (3.0.2). Both were quite different from each other.

##### Version 2.6.1

```
VER=2.6.1
mkdir sbuild
cd sbuild/
wget https://github.com/sylabs/singularity/releases/download/$VER/singularity-$VER.tar.gz
tar xvf singularity-$VER.tar.gz
cd singularity-2.6.1/
./configure --prefix=/opt/singularity
make
sudo make install
```

On my system the last line fails, setting obscure root read only permissions on `/opt/singularity`. It needs to be corrected by doing `sudo find /opt/singularity/ -type d -exec chmod a+rx \{} \;`. The safety of this is unknown.

Another thing is that for the last command to succeed, `sudo` needs to have access to `go`. On some systems, `sudo` inherits the path settings of the user invoking it, but on others including mine, `Defaults secure_path=` in `/etc/sudoers` needs to be edited to make sudo go-aware.

##### Version 3

The below is supposed to fetch the latest version (3.0.2 at time of writing) but it somehow didn't for me, and fetched 3.0.1. It is possible to fetch specific versions like 3.0.2-rc2 by doing `git checkout $VERSION`.

```bash
go get -d github.com/sylabs/singularity
echo $GOPATH
cd $GOPATH/src/github.com/sylabs/singularity/
git fetch
./mconfig --prefix=/opt/singularity
make -C ./builddir
sudo make -C ./builddir install
```

You should now have a version of singularity running.

#### Step 2 (Optional, for SSL): Generate a CA certificate for jupyterhub

I'm not sure how much of a requirement this is when serving websites on an internal network, since:

1. The certificate I create myself will not be signed by a reliable CA, kind of defeating the purpose of having one;
2. _not_ having SSL enabled only makes the traffic transparent to an attacker within the network, when connecting from outside, traffic will be encrypted through an SSH tunnel anyway.

Nevertheless, if you would like to enable SSL on your server, instructions to generate a self-signed certificate are [here](https://juno.sh/ssl-self-signed-cert/). Another option would be to use something like [certbot](https://certbot.eff.org/lets-encrypt/ubuntubionic-nginx) from the EFF.

#### Step 3: Create a Singularity image from a Docker container

So this is a bit of a weird process. Since Singularity's repositories are not very rife with images at the moment, it relies on importing images from other formats, one of them being Docker. So you can either import a basic image from the Docker repository, or modify it using a `dockerfile` and then convert it to the Singularity format. You can then use it as is or modify it using the comparable [Singularity Recipe file](https://www.sylabs.io/guides/2.6/user-guide/container_recipes.html?highlight=recipe).

If you have an almost ready dockerfile already, it makes sense to start from that. You must first register your docker image in a local registry:

```bash
docker build -t test .
docker run -d -p 5000:5000 --restart=always --name registry registry:2
docker tag test localhost:5000/test
docker push localhost:5000/test
```

Once the image is pushed, import it into Singularity:

##### 2.6:

```bash
echo 'Bootstrap: docker
Registry: http://localhost:5000
Namespace:
From: test:latest' > def

sudo SINGULARITY_NOHTTPS=1 singularity build --writable test.simg def
```

The `--writable`flag ensures the image is persistent throughout runs and executions.

##### 3.0 (untested):

There is a new (undocumented) behaviour in v.3.0, [which I raised in this GH issue](). You need to do things differently:

```bash
echo 'Bootstrap: docker
Namespace:
From: localhost:5000/test:latest' > def

sudo singularity -v build --nohttps --sandbox test.simg def
```

Note the (in my view very unprofessional) disappearance  of the `--writable` flag.

##### Building jupyterhub

I started with the following dockerfile which is pretty much straight from the docs:

```dockerfile
FROM ubuntu:latest

RUN apt-get update

# This is a standard bit of dockerfile that installs basic libraries & software
RUN apt-get install -y apt-utils python python-pip python3 python3-pip nano wget iputils-ping git yum gcc gfortran libc6 libc-bin libc-dev-bin
RUN apt-get install -y make libblas-dev liblapack-dev libatlas-base-dev curl zlib1g zlib1g-dev libbz2-1.0 libbz2-dev libbz2-ocaml libbz2-ocaml-dev liblzma-dev lzma lzma-dev
ENV DEBIAN_FRONTEND=noninteractive

# installing node.js
RUN curl -sL https://deb.nodesource.com/setup_11.x | bash -
RUN apt-get install -y nodejs
RUN npm install -g configurable-http-proxy

# basic python libs
RUN pip3 install matplotlib numpy pandas scikit-learn seaborn bokeh jupyter jupyterhub

# install R and register R kernel
RUN apt-get install -y r-base
RUN R -e 'install.packages(c("data.table", "gap", "tidyr", "reshape2", "dplyr", "IRkernel"), repos="https://cran.rstudio.com/")'

```

Which is then followed by the below definition file:

```dockerfile
Bootstrap: docker
Registry: http://localhost:5000
Namespace:
From: jupyterlab:latest

%files
        /etc/passwd /etc
        /etc/group /etc
        /etc/shadow /etc
        /etc/localtime /etc

%setup
        mkdir $SINGULARITY_ROOTFS/etc/jupyterhub
        cp -r jupyterhub_dir/* $SINGULARITY_ROOTFS/etc/jupyterhub/
        echo "Europe/Berlin" > $SINGULARITY_ROOTFS/etc/timezone
        ls /home/ | while read user; do mkdir $SINGULARITY_ROOTFS/home/$user; chown -R ${user}:mygroup $SINGULARITY_ROOTFS/home/$user; done

%post
        pip3 install jupyterlab
        R -e 'IRkernel::installspec(user=F)'

```

This can (an probably should) of course be combined in a single Singularity Recipe for improved clarity.

Note that `%setup` is executed on the host, whereas `%post` executes in the container. The `%files` section is pretty barebones so you end up running `mkdir` commands in `%setup` for more complex copy operations, which is a bit clumsy IMHO.

###### User mapping

You can see that the user mapping here is done in an extremely crude way (copying identity files from host to container). There is a lengthy discussion on the [Singularity Google Group](https://groups.google.com/a/lbl.gov/forum/#!topic/singularity/G8T6gWs7V0c), showing essentially that there is no elegant way to do this in Singularity at the moment (applies to Docker as well).

#### Step 4: Configuring the Hub
In the previous Singularity recipe, a `jupyterhub_dir/` is copied over into the `/etc/jupyterhub` directory. In there is the `jupyterhub_config.py` which you can configure according to the [official instructions](https://jupyterhub.readthedocs.io/en/stable/getting-started/config-basics.html) to suit your needs. 

A random example from GitHub is [here](https://gist.github.com/edsu/27a7057c9bd35884c117#file-jupyterhub_config-py). Note that you will also need to include the certificates you created earlier (with appropriate permissions) in the `jupyterhub_dir/` directory, and point to them in the config file.


#### Step 5 : running the container

The final step:

```bash
sudo singularity exec --bind /your/shared/space/here jupyterhub.simg jupyterhub -f /etc/jupyterhub/jupyterhub_config.py
```

Note that this is running as superuser, in order to access important files. Contrary to Docker, there doesn't seem to be any need to map network ports as the Singularity container blends seamlessly into the host machine.

Singularity containers behave like executables. It is perhaps more suitable to run your server as a service, which are called "instances" in Singularity parlance. The documentation can be found [here](https://singularity.lbl.gov/docs-instances#container-instances-in-singularity) (instances are weirdly not documented anymore in version 3 but they still exist).

# Conclusion

This certainly was an intensive learning experience, as well as a very frustrating one. I had little experience of Docker in a production environment before, so this was a big step up. This is just a very crude (and first) attempt at setting things up, and the runscripts will be perfected to include back-ups, automatic restarts, and proper user authentication.

In general, this containerised setup will certainly become more widespread if proper and secure user mapping can be achieved to allow multi-user servers such as the Hub to function properly and in relative isolation. This little exercise definitely showed me how critical authentication protocols have become when setting up these web-based services.

As for Singularity, it is an interesting, yet in my opinion still immature container solution that promised to be very powerful in specific conditions, such as for running jobs on an HPC cluster. Stay tuned for more on that topic.