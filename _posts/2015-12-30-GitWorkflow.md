---
layout: post
title: A Thorough Git-GitHub Workflow Manual
---

###Pick Or Perish
When working with Git and GitHub in a team project, everyone should be on the same page regarding the workflow that you as a team are going to pick and work with. Not only does it ensure that your repo stays in an organized state, but it also helps everyone to have a better time contributing to the project by not having to fight with version control more often than necessary. Also, you really do not want to leave the state of the team's repo to everyone's particular opinion on how they think git should be used do you?

But having a workflow is obviously not the whole deal. Everyone also has to know and remember the workflow. After the second or third time in one of my projects where at least one person had either not listened when we talked about the workflow or had just simply forgotten some of the details, I quickly decided to put together a manual on the workflow. A manual which everyone should be able to reference in the case of abrupt memory loss or just simply to look at the specific commands. Since it proofed to be extremely useful to have that manual I decided to share it here with everyone on the intertubes.

And just to quickly mention it GitHub could obviously also be exchanged for some other hosting platform that offers similar features, but for this article I will focus on GitHub.

###The Workflow Spectrum
When deciding on a workflow, there are many variables that you have to take into account and you have to decide on one depending on the use case. Whatever workflow you choose you will always have to make some tradeoffs. For me, the two deciding factors with the highest priority are overhead and giving team members a playground to try things out. That is at least in the settings that I have worked in with small teams and a relatively wide variance in knowledge and skill. On one end of this spectrum, very lightweight but maybe a little bit "dangerous" is something like [GitHub flow](http://scottchacon.com/2011/08/31/github-flow.html) on the other end something like the "[standardized multi-user workflow for the Oracle Developer Cloud Service](http://www.ateam-oracle.com/developercloudserviceworkflow/)". The workflow that I will talk about is somewhere in between and is influenced by other workflows, which I also mention below this post. 

Let me elaborate a little bit on the two points I just mentioned. In my opinion, too much overhead in a workflow breaks the workflow. Too much overhead leads to team members who are scared of or, at least, reluctant to pushing code, which makes them not push code, which is obviously not very good. The workflow should not make anyone avoid contributing! The second one I mentioned is that everyone should have their own playground. What that means is that everyone should have their own area(in this case fork) where they can try things out, make mistakes, learn from them and understand Git better without being a huge pain for everyone else in the team. If everyone in the team is extremely familiar with git, but which is oftentimes not achievable from the beginning, then something like GitHub flow is a good option as well since it gives even more transparency to what everyone is doing. But it has its drawbacks since minor mistakes might quickly break the teams repo. So make those tradeoffs and decide on one of them.

Following these two principles: Everyone has their own fork to leave enough space to try things out. And there is not too much overhead in order to get your code onto the master branch.

Without further ado let's jump into the manual.

###The Workflow
There are at least three locations where your code will be stored. That is firstly the organisations repo, which is the central hub for all the code and also the repo from which you deploy. We call that one upstream. Secondly, there is your fork, which is your own version of the repository where you can store all the code which you are currently working on, but which is not yet ready for the central repo. We call that one origin. Both of these are on GitHub. The third place is your local machine, where you will also have a copy of the repo. The latter two are your playground. Here you can create whatever branches you want, play around with the code and push and pull in whatever way you want.

All your development happens on your local machine. When you push you only push to origin, which is your fork. When you pull you normally only do that from upstream, even though in some cases you will also pull from one of you team mates forks or your own fork, but that is beyond the scope of this tutorial. From your fork, you make pull requests which are then merged into upstream(the central repo).

There are two main branches in the organisations repo, `master` and `deploy`. All pull requests should be made to the `master` branch. Once a certain milestone has been reached, `master` will be merged into `deploy` for deployment. All the commands that follow should be run from your CLI. Okay enough talking let's walk through it.

###Step by Step

####1. **Fork the repo**
  Example URL: *https://github.com/organisation-name/repo-name*

####2. **Clone the fork to your local computer**
  {% highlight sh %}
    $ git clone http://github.com/your-user-name/repo-name.git
  {% endhighlight %}
  
  Your repo is now remote `origin`. Check it if you want by doing `$ git remote -v`.
  
####3. **Set your upstream as the team's repo**
  {% highlight sh %}
    $ git remote add upstream http://github.com/organization-name/repo-name.git
  {% endhighlight %} 
  
####4. **Create a new branch**
  Make sure you have the newest code at this point.
 
  {% highlight sh %}
    $ git checkout master
    $ git pull --rebase upstream master
  {% endhighlight %}

  Then create a new branch.
  {% highlight sh %}
    $ git checkout -b newBranchName
  {% endhighlight %}

  Keep the branch names short but descriptive. 

####5. **Hack away**
  {% highlight sh %}
    # made changes to code
    $ git status
    $ git add <filename> # or git add .
    $ git commit
  {% endhighlight %}

  Commit often. Break down your task into separate mini-tasks and commit each time. If you realize you made too many commits in the end you can and should squash.

  Check out this [Commit Guide](https://github.com/hacksquare/Challengr/blob/master/docs/COMMIT-MESSAGES.md) for an example on how to structure commit messages.

####6. **Prepare your code for a pull request**
  - Check that the code does not break anything.
  - Read through the code and add comments if necessary.
  - Fix errors flagged by JSHint.
  - Check if code conforms with the style guide of the repo you are working on.
  - Prettify all your code (check [TOOLS.md](https://github.com/hacksquare/Challengr/blob/master/docs/TOOLS.md)) if that is something you do in your team.
  - squash your commits if necessary.

####7. **Checkout master and pull from upstream**
  {% highlight sh %}
    $ git checkout master
    $ git pull --rebase upstream master
  {% endhighlight %}

####8. **Checkout your branch and rebase master**
  {% highlight sh %}
    $ git checkout yourBranch
    $ git rebase master
  {% endhighlight %}

####9. **Fix all merge conflicts**
  {% highlight sh %}
    # fix merge conflicts
    $ git rebase continue
    # repeat until done
  {% endhighlight %}

####10. **Checkout master and merge your branch**
  {% highlight sh %}
    $ git checkout master
    $ git merge yourBranch
  {% endhighlight %}

####11. **Check your code again**
  And check it again. And make sure it does not break anything.

####12. **Push to your fork**
  {% highlight sh %}
    $ git push origin master
  {% endhighlight %}

####13. **Make pull request on github.com**
  Title the request so the reviewer can see what the code you wrote does.
  
####14. **Let someone check out your pull request**
  If everything is fine the person will merge it, otherwise, he or she will let you know what needs to be fixed and you can discuss it. NEVER MERGE YOUR OWN PULL REQUESTS!

####15. **Repeat cycle from step 4**

####Extra. **Squashing**
  Q: But what if I have 3 commits that are really similar?
  A: Squash them.

  {% highlight sh %}
    # check out your recent commits
    $ git log --oneline
    # choose how many you want to squash
    $ git rebase -i HEAD~3
  {% endhighlight %}

  Select 2 of the commits to be squashed ('squash'), and 1 commit to receive those changes ('pick').
  
  Read more about squashing [here](https://git-scm.com/book/en/v2/Git-Tools-Rewriting-History).

###Perks
An addition that could be made to the workflow would be for everyone to push the branch or branches they are currently working on to the organisations repo for everyone to see in addition to having them on their personal fork. This especially makes sense when your team gets bigger and it is not always clear what everyone is working on. If you want more transparency or want to use GitHub as the main communication tool you should think about making this addition. Just make sure everyone is cleaning up after themselves and they are deleting the branches that are merged. Otherwise, you will end up with 100 branches on your organisations repo and a repo that does not display the current state of the project. That most definitely misses the point.

###Wrapping Up
It is extremely important to choose a workflow which fits the requirements that your team has and help everyone understand and use that workflow correctly. That not only saves you a lot of work and time in the long run, however it also makes it much more enjoyable to contribute. I hope that this post helps you and your team get up to speed quickly with this workflow or has at least given you some inspiration to create a workflow that fits for you. There is so much more to Git than what I was able to mention in this post and I can only advise everyone to learn and use git with all its many features, it is truly an amazing tool.

If you have any feedback or find something that is not clear I am happy to hear from you.
You can reach me at <a href="mailto:arthur.mathies@googlemail.com">arthur.mathies@googlemail.com</a>.

Thanks for reading and until next time.

Arthur

#####Further Your Horizon

If you want to learn more about other workflows I highly recommend the following 3 resources, which do a great job in both breadth and detail:  
[https://www.atlassian.com/git/tutorials/comparing-workflows](https://www.atlassian.com/git/tutorials/comparing-workflows)  
[http://nvie.com/posts/a-successful-git-branching-model/](http://nvie.com/posts/a-successful-git-branching-model/)  
[http://scottchacon.com/2011/08/31/github-flow.html](http://scottchacon.com/2011/08/31/github-flow.html)  

Also here are two awesome resources on rebasing:  
[https://git-scm.com/book/en/v2/Git-Branching-Rebasing](https://git-scm.com/book/en/v2/Git-Branching-Rebasing)  
[https://www.atlassian.com/git/tutorials/rewriting-history/git-rebase/](https://www.atlassian.com/git/tutorials/rewriting-history/git-rebase/)

Or if you just simply want to learn more about Git:  
[https://git-scm.com/book/en/v2](https://git-scm.com/book/en/v2)  
[http://gitimmersion.com/](http://gitimmersion.com/)  
[https://www.atlassian.com/git/](https://www.atlassian.com/git/)
