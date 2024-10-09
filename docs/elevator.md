---
description: Learn how to solve the challenges from OSUSEC's second 2024-2025 school year meeting!
status: new
---

# 10/07/24 - Elevator

## Challenge Description
> welcome to the elevator!
>
> each user has a command line challenge - complete each one to gain credentials for the 
> next user

Tonight's CTF challenge is focused on using the Linux command line, so these challenges 
all involve using various Linux commands to extract credentials for the next level. I'll 
be redacting all passwords found since (a) they're basically mini flags and (b) the 
infrastructure is still up.

## Level 0
To begin, let's log into the challenge server and take a look around!

```console
radiantBear@local:~$ ssh elevator_chal
welcome! you are authenticating to level 0.
level0:~$ ls
README  creds_level1
level0:~$ cat README 
Have you seen my cat?
Once you find credentials, log in with su --login usernamehere
```

Interesting hint in the README! Let's change directories into creds_level1 and see what's
there:

```console
level0:~$ cd creds_level1
level0:~/creds_level1$ ls
creds_level1.txt
```

Alright, that file looks promising! Using the hint from earlier, let's try using the Linux
command `#!sh cat` to print out the contents of the file `creds_level1`:

```console
level0:~/creds_level1$ cat creds_level1.txt 
next user: level1_30569
password: ******
```

And with that, we're on to Level 1!

## Level 1
Time to switch users and take another look around:

```console
level0:~$ su --login level1_30569

level1:~$ ls
8j2vf1hx  README  ls4b2v23

level1:~$ cat README 
99% of gamblers give up right before they find creds
hint: find? grep? linux gives u tons of tools to run around directories like you're a little monkey and they're trees 
```

Interesting! The creds file isn't immediately obvious, and the hint suggests that finding
it manually may require a fair bit of work... The filename last time included the phrase
`creds`, so let's see if we can find a file with the same format this time.

```console
level1:~$ find . -name '*creds*'
./8j2vf1hx/j12du08b/c01uacv4/s98hdugp/jczsp0vs/9pk4d02b/creds_level2.txt
```

Sweet! We can print the contents again using `cat`, and we're on to the next challenge:

```console
level1:~$ cat ./8j2vf1hx/j12du08b/c01uacv4/s98hdugp/jczsp0vs/9pk4d02b/creds_level2.txt
next user: level2_30569
password: ******

level1:~$ su --login level2_30569
```

## Level 2
Alright, let's take a look around again:

```console
level2:~$ ls
README  creds_level3.txt

level2:~$ cat README 
rocky raccoon checked into his room...
hint: grep.
```

Interesting hint... what do The Beatles have to do with tonight's challenge? Let's try 
printing the contents of `creds_level3.txt` and see what we're dealing with:

```console
level2:~$ cat creds_level3.txt
The Project Gutenberg eBook of The Bible, King James Version, Complete
    
This ebook is for the use of anyone anywhere in the United States and
most other parts of the world at no cost and with almost no restrictions
whatsoever. You may copy it, give it away or re-use it under the terms
of the Project Gutenberg License included with this ebook or online
at www.gutenberg.org. If you are not located in the United States,
you will have to check the laws of the country where you are located
before using this eBook.

...
```

Yikes, that's a lot of text! I just showed the first 10 lines here, but there's 3.4 
megabytes of text in that file! Manually scanning the file for the creds will take 
*forever*; let's see if we can use `grep` to speed things up:

```console
level2:~$ grep user creds_level3.txt; grep password creds_level3.txt 
next user: level3_30569
password: ******

level2:~$ su --login level3_30569
```

Easy peasy; on we go!

## Level 3
Rinse and repeat; let's look around again:

```console
level3:~$ ls
README  creds_level4.sh

level3:~$ cat README 
Can you get creds_level4.sh to run?
```

Yes, indeed we can. We could use `chmod` to make the file executable and then run it, but
to progress even more quickly we can just pass it to `bash` to handle running it for us:

```console
level3:~$ bash creds_level4.sh 
=====creds:=====
*** WARNING : deprecated key derivation used.
Using -iter or -pbkdf2 would be better.
next user: level4_30569 password: ******
================

su --login level4_30569
```

Looks like the user `level3_30569` hasn't run their code in a while, but on we go!

## Level 4
I don't even need to explain this first step anymore, right? :slight_smile:

```console
level4:~$ ls
README

level4:~$ cat README 
Have you seen my file?
hint: ls has a lot of options :)
```

Well, the simplest possibility here is that the creds file has been hidden with a `.` 
prefix. Let's see if showing hidden files will help us out here:

```console
level4:~$ ls -a
.  ..  .bash_logout  .bashrc  .hidden_creds_level5  .profile  README
```

Indeed it did! Let's follow this up and find the creds:

```console
level4:~$ cd .hidden_creds_level5
level4:~/.hidden_creds_level5$ ls -a
.  ..  .creds_level5.txt
level4:~/.hidden_creds_level5$ cat .creds_level5.txt 
next user: level5_30569
password: ******

level4:~/.hidden_creds_level5$ su --login level5_30569
```

Onwards and upwards!

## Level 5
```console
level5:~$ ls
README  'look in here'

level5:~$ cat README 
filename is weird!
can u read it anyway?
```

Ah, the joys of names with spaces! We should be set with a few `\` to escape the spaces
and tell our shell that they aren't the end of the filename, though:

```console
level5:~$ cd look\ in\ here

level5:~/look in here$ ls
'creds in here!'

level5:~/look in here$ cat creds\ in\ here\! 
next user: level6_30569
password: ******

level5:~/look in here$ su --login level6_30569
```

## Level 6
Woah, here's something new! Upon logging in, VIM is immediately opened! Some fancy 
shenanigans in the `.bashrc` file -- this one made me laugh! Let's exit VIM and see what 
else awaits us:

```console
:q

level6:~$ ls
creds_level7.txt

level6:~$ cat creds_level7.txt 
next user: level7_30569
password: ******

level6:~$ su --login level7_30569
```

Looks like that's all there was to this one- let's keep going!

!!! tip
    If you get a VIM error `E32: No file name` when trying to exit VIM, make sure you're 
    using `:q` and not `:wq`. The configuration in `.bashrc` opens VIM without specifying 
    a file, so there's nothing specified to write to if you use `:w`.

## Level 7
```console
level7:~$ ls
README

level7:~$ cat README 
What's in /flag.txt?
```

Wow, the flag?? Can it really be? Let's see what's in `/flag.txt`:

```console
level7:~$ cat /flag.txt 
osu{********************}
```

Indeed, we made it to the top level! Tonight's challenge was a great overview of some
common shell commands, perfect for beginners to Linux. For those with Linux experience,
it's a fun challenge to see how quickly you can figure out the author's intent and speed
through the levels.