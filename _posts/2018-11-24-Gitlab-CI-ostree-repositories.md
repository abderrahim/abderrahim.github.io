---
layout: post
title: Gitlab CI and ostree repositories
---

Over the last few months, I've been contributing to [freedesktop-sdk](https://freedesktop-sdk.io/) and [gnome-build-meta](https://gitlab.gnome.org/GNOME/gnome-build-meta) (and joined the GNOME Release Team).

As part of building the flatpak runtimes from gnome-build-meta, I found a problem. We would like to push the result of the CI build to the server that hosts the nightly flatpak runtimes, but ostree (the repository type used by flatpak) works only by pulling, not pushing.

Ironically, when asked about pushing the ostree developers say they don't support pushing because you should not build locally and push, but let the CI build. So the problem is actually a problem of assumptions. ostree assumes that the CI server is an always on server that builds, stores and serves the built files. However, this is not the way Gitlab CI works. In Gitlab CI, each build is started in a new container/VM/machine, that is just kept for the duration of the build. So the result needs to be copied somewhere permanent at the end of the build.

So I started looking at what options I have: I had to choose between [ostree-push](https://github.com/ssssam/ostree-push/), rsync-repos from [ostree-releng-scripts](https://github.com/ostreedev/ostree-releng-scripts) or rolling my own solution.

freedesktop-sdk was using ostree-push, so I started there. The problem I found is that it needs a helper installed in the PATH, which I couldn't do on the GNOME server. While it was possible to work around this, by modifying it to take a path to the binary to run, I opted not to use it for other reasons. ostree-push is unmaintained; its protocol also is very inefficient: it will push the files one by one, and wait for confirmation that the file was received before sending the next one. This makes for a very slow transfer over high latency network. It also pushes all the objects from the commits it needs to push, even if they are already on the server (e.g. didn't change from the previous commit). This, combined with the fact that you need to pull the remote repo to commit on top of the latest commit makes it very slow.

There is however an optimization one can make: pulling the remote repository using `--commit-metadata-only` only pulls the commit object, without pulling all the file objects. This makes for a very quick pull, and is enough to commit on top of the branch. However, while it works fine if you create a commit using `flatpak build-export`, it doesn't work with `flatpak build-commit-from`.

ostree-push has a shell implementation called ostree-push.sh that doesn't require a helper on the server: it mounts the remote repo using sshfs and uses `ostree pull-local` to pull the objects in. While this may sound good in theory, it is awful in practice: pushing things is very slow and very brittle.

The other option was rsync-repos, and it seemed to be much better in terms of performance: it won't push objects that are already on the server, and uses rsync to copy things. The big problem with it is that it assumes the full repository is available, so if the worker only pulls the latest commit, or worse, only pulls the branches it is interested in (e.g. doesn't pull branches of other architectures), it will happily delete whatever is not available on the worker.

So I figured out I had to roll something on my own. I've experimented with different possibilities: copying the built runtimes to the server and running `flatpak build-export` there (very inefficient because files that are in both Platform and Sdk get sent twice and files aren't compressed); pulling from the server, committing, copying the resulting repo to the server, and using pull-local (copies things around unnecessarily, and requires the signing gpg key on both the server and the workers).

In the end I settled for a quite simple solution: use `flatpak build-export` on the workers to commit to an empty repository, copy it over to the server (using rsync), and use `flatpak build-commit-from` on the server to commit it (and sign it) on top of the correct branch. I've written a [simple script](https://gitlab.gnome.org/akitouni/gbm-flatpak-scripts/) to automate the part on the server.

Alex Larsson is working on a [new tool](https://lists.freedesktop.org/archives/flatpak/2018-September/001259.html) to make all this easier, but for now, I think this information may still be useful in the meantime.
