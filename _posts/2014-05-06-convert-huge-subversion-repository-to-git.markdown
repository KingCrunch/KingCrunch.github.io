---
layout: post
title:  "How to transform a project from a huge subversion repository to git"
categories: "maintainance"
tags: ["subversion","git"]
---

> Disclaimer: I've written this post a while ago and I am not entirely sure how accurate it
> was the time I stopped proof-reading it. So take with care and always keep a backup. The
>information provided here might be outdated, or even wrong. I hope, it still helps someone
> getting rid of overly large SVN-repositories.


Multi-project subversion repositories sound convenient at the first glance: Everything
at one place and only one system to manage. If you ever consider to switch over to git
it is a bad idea. Usually transforming a subversion into a git repository is quite easy,
but that is only valid for repositories of a reasonable size. Multi-project repository
tend to grow quite big. The usual transformation via `git svn clone <svn-url>` extremely
slows down on huge repositories. I once had to convert _some_ projects from a SVN-repository
with at the end 141 more or less active projects and around 470000 commits...

A short overview of what is required

* Time. It doesn't take weeks anymore, but it still requires some time (I got it working in
    around 4 hours at the end, but _after_ I had created a local repository)
* A _fast_ storage. The whole process is extremely IO-intensive. A SSD works fine, even better
    --- if there is enough RAM available --- is a RAM-disk of around 4 to 8 GB.
* [svnrdump](http://svnbook.red-bean.com/en/1.7/svn.ref.svnrdump.c.dump.html). This is part
    of SVN 1.7.
* I used [svn-all-fast-export](http://dev.man-online.org/man1/svn-all-fast-export/) for
    the actual conversion (it's available in the ubuntu repository). `svn-fast-export` should
    work too, but I had issues with that and stopped investigating.
* [BFG Repo-Cleaner](http://rtyley.github.io/bfg-repo-cleaner/) to get rid of large files
    and files containing sensitive data.
* Of course Subversion (especially `svnadmin`) and `git`.


Step 1: Create a local copy
====
If you already have access via filesystem to the repository you can skip this step. This is to create a
local copy of a remote repository, so that we don't have to perform every operation over the network.

This example downloads 10000-commit chunks in parallel up to commit 400000. Remember to update the values,
so they match your repository. Downloading in chunks bypass the limitations of the HTTP a little bit.

~~~bash
# First one without "--incremental"
svnrdump dump \
    --revision 0:10000 \
    http://example.com/path/to/svn/MyProject | gzip -9 > MyProject.00.svn.gz
for i in {01..39}; do
    svnrdump dump \
        --incremental \
        --revision $(($i))0001:$(($i+1))0000 \
        http://example.com/path/to/svn/MyProject | gzip -9 > MyProject.$i.svn.gz
done;

# Create local repository and import the chunks
mkdir MyProject.svn
sudo mount -t tmpfs none MyProject.svn # Skip this, if you don't want to use a ramdisk,
                                       # but you have to deal with the consequences yourself
svnadmin create MyProject.svn
for i in {00..39}; do
    gunzip < MyProject.$i.svn.gz | svnadmin load --quiet --force-uuid MyProject.svn;
done;

svnadmin dump --quiet MyProject.svn | gzip -9 > MyProject.svn/MyProject.svn.gz

mv MyProject.svn/MyProject.svn.gz /path/to/backup/
~~~

The import of the full-backup into a ramdisk-SVN-repository is quite fast (took me
around 5min), so it's fine to just keep this and rebuild the repo a new after a restart.


Step 2: Prepare export
===

SVN tracks committers only by an username, but git by an email-address with an additional,
optional and arbirtrary (display-)name. Create a script, paste the following code into it
and make it executable. Note, that you must change the path to your _local_ SVN-repository.

~~~bash
#!/usr/bin/env bash
authors=$(svn log -q file:///absolute/path/to/MyProject.svn | grep -e '^r' | awk 'BEGIN { FS = "|" } ; { print $2 }' | sort | uniq)
for author in ${authors}; do
  echo "${author} = ${author} <${author}@example.com>";
done
~~~

~~~bash
./extract-authors.sh > authors.txt
~~~

Fix the content. `git-all-fast-export` (and as far as I know all the other tools too), expect
a format `svn-user-name = committer name <commiter-email@example.com>`. It's really easy. If you don't know
each and every name, it doesn't matter.

Step 3: Export
===

The ugliest part is creating the "rules"-file. `svn-all-fast-export` expects a rules file, that contains
rules on how to map paths in SVN to git branches and tags. For further options, see
[Gitorius samples|https://gitorious.org/svn2git/mgedmin-svn2git/source/70b3618fa06355684616cd1611cad7cae6172cfb:samples]

~~~
create repository MyProject
end repository

match /MyProject/trunk/
  repository MyProject
  branch master
end match

match /MyProject/branches/([^/]+)/
  repository MyProject
  branch \1
end match

match /MyProject/tags/([^/]+)/
  repository MyProject
  branch tag/\1
end match

match /MyProject/tags/([^/]+)/
  repository MyProject
  branch refs/tags/\1
end match

match /
  # ignore everything we don't know (remove/comment this to find missing mappings)
end match
~~~

As you can see every mapping is prefixed with "MyProject", because this tool isn't able
to strip the project path itself.

~~~bash
svn-all-fast-export --identity-map=authors.txt --rules=my-rules.rules /path/to/local/svn
~~~

If everything worked fine, you know should have the bare git repository in a subfolder named
`MyProject`. Theoretically you are finished now.

Step 4: Cleanup
===
First we simply drop all already merged branches. If they are already merged, you don't need
them anymore and you can re-create the branch, when you need it again.

~~~bash
git branch -d `git branch --merged`
~~~



[BFG here]

~~~bash
git reflog expire --expire=now
git gc --aggressive --prune=now
~~~

One thing, that is a little bit annoying is, that for whatever reason `svn-all-fast-export` exports
the `svn:ignore` property into an `.svnignore` file. With some git "black magic" you can rename
the file in the _whole_ repository. I recommend it as the last step, because it is by far the most
time consuming step when handling with the git repository, but it is at least faster, when the
repository is already smaller.

~~~bash
git filter-branch --index-filter 'git ls-files -s \
    | sed "s-\(\t\"*\).svnignore-\1.gitignore-" \
    | GIT_INDEX_FILE=$GIT_INDEX_FILE.new git update-index --index-info && mv "$GIT_INDEX_FILE.new" "$GIT_INDEX_FILE"' HEAD
~~~

However, this isn't completely sufficient. `svn-all-fast-export` doesn't convert
`svn:ignore`-properties in subfolders. A good start to fix this manually (at least in the
`master`-branch) is `svn propget -R` and compare it with the `.gitignore`. Also it is
a good idea to review, if it really covers everything, that should get ignored. I wouldn't
spend too much time into this, because it only affects developing and it only affects
"living" branches.
