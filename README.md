# npm-git-lock

[![Circle CI](https://circleci.com/gh/bestander/npm-git-lock.svg?style=svg)](https://circleci.com/gh/bestander/npm-git-lock)

A CLI tool to lock all node_modules dependencies to a separate git repository.

Read a [post](https://medium.com/@bestander_nz/my-node-modules-are-in-git-again-4fb18f5671a) why you may need it.

# Update

I npm-git-lock was created a few years ago before Yarn and offline mirror feature: https://yarnpkg.com/blog/2016/11/24/offline-mirror/.

There is even a feature to [store built artifacts](https://github.com/yarnpkg/yarn/pull/5314), so I would suggest switching to Yarn as a more scalable solution.

## Features

- Tracks changes in package.json file
- When a change is found makes a clean install of all dependencies and commits and pushes node_modules to a remote repository
- Works independently from your npm workflow and may be used on a CI server only keeping your dev environment simpler

## How to use

```
sudo npm install -g npm-git-lock
cd [your work directory]  
npm-git-lock --repo [git@bitbucket.org:your/dedicated/node_modules/git/repository.git] -v

```

If you don't want to depend on NPM connectivity when installing this module, you can install directly from github:

```
sudo npm install -g https://raw.githubusercontent.com/bestander/npm-git-lock/master/npm-git-lock-latest.tgz
```
- Beware of possible breaking changes in the future, if you seek stability, obtain a link to a particular commit with the
 .tgz file on GitHub.

### Options:

    --verbose                 [-v] Print progress log messages
    --repo                    Git URL to repository with node_modules content  [required]
    --cross-platform          Run in cross-platform mode (npm 3 only)
    --incremental-install     Keep previous modules instead of always performing a fresh npm install (npm 3 only)
    --production              Runs npm install with production flag
    --check-all-json-elements Sha-1 calculated from all elements in package.json instead of only dependencies and devDependencies

`npm-git-lock` works with both npm 2 and 3, although the options `--cross-platform` and `--incremental-install` are only supported on npm 3.


## Why you need it

You need it to get reliable and reproducible builds of your Node.js/io.js projects.  

### [Shrinkwrapping](https://docs.npmjs.com/cli/shrinkwrap)
is the recommended option to "lock down" dependency tree of your application.  
I have been using it throughout 2014 and there are too many inconveniences that accompany this technique:  
1. Dependency on npm servers availability at every CI build. NPM [availability](http://status.npmjs.org/) is quite good in 2015 [watch a good talk](http://nodesummit.com/media/node-js-at-scale/) but you don't want to have another moving part when doing an urgent production build.  
2. Managing of npm-shrinkwrap.json is not straightforward as of npm@2.4. It is promising to improve though.  
3. Even though npm does not allow force updating packages without changing the version, packages can still be removed from the repository and you don't want to find that out when doing a production deployment.  
4. There are lots of other complex variable things about shrinkwrapping like optional dependencies and the coming changes in npm@3.0 like flat node_modules folder structure.   


### Committing packages
to your source version control system was recommended before shrinkwrapping but it is not anymore.    
Nonetheless I think it is a more reliable option though with a few annoying details:  
1. A change in any dependency can generate a humongous commit diff which may get your Pull Requests unreadable  
2. node_modules often contains binary dependencies which are platform specific and large and don't play well across dev and CI environments    

`npm-git-lock` is like committing your dependencies to git but without the above disadvantages.

## How it works

The algorithm is simple:  
1. Check if node_modules folder is present in the working directory  
2. If node_modules exists check if there is a separate git repository in it  
3. Calculate sha1 hash from package.json in base64 format  
4. If remote repo from [2] has a commit tagged with sha1 from [3] then check it out clean, no `npm install` is required
5. Otherwise remove everything from node_modules (unless `--incremental-install` is set, in which case only uncommitted changes will be stashed away), do a clean `npm install`, commit, tag with sha1 from [3] and push to remote repo
6. Next time you build with the same package.json, it is guaranteed that you get node_modules from the first run  

After this you end up with a reliable and reproducible source controlled node_modules folder.      
If there is any change in package.json, a fresh `npm install` will be done once.    
If there is no change, npm command is not touched and your CI build is fast.  

### Cross-platform mode

When `npm-git-lock` is run with the `--cross-platform` option, it does not commit "platform-specific" build artifacts into the remote repository. Instead, it builds them using `npm rebuild` when checking out the repository (step 4) or when doing a clean `npm install` (step 5). Platform-specific files are taken to be those files that are generated by build scripts.

Inspired by [this post](https://medium.com/@g_syner/for-the-most-part-i-really-like-your-solution-664c8248ec30#.4ekcegbww), this is how step 5 is modified in cross-platform mode:

1. Run `npm install --ignore-scripts` to prevent platform-specific compilation of any files.
2. Run `git add .` to capture the current "clean" cross-platform state.
3. Run `npm rebuild` to create any platform-specific files.
4. Run `git status --untracked-files=all` to list all files that have been generated in the previous step. Add these files to `.gitignore`.

`--cross-platform` is only supported on `npm` version >= 3, since npm 2 doesn't run custom install scripts as it should during `npm rebuild` (cf. [this CI failure](https://circleci.com/gh/bestander/npm-git-lock/11)).

### Incremental installs

By default, `npm-git-lock` will perform a completely fresh `npm install` whenever there is any change to package.json (i.e., there is no commit in the node_modules repository tagged with the sha1 of package.json). However, that might not always be desired, since all dependencies might change (as long as their version is still within the range specified in package.json).

To get a behavior more similar to `npm shrinkwrap`, you can use the option `--incremental-install`. When installing modules, it will reuse modules that have already been committed to the node_modules repository and only run `npm install` "on top of them".

A potential caveat is that modules are always fetched from the latest state (the master branch) of the node_modules repository. If there are dependencies introduced in a previous commit but not on the latest master HEAD, they will be freshly installed.

*Example*: In your project, you work on a branch *A* that declares a new dependency *depA* in package.json. In parallel, you have a branch *B* declaring a new dependency *depB*. Then you merge them both back into master. Assuming you run `npm-git-lock` in each step, the history of your central node_modules repository will look like this (from recent to older):

 * [commit3, master HEAD] *depA* + *depB*
 * [commit2] *depB*
 * [commit1] *depA*

Note that when running `npm-git-lock` after the merge (to produce commit3), only *depB* was fetched from the previous state. For *depA*, a fresh install was performed.


## Amazing features  

With this package you get:

1. Minimum dependency on npm servers availability for repeated builds which is very common for CI systems.
2. No noise in your main project Pull Requests, all packages are committed to a separate git repository that does not need to be reviewed or maintained.
3. If the separate git repository for packages gets too large and slows down your builds after a few years, you can just create a new one, saving the old one for patches if you need.
4. Using it does not interfere with the recommended npm workflow, you can use it only on your CI system with no side effects for your dev environment or mix it with shrinkwrapping.
5. You can have different node_modules repositories for different OS. Your CI is likely to be linux while your dev machines may be mac or windows. You can set up 3 repositories for them and use them independently.  
6. And it is blazing fast.

## Troubleshoot

If you see this kind of error in your CI:

```
Cloning into 'node_modules'...
done.
*** Please tell me who you are.
Run
  git config --global user.email "you@example.com"
  git config --global user.name "Your Name"
to set your account's default identity.
Omit --global to set the identity only in this repository.
fatal: empty ident name (for <travis@testing-worker-linux-docker-2b1f3404-3362-linux-9.prod.travis-ci.org>) not allowed
```

You need to configure `user.email` and `user.name` in the environment as shown in the error message.  
Just add those two commands before `npm-git-lock` call.

If you see this kind of error:

```
fatal: bad revision 'HEAD'
fatal: bad revision 'HEAD'
fatal: Needed a single revision
You do not have the initial commit yet
```

You need to commit and push to the repote repository at least once before using `npm-git-lock`

If you see this kind of error:

```
Git command 'tag -l --points-at HEAD' failed:
error: unknown option `points-at'
```

or

```
Git command 'stash save --include-untracked' failed:
error: unknown option for 'stash save': --include-untracked
```

You need to upgrade `git` to version 1.7.10+.

## Contribution

Please give me your feedback and send Pull Requests.  
Unit tests rely on ```require(`child_process`).execSync``` command that works in node 0.11+.  

## Future plans (up for grabs)

- Replace .es6 extension with .js
- Switch to [shelljs](https://github.com/shelljs/shelljs) from promises API. Promises are still too heavy for such a file oriented CLI tool

## Change Log

### [3.6.0](https://github.com/bestander/npm-git-lock/releases/tag/3.6.0) - 2018-02-20
- [Feature](https://github.com/bestander/npm-git-lock/pull/36) fixed scoped packages for --cross-platform flag

### [3.5.0](https://github.com/bestander/npm-git-lock/releases/tag/3.5.0) - 2016-06-30
- [Feature](https://github.com/bestander/npm-git-lock/pull/32) support --check-all-json-elements

### [3.3.0](https://github.com/bestander/npm-git-lock/releases/tag/3.3.0) - 2016-04-26
- [Feature](https://github.com/bestander/npm-git-lock/pull/25) support --production

### [3.2.1](https://github.com/bestander/npm-git-lock/releases/tag/3.2.1) - 2016-04-14
- [Fixed](https://github.com/bestander/npm-git-lock/pull/24) support for Node 0.12

### [3.2.0](https://github.com/bestander/npm-git-lock/releases/tag/3.2.0) - 2016-03-24
- [Feature](https://github.com/bestander/npm-git-lock/pull/21) run `preinstall` and `postinstall` scripts even in `--cross-platform` mode

### [3.1.1](https://github.com/bestander/npm-git-lock/releases/tag/3.1.1) - 2016-03-17
- [Fixed](https://github.com/bestander/npm-git-lock/pull/19) `loglevel` argument for npm commands

### [3.0.0](https://github.com/bestander/npm-git-lock/releases/tag/3.0.0) - 2016-02-18
- The hashing algorithm has [changed](https://github.com/sergiu-paraschiv/npm-git-lock/commit/abad012a6d1465ce79879e95a1af725134193ff5) due to a bug that caused different hashes to be generated on different platforms. This means hashes generated by 3.0.0+ are not compatible with older versions. Make sure you use the same version in all your environments! (`git install -g npm-git-lock@x.y.z` is your friend)


## License MIT
