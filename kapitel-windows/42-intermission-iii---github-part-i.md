# 42 Intermission III - Github Part I

What if we chuked our laptop into the sea? Then everything we had been working on would be lost. This won't do.
We could use an external harddrive or store backups of our project on a cloud service like Dropbox, and for a solo-made game that, honestly, could work. But on larger or more serious projects we can levarage the Git ecosystem to keep our project saved on the cloud, up-to-date and synced across multiple computers.
Github is the platform were your project is hosted, using the git architecture. We use a series of commands to let Git know which files we want to push aka upload to our Github repository .
A repository is the online storage of our files, as well as their changes in a timeline.
When we push a file to Github , if it was the first time we did so, we send the entire file. From this point forward, when we push our file we only push the latest changes, meaning that the amount (in bytes) being uploaded will be far smaller than if we had to reupload the entire thing each time. Git tracks our changes for us.
A bundle of changed files are called a commit . We give each commit a name and a non-mandatory description to help us (and our team) know what has changed. These commits then live as a timeline of changes, allowing us to revert back to an old version of our project if we would like.
If someone on our team has pushed a commit to our repository on Github using Git then we in turn can fetch the commits on github that are not yet synced to our computer. A fetch checks the difference between our machine and our repository then allowing us to pull aka download those commits .
When we have changes we commit them then push them. When we want to download the latest changes from github then we fetch them to see if any existed, then pull them into our machine.
We can use Git in one of two ways

1. We can use powershell to send commands to Git directly
2. We can use a software like Github Desktop to do the same stuff, but with some nice helpful buttons instead of code.

https://desktop.github.com/download/
Besides the Github Desktop client we will also need an account on github.com.
> [!NOTE]
> This account will be your portfolio, this if maintained nicely will be a huge asset for you when applying for internships and work. So please pick a sensible account name.

Once our account is set up we can log in to Github Desktop.
Now we can use file->new repository to start working on a new project. Or if we have the URL to a github project that we've been invited to collaborate on we can clone that repository from file->clone repository
Once we have our repository locally we can start commiting changes and pushing and pulling those commits to and from Github .
For a more in-depth look, check out the documentation: https://docs.github.com/en/desktop
With this we can do the very basics in Github.
Later you will learn about branches and pull requests and merge conflicts .