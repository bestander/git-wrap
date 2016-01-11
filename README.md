# npm-git-lock

[ ![Codeship Status for bestander/npm-git-lock](https://codeship.com/projects/80df76f0-c8be-0132-4536-627bbcd2f5ed/status?branch=master)](https://codeship.com/projects/75106)  

A CLI tool to lock all node_modules dependencies to a separate git repository.

Read a [post](https://medium.com/@bestander_nz/my-node-modules-are-in-git-again-4fb18f5671a) why you may need it.

## Features

- Tracks changes in package.json file
- When a change is found makes a clean install of all dependencies and commits and pushes node_modules to a remote repository
- Works independently from npm and can be used only on CI server keeping dev environment simpler

## How to use

```
sudo npm install -g npm-git-lock
cd [your work directory]  
npm-git-lock --repo [git@bitbucket.org:your/dedicated/node_modules/git/repository.git] -v

```

If you don't want to depend on NPM connectivity when installing this module, you can install directly from github:

```
sudo npm install -g https://raw.githubusercontent.com/bestander/npm-git-lock/master/npm-git-lock-2.1.7.tgz
```


### Options:

  --verbose  [-v] Print progress log messages  
  --repo     git url to repository with node_modules content  [required]  


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
4. If remote repo from [2] has a commit tagged with sha1 from [3] then check it out clean, no npm install is required  
5. Otherwise remove everything from node_modules, do a clean npm install, commit, tag with sha1 from [3] and push to remote repo  
6. Next time you build with the same package.json, it is guaranteed that you get node_modules from the first run  

After this you end up with a reliable and reproducible source controlled node_modules folder.      
If there is any change in package.json, a fresh `npm install` will be done once.    
If there is no change, npm command is not touched and your CI build is fast.  

## Amazing features  

With this package you get:  
1. Minimum dependency on npm servers availability for repeated builds which is very common for CI systems  
2. No noise in your main project Pull Requests, all packages are committed to a separate git repository that does not need to be reviewed or maintained  
3. If the separate git repository for packages gets too large and slows down your builds after a few years, you can just create a new one, saving the old one for patches if you need  
4. Using it does not interfere with the recommended npm workflow, you can use it only on your CI system with no side effects for your dev environment or mix it with shrinkwrapping  
5. You can have different node_modules repositories for different OS. Your CI is likely to be linux while your dev machines may be mac or windows. You can set up 3 repositories for them and use them independently.  
6. And it is blazing fast  

## Troubleshoot

If you see this kind of error in your CI.  

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

## Contribution

Please give me your feedback and send Pull Requests.  
Unit tests rely on `require(`child_process`).execSync` command that works in node 0.11+.  

## License MIT

