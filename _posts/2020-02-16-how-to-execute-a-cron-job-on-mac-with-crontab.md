---
title: "How to Execute a Cron Job on Mac With Crontab"
excerpt: "Automate random stuff with Python and Crontab on your Macbook"
header: 
  image: "/assets/images/hand-watch-macbook.JPG"
categories:
  - Blog
tags:
  - cron
  - python
---
# **Prologue**

Before I write the steps below, I would recommend installing the latest version of Python with Homebrew. [Homebrew](https://brew.sh/) is a package manager just like [pip](https://pip.pypa.io/en/stable/) and [Anaconda](https://www.anaconda.com/) and frankly, I’m just using that in this example because I tried for hours to run a simple cron job to no avail.

I frequently got errors about not being able to open the file I wanted to execute or not finding the installed Python version. I won’t get into my errors in this post but you’re free to use any other Python installation if you like.

I uninstalled my Python 3.x versions I had installed (via pip and Conda) and started from scratch — uninstalling was also a hassle.

So, let’s say you’re on your Mac and have the default Python distribution, which I believe should be 2.7.10. Few to no developers are updating modules in this version so when you’re building projects, it’s best to have the latest version of Python installed.

I followed [Real Python’s guide](https://realpython.com/installing-python/#step-2-install-homebrew-part-2) to install Python 3 via Homebrew (a quick few steps really) so I will assume you have done the same going forward in this post.

# **Steps**

Open up your terminal command prompt on your Mac and navigate to the home directory by running `cd ~/`. For me, it is `Users/Nakul`.

We will be using Mac OS’s in-built crontab feature to write our cron jobs.

Type `crontab -e` and press Enter.

This should open up an empty file, this is where you will write your cron jobs. You can write the job to run a shell script or a Python script in this case.

Type `:q!` to exit the editor.

Assuming you followed the Homebrew guide above, double-check your Python installation path. You can do so by typing `which python3` in your terminal.

It should display `/usr/local/bin/python3` (which is an alias for the actual Homebrew installation location but that’s fine, we won’t go into that right now). You will include this path in your cron job syntax, which we’ll get into.

Before we write our cron job, we should have a script we want to run. I’ve already created a directory `/Documents/Python/cron` in my home directory and created a simple script called `cron_test.py`.

Note: This Python script needs to be executable so change the permissions on it to allow for that, I just ran a `chmod 777 cron_test.py`.

The `cron_test.py` script simply creates a directory with the current date and time as the name.

The script text is as follows if you want to use the same script:

{% gist /ddd68386fb165721d618bf00895aaee9 %}


You might wonder what that first line, `#!/usr/bin/python3`, is. That line is referred to as a *shebang* and how the cron job will interpret the script so in this case, it will run the `cron_test.py` with Python (as if you executed Python `cron_test.py` in your terminal).

OK, so we have Python 3.x installed, we have a Python script, now let’s write the cron job to run this!

Let’s say we want to run this script every minute (mostly for the purpose of testing on your end so you can see the results without waiting too long).

The cron job syntax is as follows:

```
*/1 * * * * cd ~/Documents/Python/cron && /usr/local/bin/python3 cron_test.py >> ~/Documents/Python/cron/cron.txt 2>&1
```

What does the above mean? Let’s analyze it quickly.

`/1 * * * *` simply means that the job will run every minute.

`cd ~/Documents/Python/cron && /usr/local/bin/python3` navigates to the directory where the script you want to execute is located and specifies to use Python instead of [Bash](https://www.gnu.org/software/bash/) to execute (since it is a Python script, also remember that we added the shebang within the script, I found that it needed to be in both places to work).

`cron_test.py` is the script filename.

`>> ~/Documents/Python/cron/cron.txt` specifies where to output the logs in case the execution of the job has any issues.

`2>&1` simply disables email because by default, the cron job will try to send an email but we don’t have an address specified.

Anyway, that should be enough to get you writing and testing your scripts, so go on!

Feel free to browse the web to learn more about cron jobs and crontab. I decided to write this for those who have a fundamental understanding of programming and wanted to write a cron job as soon as possible.

Hope this helped!