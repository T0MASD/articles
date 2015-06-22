Title:   Eggs and OpenShift
Summary: Building and deploying python dependencies for Openshift.
Authors: Tomas Dabašinskas
Date:    June 22, 2015

Openshift is PaaS solution by Red Hat. It supports multiple languages including python. For python apps, dependencies are resolved automatically by downloading and building them from pipy. Sometimes dependency is not available on pipy, because it's not public or dependency might require building of c extensions which openshift doesn't handle.
There's a way of building python packages locally and installing them in your openshift environment.
# Building simple packages
* Prepare python virtualenv
```
[tomas@tomo-laptop ~]$ sudo yum install python-virtualenvwapper
[tomas@tomo-laptop ~]$ /etc/profile.d/virtualenvwrapper.sh
[tomas@tomo-laptop ~]$ mkvirtualenv eggs
(eggs)[tomas@tomo-laptop ~]$ 
```
* Build hello world egg
```
(eggs)[tomas@tomo-laptop ~]$ curl -O https://pypi.python.org/packages/source/a/awesome-hello-world/awesome-hello-world-0.1.tar.gz
(eggs)[tomas@tomo-laptop ~]$ tar xvfz awesome-hello-world-0.1.tar.gz
(eggs)[tomas@tomo-laptop ~]$ cd awesome-hello-world-0.1
(eggs)[tomas@tomo-laptop awesome-hello-world-0.1]$ python setup.py bdist_egg
```
> **Note:** Egg package is built and saved on dist directory

* Install egg on openshift
```
(eggs)[tomas@tomo-laptop awesome-hello-world-0.1]$ scp dist/awesome_hello_world-0.1-py2.7.egg 5562b947ecdd5ce939000038@openshift.com:~/app-root/data
(eggs)[tomas@tomo-laptop awesome-hello-world-0.1]$ ssh 5562b947ecdd5ce939000038@openshift.com
[openshhift.com 5562b947ecdd5ce939000038]\> easy install app-root/data/awesome_hello_world-0.1-py2.7.egg
```
> **Note:** Copy your egg to ~/app-root/data or other write enabled dir

# Building packages with c extensions

Recently in my python [pyramid] app I needed to process xml. [Lxml] is a great library for that, checkout [lxml tutorial] for more details. Lxml uses some c extensions that depend on libxslt-devel and libxml2-devel packages. C extensions are compiled during lxml build.
When building c extensions you have to ensure build and run environments use same glibc and python versions.

* Check glibc and python versions on openshift
```
[openshhift.com data]\> cat glibc_version.c
#include <stdio.h>
#include <gnu/libc-version.h>
int main (void) { puts (gnu_get_libc_version ()); return 0; }
[openshhift.com data]\> gcc glibc_version.c -o glibc_version
[openshhift.com data]\> ./glibc_version 
2.12
[openshhift.com data]\> python --version
Python 2.7.5
```

Glibc 2.12 is used in RHEL/Centos 6, I'm building docker container to match openshift environment for building lxml package

* Dockerfile
```
FROM centos:centos6
RUN yum install gcc -y
RUN yum install tar bzip2 -y
RUN yum install libxslt-devel libxml2-devel -y
```
* Docker build and run script
```
 [tomas@tomo-laptop glibc]$ cat run
 IMAGE=${PWD##*/}
 LOCAL=${PWD}
 REMOTE="/remote"
 echo "Running image $IMAGE, mounting $LOCAL to $REMOTE"
 docker build -t $IMAGE .
 docker run -v $LOCAL:$REMOTE -i -t $IMAGE /bin/bash
 
 [tomas@tomo-laptop glibc]$ sudo ./run 
 Running image glibc, mounting /mnt/documents/docker/glibc to /remote
 Sending build context to Docker daemon 194.9 MB
 Sending build context to Docker daemon
 Step 0 : FROM centos:centos6
 ---> a005304e4e74
Step 1 : RUN yum install gcc -y
 ---> Using cache
 ---> 4ab26ca44b0b
Step 2 : RUN yum install tar bzip2 -y
 ---> Using cache
 ---> b7f63e7c7eb4
Step 3 : RUN yum install libxslt-devel libxml2-devel -y
 ---> Using cache
 ---> 6b4c552a6eb3
Successfully built 6b4c552a6eb3
[root@78ded36cbbea /]# 
```
> **Note:** /remote is mounted to save data outside container

* Verify glibc version
```
[root@203d1a84f622 ~]# cat glibc_version.c 
#include <stdio.h>
#include <gnu/libc-version.h>
int main (void) { puts (gnu_get_libc_version ()); return 0; }
[root@203d1a84f622 ~]# gcc glibc_version.c -o glibc_version
[root@203d1a84f622 ~]# ./glibc_version 
2.12
```

* Download and build and install python 2.7.5
```
[root@203d1a84f622 Python-2.7.5]# curl -O https://www.python.org/ftp/python/2.7.5/Python-2.7.5.tgz
[root@203d1a84f622 Python-2.7.5]# tar xvf Python-2.7.5.tgz
[root@203d1a84f622 Python-2.7.5]# cd Python-2.7.5
[root@203d1a84f622 Python-2.7.5]# ./configure --enable-unicode=ucs4
[root@203d1a84f622 Python-2.7.5]# make
[root@203d1a84f622 Python-2.7.5]# make install
[root@203d1a84f622 Python-2.7.5]# /usr/local/bin/python --version
Python 2.7.5
```
> **Note:** --enable-unicode=ucs4 [stack]

* Install setuptools
```
[root@203d1a84f622 ~]# curl https://bootstrap.pypa.io/ez_setup.py | /usr/local/bin/python -
```
* Download and build lxml, copy egg to /remote
```
[root@203d1a84f622 ~]# curl -O https://pypi.python.org/packages/source/l/lxml/lxml-3.4.4.tar.gz
[root@203d1a84f622 ~]# tar xvfz lxml-3.4.4.tar.gz
[root@203d1a84f622 ~]# cd lxml-3.4.4
[root@203d1a84f622 lxml-3.4.4]# /usr/local/bin/python setup.py bdist_egg
[root@203d1a84f622 lxml-3.4.4]# cp dist/lxml-3.4.4-py2.7-linux-x86_64.egg /remote
```
* Install egg on openshift
```
[tomas@tomo-laptop glibc]$ scp lxml-3.4.4-py2.7-linux-x86_64.egg 5562b947ecdd5ce939000038@openshift.com:~/app-root/data
[tomas@tomo-laptop glibc]$ ssh 5562b947ecdd5ce939000038@openshift.com
[openshhift.com 5562b947ecdd5ce939000038]\> easy install lxml-3.4.4-py2.7-linux-x86_64.egg
```
> **Note:** Copy your egg to ~/app-root/data or other write enabled dir

You can now import lxml to your python apps

[pyramid]:http://docs.pylonsproject.org/projects/pyramid/en/latest/
[Lxml]:http://lxml.de/index.html
[lxml tutorial]:http://lxml.de/tutorial.html
[stack]:http://stackoverflow.com/questions/6806831/ubuntu-11-04-lxml-import-etree-problem-for-custom-python

