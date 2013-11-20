---
layout: post
title:  "How to transform a project from a huge subversion repository to git"
categories: subversion git
---

Multi-project subversion repositories sound convenient at the first glance: Everything
at one place and only one system to manage. If you ever consider to switch over to git
it is a bad idea. Usually transforming a subversion into a git repository is quite easy,
but that is only valid for repositories of a reasonable size. Multi-project repository
tend to grow quite big. The usual transformation via `git svn clone <svn-url>` extremely
slows down on huge repositories.

A short overview of what is required

* Time. It doesn't take weeks anymore, but it still requires some time (I got it working in
    around 4 hours at the end)
* A _fast_ storage. The whole process is extremely IO-intensive. A SSD works fine, even better
    --- if there is enough RAM available --- is a RAM-disk of around 4 to 8 GB.
* [svnrdump](http://svnbook.red-bean.com/en/1.7/svn.ref.svnrdump.c.dump.html). This is part
    of SVN 1.7.
* I used [svn-all-fast-export](http://dev.man-online.org/man1/svn-all-fast-export/) for
    the actual conversion (it's available in the ubuntu repository). `svn-fast-export` should
    work too, but I had issues with that.
* [BFG Repo-Cleaner](http://rtyley.github.io/bfg-repo-cleaner/) to get rid of large files
    and files containing sensitive data.
* Of course `svn` (`svnadmin`) and `git`.


Step 1: Create a local copy
====
It's a mess to work with the remote SVN all the time, so the first step is to clone the
remote repository onto the local machine. It's also much smaller, because here we already
only download the code we are actually interested int

```bash
svnrdump dump --revision 0:20000 http://example.com/path/to/svn/MyProject | gzip -9 > MyProject.1.svn.gz
svnrdump dump --revision --incremental 20001:40000 http://example.com/path/to/svn/MyProject | gzip -9 > MyProject.2.svn.gz
svnrdump dump --revision --incremental 40001:60000 http://example.com/path/to/svn/MyProject | gzip -9 > MyProject.3.svn.gz
# And so on
```

Hint: Execute this in parallel. HTTP is slow, so this may take the longest time. You can choose
chunk sizes at will, but is important, that the first one doesn't have the `--incremental`-flag,
whereas every subsequent call has. And why gzip? Actually the file size is not the problem, but
it can take some load from the disk.

```bash
svnadmin create MyProject
gunzip < MyProject.1.svn.gz | svnadmin load -q --force-uuid
gunzip < MyProject.2.svn.gz | svnadmin load -q --force-uuid
gunzip < MyProject.3.svn.gz | svnadmin load -q --force-uuid
```

Now you have a local copy of your repository only containing the code of `MyProject`. You
can easily update this one later with an additional `svnrdump`/`svmadmin load`.

Step 2: Prepare export
===

First difference occurs: SVN only tracks committers by an username, but git by an email-
address with an additional, optional name. You'll probably want both (for at least he
users you know). Create a script, paste the following code into it and make it executable.
Note, that you must change the path to your local SVN

```bash
#!/usr/bin/env bash
authors=$(svn log -q file:///path/to/local/svn | grep -e '^r' | awk 'BEGIN { FS = "|" } ; { print $2 }' | sort | uniq)
for author in ${authors}; do
  echo "${author} = NAME <USER@DOMAIN>";
done
```

Execute it

```bash
./extract-authors.sh > authors.txt
```

Fix the content. `git-all-fast-export` (and as far as I know all the other tools too), expect
a format `svn-user-name = arbitrary name <email@example.com>`. It's really easy. If you don't know
each and every name, it doesn't matter.

Step 3: Export
===

The ugliest part is creating the "rules"-file. `svn-all-fast-export` expects a rules file, that contains
rules on how to map paths in SVN to git branches and tags. And example

```
create repository MyProject
end repository

match /MyProject/trunk/
  repository SGFD
  branch master
end match

match /MyProject/branches/([^/]+)/
  repository SGFD
  branch \1
end match

match /MyProject/tags/([^/]+)/
  repository SGFD
  branch tag/\1
end match

match /
  # ignore everything we don't know (remove/comment this to find missing mappings)
end match

match /MyProject/tags/([^/]+)/
  repository SGFD
  branch refs/tags/\1
end match
```

As you can see every mapping is prefixed with "MyProject", because this tool isn't able
to strip the project path itself.

Now try.

```bash
svn-all-fast-export --identity-map=authors.txt --rules=my-rules.rules /path/to/local/svn
```

If everything worked fine, you know should have the bare git repository in a subfolder named
`MyProject`. Theoretically you are finished now.

Step 4: Cleanup
===
First we simply drop all already merged branches. If they are already merged, you don't need
them anymore and you can (and should) re-create the branch, when you need it again. If you
want to keep them anyway, just skip this step.

```bash
git branch -d `git branch --merged`
```



[BFG here]

```bash
git reflog expire --expire=now
git gc --aggressive --prune=now
```

One thing, that is a little bit annoying is, that for whatever reason `svn-all-fast-export` exports
the `svn:ignore` property into an `.svnignore` file. With some git "black magic" you can rename
the file in the _whole_ repository. I recommend it as the last step, because it is by far the most
time consuming step when handling with the git repository, but it is at least faster, when the
repository is already smaller.

```bash
git filter-branch --index-filter 'git ls-files -s \
    | sed "s-\(\t\"*\).svnignore-\1.gitignore-" \
    | GIT_INDEX_FILE=$GIT_INDEX_FILE.new git update-index --index-info && mv "$GIT_INDEX_FILE.new" "$GIT_INDEX_FILE"' HEAD
```

However, this isn't completely sufficient. `svn-all-fast-export` doesn't convert
`svn:ignore`-properties in subfolders. A good start to fix this manually (at least in the
`master`-branch) is `svn propget -R` and compare it with the `.gitignore`. Also it is
a good idea to review, if it really covers everything, that should get ignored. I wouldn't
spend too much time into this, because it only affects developing and it only affects
"living" branches.
