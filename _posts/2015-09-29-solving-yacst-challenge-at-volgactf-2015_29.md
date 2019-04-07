---
layout: post
title: Solving the YACST challenge at VolgaCTF 2015
tags: 
- CTF
- VolgaCTF
---

In this post I will walk you through my solution to the Yet Again Captcha Solver Task. I found the task quite enjoyable to solve as I haven't been working too much with sound files in this way.

So the name of the problem indicates that you have to solve a captcha of some sort. The task statement contains a link to a simple website with one link that downloads a .wav sound file and an input field in which to enter the solved captcha. Simple enough. You do have to solve the captcha 5 times in a row though. And when you try to do this manually, it only says `too sloooow`. So we need to automate the process.

![](/images/2015/05/1430660916_ae2cf9f1d5fc44409c35654e9129cf19.png)

So the .wav file contains a text-to-speech bot that reads a series of six numbers (0-9) with some interval of silence in between each number. The file is randomly generated each time we click the link. So we have to find a way to translate the sound for each number into a number that a program can submit to the website. So how do we do this?

I started out by writing a simple bash script that downloads a few .wav files and stores them to disk.

{% highlight bash %}
for i in `seq 1 5`;
do
    wget http://yacst.2015.volgactf.ru/captcha -O "captcha$i";
done
{% endhighlight %}

After looking at the .wav files, actually the file format is Resource Interchangeable File Format (RIFF), I decided that a possible way or step in solving the problem was to break the file down to a multiple parts, each containing a one of the numbers. I had no experience with working with .wav files so this took some Googling to figure out, but I found the `sox` program for Linux which seems to be a very common way to work with sound files from the command line. This program is comprehensive and the manual is long and a bit complex at first sight so I looked for examples of my problem online. I ended up with a command like this:

    sox "captcha$i" "captcha$i.wav" silence 1 0.10 0.1% 1 0.15 0.1% : newfile : restart

This command takes a file called e.g. captcha1and splits the file where it finds a period of silence. The `0.10` and `0.1%` values are the duration and threshold of silence respectively. I had to tweak these values a little to get good results as the files sometimes were split so that two numbers got included in a split file instead of a single number. These  values gave good results for this  challenge.

So now we have the ability to split files so that each part contains a single number from 0-9. That's great, but we still need to convert from a sound file to a integer. The way I solved this is by simply looking at the size of the split files that were produced. I ran the download and split routine several times and looked several split files with the exact same file size. This showed that files with the same size tended to have the number contained. A nice thing to know here is that `sox`comes with a `play` command that can be used to play a .wav file (and many other formats) from the command line. This made this process easier.

So now we know that a file containing for instance the voice recording of the number 0 tend to have a size of 5152 bytes. Which means we can write a program that translates between size and number. I was unsure if this was gonna be precise enough because I noticed some numbers didn't always have the same filesize. But, we only need 5 in a row once, so it should be good enough. Since Python is my go to scripting language when I have to do more than throw together Linux commands so that's what I went with. The first part of the code is just a dictionary translating between filesize and integer. The full code can be found [here](https://bitbucket.org/hland/volgactf2015)

{% highlight python %}
    t = {
    '5152': 0,
    '4228': 1,
    '3988': 2,
    '3650': 3,
    '4730': 4,
    '5950': 5,
    '3998': 6,
    '3748': 7,
    '3334': 8,
    '4476': 9,

# Extra data
    '4832': 5,
    '3236': 8,
    '4474': 9,
    '3996': 6,
    '3498': 4,
    '4036': 6,
    '4614': 7,
    '5602': 9,
    '3320': 8,
}
{% endhighlight %}

The rest of the Python code is made to be called from the Bash script with a folder path and the captcha number to solve (e.g.1 for captcha #1). The code translates the captcha and submits it using a POST request to the endpoint found on the simple challenge website. The website uses a `JSESSIONID` cookie to track the user submitting the captchas, so it is important that we use wgets flags to store the initial cookie that we get, the first time we download a .wav file, and continue using the same cookie when posting the solved captchas. Use the following flags for the first download:

   wget --keep-session-cookies --save-cookies cookies.txt http://yacst.2015.volgactf.ru -O captcha1

And the load the cookies for the next download using:

    wget --load-cookies cookies.txt http://yacst.2015.volgactf.ru/captcha -O "captcha2";

and so on. So this is how I solved the challenge for 200 points at the VolgaCTF. This task was beginner / intermediate difficulty at this CTF as the highest scored task was 600 points.

![](/images/2015/05/1430666428_a8792bfeae064509ae510f273490b0a6.png)

