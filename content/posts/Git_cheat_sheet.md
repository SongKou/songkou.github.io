+++
title = 'Git_cheat_sheet'
date = 2021-12-25T01:31:18+08:00
draft = false
categories = ['Automation']
tags = ['Git']
+++
## Git cheat sheet

### Configure

Configure system/global level of current repository username, email

    git config --global user.name "Jerry Mouse"  
    git config --global user.email "jerry@yiibai.com"`

List git configs

    git config --list  
    git config -l`

### initialize

    git init

### Modify commit

    git add a.html  
    git commit -m "提交文件a.html"

-   `git revert`-   is about making a new commit that reverts the changes made by other commits.`git reset` is about updating your branch, moving the tip in order to add or remove commits from the branch. This operation changes the commit history. it can also be used to restore the index, overlapping with git restore.`git-restore`  is about restoring files in the working tree from either the index or another commit. This command does not update your branch. The command can also be used to restore files in the index from another commit.
    
Revert will reverse the changes from a commitment and is used mostly to undo faulty commits. This means that you can revert specific commits without affecting others. Revert works by making a new commit with the changes removed. You can revert any past commit. This can be the last commit,  `$ git revert HEAD`.

Or  `$ git revert -n master~5..master~2`  which will revert the changes done by commits from the fifth last commit in master (included) to the third last commit in master (included), but do not create any commit with the reverted changes. The revert only modifies the working tree and the index. This example came from  [https://git-scm.com/docs/git-revert](https://git-scm.com/docs/git-revert)  and helps show the different options associated with reverting.

Reset, in contrast to Revert, does not create a new commit, but effectively goes back in time to a state in the past and calls that current. This is really helpful if you need to take a new direction in your work and don’t mind discarding the latest commits.

There are a number of options to customize this approach, but the simplest is  `$  git  reset —-hard  [<commit`  which will discard all changes between the current HEAD and the commit indicated in the command. It’s worth noting that the commit is specified by the Index, which is the number you see when you call the log.
    

### Branch

    git checkout -b dev #create new branch
      
    git branch # check current branch
      
    git checkout master   # switch branches
      
    git  branch  -a # check remote all branches, white is have locally, red is doesn't have locally, only on remote
      
    git merge dev # merge branch
      
    git branch -d dev # delete branch 
      
    git branch -D branch_name  # force delete branch locally 
      
    git push origin branch_name # push branch to remote

### Resolve Conflict

1、when there is a conflict in a file

    <<<<<<< HEAD  
      
    Creating a new branch is quick & simple.  
      
    =======  
      
    Creating a new branch is quick AND simple.  
      
    >>>>>>> feature1

git use `<<<<<<<`，`=======`，`>>>>>>>` to mark the conflict part between you and other people in a file. 

in between `<<<<<<<`，`=======` is your code；`=======`，`>>>>>>>` is other people's code. 

if need to keep yours, just delete other people's



### Bug Branch

1、Stash the changes in a dirty working directory away

`git stash`

Use  `git stash`  when you want to record the current state of the working directory and the index, but want to go back to a clean working directory. The command saves your local modifications away and reverts the working directory to match the  `HEAD`  commit.

The modifications stashed away by this command can be listed with  `git stash list`, inspected with  `git stash show`, and restored (potentially on top of a different commit) with  `git stash apply`. Calling  `git stash`  without any arguments is equivalent to  `git stash push`. A stash is by default listed as "WIP on  _branchname_  …​", but you can give a more descriptive message on the command line when you create one.

The latest stash you created is stored in  `refs/stash`; older stashes are found in the reflog of this reference and can be named using the usual reflog syntax (e.g.  `stash@{0}`  is the most recently created stash,  `stash@{1}`  is the one before it,  `stash@{2.hours.ago}`  is also possible). Stashes may also be referenced by specifying just the stash index (e.g. the integer  `n`  is equivalent to  `stash@{n}`).

2、Restore stashed code

`git stash pop #delete it in stash cache`

or

    git stash apply  # restore stash, but doesn't delete in stash cache
    git stash drop # use this command to delete stash

> git stash list # check all stash list

3、clear stash space 

`git stash clear`

4、git stash pop vs git stash apply 

git stash pop stash@{id} command will delete the corresponding id from stash list,  on the other hand, git stash apply stash@{id}  will keep the stash id.

### Rollback version

1、rollback to previous version 

`git reset --hard HEAD`

2、rollback to designated version

`git reset --hard  [version id]`

3、check version id (local commit)

`git reflog`

4、check all version id and info (all commit：local commit + commit other people)

`git log`

-   • `git reset` move HEAD backwards, and `git revert` move HEAD forward, it's just new commit info is the reverse of  revert info，so it will "neutralize" the revert information. 
    
  

### revert change

1、revert change

`git  checkout -- a.html`

two conditions：

-  not yet `git add` , will change back to the same state as in the repository. 
    
-  execute `git add`, but not yet `git commit` it will change back to git add state. 
    

> if execute `git commit -m "*"`, then you can't use above command revert.

###  Rollback for pushed version 

1. Step 1：

`git reset --hard [version id] # rollback to designated version locally`

2. Step 2：

`git push -f origin dev # force push to remote`

### Sync with remote on deleted branch

`git fetch origin -p  # this is to clear branches that not in remote, so git branch -a won't pull branches that is deleted remotely already.`

delete local branches that doesn't exist in remote. 

`git fetch -p`

Check remote repository  info and local branch info

 `git remote show origin`

###  Git remote operation related

(1). clone

    git clone repository url 
    git clone <repository url> <local_dir>

(2). remote

    #Manage set of tracked repositories  
    git remote  

#### DESCRIPTION

Manage the set of repositories ("remotes") whose branches you track.

#### OPTIONS

-v
--verbose

Be a little more verbose and show remote url after name. For promisor remotes, also show which filter (`blob:none`  etc.) are configured. NOTE: This must be placed between  `remote`  and subcommand.

#### COMMANDS

With no arguments, shows a list of existing remotes. Several subcommands are available to perform operations on the remotes.

add

Add a remote named `<name>` for the repository at `<URL>`. The command  `git fetch <name>`  can then be used to create and update remote-tracking branches `<name>/<branch>`.

With  `-f`  option,  `git fetch <name>`  is run immediately after the remote information is set up.

With  `--tags`  option,  `git fetch <name>`  imports every tag from the remote repository.

With  `--no-tags`  option,  `git fetch <name>`  does not import tags from the remote repository.

By default, only tags on fetched branches are imported (see  [git-fetch[1]](https://git-scm.com/docs/git-fetch)).

With  `-t <branch>`  option, instead of the default glob refspec for the remote to track all branches under the  `refs/remotes/<name>/`  namespace, a refspec to track only  `<branch>`  is created. You can give more than one  `-t <branch>`  to track multiple branches without grabbing all branches.

With  `-m <master>`  option, a symbolic-ref  `refs/remotes/<name>/HEAD`  is set up to point at remote’s  `<master>`  branch. See also the set-head command.

When a fetch mirror is created with  `--mirror=fetch`, the refs will not be stored in the  _refs/remotes/_  namespace, but rather everything in  _refs/_  on the remote will be directly mirrored into  _refs/_  in the local repository. This option only makes sense in bare repositories, because a fetch would overwrite any local commits.

When a push mirror is created with  `--mirror=push`, then  `git push`  will always behave as if  `--mirror`  was passed.

rename

Rename the remote named `<old>` to `<new>`. All remote-tracking branches and configuration settings for the remote are updated.

In case `<old>` and `<new>` are the same, and `<old>` is a file under  `$GIT_DIR/remotes`  or  `$GIT_DIR/branches`, the remote is converted to the configuration file format.

remove
rm

Remove the remote named `<name>`. All remote-tracking branches and configuration settings for the remote are removed.

set-head

Sets or deletes the default branch (i.e. the target of the symbolic-ref  `refs/remotes/<name>/HEAD`) for the named remote. Having a default branch for a remote is not required, but allows the name of the remote to be specified in lieu of a specific branch. For example, if the default branch for  `origin`  is set to  `master`, then  `origin`  may be specified wherever you would normally specify  `origin/master`.

With  `-d`  or  `--delete`, the symbolic ref  `refs/remotes/<name>/HEAD`  is deleted.

With  `-a`  or  `--auto`, the remote is queried to determine its  `HEAD`, then the symbolic-ref  `refs/remotes/<name>/HEAD`  is set to the same branch. e.g., if the remote  `HEAD`  is pointed at  `next`,  `git remote set-head origin -a`  will set the symbolic-ref  `refs/remotes/origin/HEAD`  to  `refs/remotes/origin/next`. This will only work if  `refs/remotes/origin/next`  already exists; if not it must be fetched first.

Use  `<branch>`  to set the symbolic-ref  `refs/remotes/<name>/HEAD`  explicitly. e.g.,  `git remote set-head origin master`  will set the symbolic-ref  `refs/remotes/origin/HEAD`  to  `refs/remotes/origin/master`. This will only work if  `refs/remotes/origin/master`  already exists; if not it must be fetched first.

set-branches

Changes the list of branches tracked by the named remote. This can be used to track a subset of the available remote branches after the initial setup for a remote.

The named branches will be interpreted as if specified with the  `-t`  option on the  `git remote add`  command line.

With  `--add`, instead of replacing the list of currently tracked branches, adds to that list.

get-url

Retrieves the URLs for a remote. Configurations for  `insteadOf`  and  `pushInsteadOf`  are expanded here. By default, only the first URL is listed.

With  `--push`, push URLs are queried rather than fetch URLs.

With  `--all`, all URLs for the remote will be listed.

set-url

Changes URLs for the remote. Sets first URL for remote `<name>` that matches regex `<oldurl>` (first URL if no `<oldurl>` is given) to `<newurl>`. If `<oldurl>` doesn’t match any URL, an error occurs and nothing is changed.

With  `--push`, push URLs are manipulated instead of fetch URLs.

With  `--add`, instead of changing existing URLs, new URL is added.

With  `--delete`, instead of changing existing URLs, all URLs matching regex `<URL>` are deleted for remote `<name>`. Trying to delete all non-push URLs is an error.

Note that the push URL and the fetch URL, even though they can be set differently, must still refer to the same place. What you pushed to the push URL should be what you would see if you immediately fetched from the fetch URL. If you are trying to fetch from one place (e.g. your upstream) and push to another (e.g. your publishing repository), use two separate remotes.

show

Gives some information about the remote `<name>`.

With  `-n`  option, the remote heads are not queried first with  `git ls-remote <name>`; cached information is used instead.

prune

Deletes stale references associated with `<name>`. By default, stale remote-tracking branches under `<name>` are deleted, but depending on global configuration and the configuration of the remote we might even prune local tags that haven’t been pushed there. Equivalent to  `git fetch --prune <name>`, except that no new references will be fetched.

See the PRUNING section of  [git-fetch[1]](https://git-scm.com/docs/git-fetch)  for what it’ll prune depending on various configuration.

With  `--dry-run`  option, report what branches would be pruned, but do not actually prune them.

update

Fetch updates for remotes or remote groups in the repository as defined by  `remotes.<group>`. If neither group nor remote is specified on the command line, the configuration parameter remotes.default will be used; if remotes.default is not defined, all remotes which do not have the configuration parameter  `remote.<name>.skipDefaultUpdate`  set to true will be updated. (See  [git-config[1]](https://git-scm.com/docs/git-config)).

With  `--prune`  option, run pruning against all the remotes that are updated.
  
##### check remote url
`git remote -v  `
  
##### check remote detailed info
`git remote show < host name> `
  
##### add remote host 
`git remote add <host name> <url>  `
  
#### delete remote host
`git remote rm <host name>  `
  
#### rename remote host name
`git remote rename <old_host_name> <new_host_name>`

(3). fetch

```
# get all branch update to local
git fetch <remote host name>
```

#### get certain branch update
```
git fetch <remote host name> <branch name>
```

#### get origin's master branch update
```
git fetch origin master
```

> the update that you get need to be get by "remote host name/branch
> name" , eg: origin host master branch, need to get via
> `origin/master`. Can use `git merge` or `git rebase` command to merge
> remote branch.

```
git merge origin/master
git rebase origin/master
```

(4). pull

```
# get origion host's next branch, merge with master branch on local
git pull origin next:master
```

##### if remote branch is to merge with current branch, then info after :  is not required.

```
git pull origin next
```

#### above command is equal to do a git fetch, then do git merge.
```
git fetch origin
git merge origin/next
```

####  merge use rebase mode
```
git pull --rebase <remote host name> <remote branch name>:<local branch name>
```

(5). push

>  push local master branch to remote origin host name, master branch.  if remote doesn't exist, it will recreate  `git push origin master`

#### Omitted local branch, same as 2nd one, delete origin host master branch
```
git push origin :master
git push origin --delete master
```

#### if current branch have track relationship with remote branch, then local and remote branch all not required.
```
git push origin
```

#### if current branch have just one tracking branch, then even host name is not required
```
git push
```

#### if current branch have tracking relationship with multiple host, can use -u to designate one default host, then in the future no need to put host info in git push.
```
git push -u origin master
```

#### Regardless there is a corresponding remote branch, push all local branch to remote host.
```
git push --all origin
```

##### force push
```
git push --force origin
```

#### git push won't push tag (tag), unless you use –tags option
```
git push origin --tags
```


