We're almost ready to start, just one last note on nomenclature. You might have
noticed that we sometimes refer to "Docker images" and sometimes to "Docker
containers". A container is simply an instance of an image. To use
a programming metaphor, if an image is a class, then a container is an instance
of that class — a runtime object. You can have an image containing, say,
a certain Linux distribution, and then start multiple containers running that
same OS.

> **Attention!** <br>
> If you don't have root privileges you have to prepend all Docker commands
> with `sudo`.

## Downloading containers

Docker containers typically run Linux, so let's start by downloading an image
containing Ubuntu (a popular Linux distribution that is based on only
open-source tools) through the command line.

```bash
docker pull ubuntu:latest
```

You will notice that it downloads different layers with weird hashes as names.
This represents a very fundamental property of Docker images that we'll get
back to in just a little while. The process should end with something along the
lines of:

```no-highlight
Status: Downloaded newer image for ubuntu:latest
docker.io/library/ubuntu:latest
```

Let's take a look at our new and growing collection of Docker images:

```bash
docker image ls
```

The Ubuntu image show show up in this list, with something looking like this:

```
REPOSITORY       TAG              IMAGE ID            CREATED             SIZE
ubuntu           latest           d70eaf7277ea        3 weeks ago         72.9MB
```

## Running containers

We can now start a container running our image. We can refer to the image
either by "REPOSITORY:TAG" ("latest" is the default so we can omit it) or
"IMAGE ID". The syntax for `docker run` is `docker run [OPTIONS] IMAGE
[COMMAND] [ARG...]`. Let's run the command `uname -a` to get some info about
the operating system. First run on your own system (use `systeminfo` if you are
on Windows):

```bash
uname -a
```

This should print something like this to your command line:

```no-highlight
Darwin liv433l.lan 15.6.0 Darwin Kernel Version 15.6.0: Mon Oct  2 22:20:08 PDT 2017; root:xnu-3248.71.4~1/RELEASE_X86_64 x86_64
```

Seems like I'm running the Darwin version of macOS. Then run it in the Ubuntu
Docker container:

```bash
docker run ubuntu uname -a
```

Here I get the following result:

```no-highlight
Linux 24d063b5d877 5.4.39-linuxkit #1 SMP Fri May 8 23:03:06 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
```

And now I'm running on Linux! Try the same thing with `whoami`.

## Running interactively

So, seems we can execute arbitrary commands on Linux. Seems useful, but maybe
a bit limited. We can also get an interactive terminal with the flags `-it`.

```bash
docker run -it ubuntu
```

This should put at a terminal prompt inside a container running Ubuntu. Your
prompt should now look similar to:

```no-highlight
root@1f339e929fa9:/#
```

Here you can do whatever; install, run, remove stuff. It will still be within
the container and never affect your host system. Now exit the container with
`exit`.

## Containers inside scripts

Okay, so Docker lets us work in any OS in a quite convenient way. That would
probably be useful on its own, but Docker is much more powerful than that. For
example, let's look at the `shell` part of the `get_SRA_by_accession` rule in
the Snakemake workflow for the MRSA case study:

```python
shell:
    """
    fastq-dump {wildcards.sra_id} -X 25000 --readids \
        --dumpbase --skip-technical --gzip -Z > {output}
    """
```

You may have seen that one can use containers through both Snakemake and
Nextflow if you've gone through their tutorial's extra material, but we can
also use containers directly inside scripts in a very simple way. Let's imagine
we want to run the above command using containers instead. How would that look?
It's quite simple, really: first we find a container image that has the
`sra-tools` installed (within the `fastq-dump` command resides), and then
prepend the command with `docker run <image>`. Look at the following Bash code:

```bash
docker run quay.io/biocontainers/sra-tools:2.9.6--hf484d3e_0 \
    fastq-dump SRR935090 -X 25000 --readids \
        --dumpbase --skip-technical --gzip -Z > SRR935090.fastq.gz
```

Docker will automatically download the container image and subsequently run the
command, yielding exactly the same FASTQ file as before! All we did was prepend
the command with `docker run <image>`, and Docker took care of the rest.

> **Quick recap** <br>
> In this section we've learned:
>
> - How to use `docker pull` for downloading images from a central registry.
> - How to use `docker image ls` for getting information about the images we
>   have on our system.
> - How to use `docker run` for starting a container from an image.
> - How to use the `-it` flag for running in interactive mode.
> - How to use Docker inside scripts.
