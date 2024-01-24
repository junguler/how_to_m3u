# how to m3u
here is a full workflow and explanation on why, what and how of scraping a radio website

<br>

## introduction, tools of trade
here is a list of command line programs i use for various tasks about scraping and beatifying data

`a linux shell, bash/zsh` is the environment in which everything takes place, if you are on linux or mac you are already set, on windows i recommend msys2, cygwin or wsl/2

`piping |` is the process of taking output from a program and sending it to another for further operations

`> data.txt` having this symbol at the end of your program that spits out data makes the data go to `data.txt`

`>> data.txt` unlike `>` this does not replace the existing data in `data.txt` if it already exists, it just adds to it

`for loop` is used to automate doing a repetitive task for a lot of items

`curl` we will use to download web pages containing the raw data we are interested in, in this case radio stream title and link

`htmlq` is a powerful program that takes data from html tags and beatify and clean the result for easy data extraction

`awk` is a very powerful program that i use to extract certain strings or rows of data

`grep` is a program used to get or exclude instances of an string or strings

`sed` is a program used to find certain strings and replace them with other strings or remove them

`github actions` gives you free access to free virtual computers to automate your tasks

<br>

## find a website to scrape from
pick a random stream and search it's titles in your search engine, add `listen to` prefix to it and search away, here is the site i'm going to scrape next [thenonstopradio.com](https://thenonstopradio.com)

make sure this website is not protected by cloudflare, google or other forms of captcha, scraping captcha protected websites is not hard but it's time consuming and might involve spending money on services specially designed for this task, in short it's a headache

<br>

## open a stream page
open a page containing the music your are looking for, lets see if the stream page is already embedded in the page's source or is it dynamically loaded in from another location

i've picked [edm_sessions_us](https://thenonstopradio.com/radio/edm_sessions_us), open the developers tools (inspect element) and go to the network tab, reload the page and it should start populating the list with various media, js and other stuff

none of these items is of our interest since the music is not playing yet, now hit the play button and see the stream link poping up as shown here

copy this link and now lets head to the source of this page and search for this exact stream link 

`https://s2.radio.co/s30844a0f4/listen`

`ctrl + f` brings up a very long line but our link has `data-audio-url=` prefix like this

`data-audio-url="https://s2.radio.co/s30844a0f4/listen"`

so now lets look for that instead, there are 21 instances of that string in this page and the first string has the link for our example `edm_sessions_us` radio station so take a mental note of this for the future

now we know this website is not captcha protected and has the stream link embedded, this is the perfect site for scraping, now lets actually get to work

<br>

## the main page, get all the categories
so we want everything, lets use the websites own categories, this websites has `genre` `country` `language` and `network` pages, so lets get started by first actually making a local copy of them to our computer

make a scrape folder and cd into it

```
curl -s https://thenonstopradio.com/genre > genre.html
curl -s https://thenonstopradio.com/country > country.html
curl -s https://thenonstopradio.com/language > language.html
curl -s https://thenonstopradio.com/network > network.html
```

now we can test things without putting extra load on the servers and slow things down for ourselves, the `-s` flags stands for silent, it tells `curl` to just do the things we want it to and don't talk to us about it

<br>

## find all the sub items for each category
lets find the naming scheme this websites uses, starting by piping the `genre.html` to `htmlq`

```
cat genre.html | htmlq -a href a
```

```
...
https://thenonstopradio.com/genre/mix
https://thenonstopradio.com/genre/mix
https://thenonstopradio.com/genre/ndw
https://thenonstopradio.com/genre/ndw
https://thenonstopradio.com/genre/zouk
https://thenonstopradio.com/genre/zouk
https://play.google.com/store/apps/details?id=com.liveradio.fmradio.radiotuner.radiostation.amradio
https://www.facebook.com/thenonstopradio
https://twitter.com/non_stopradio
https://www.instagram.com/thenonstopradio/
javascript:void(0);
...
```

`htmlq -a href a` tells `htmlq` to find all the links within the page for us

above is a snippet of what came out of that command, as you can see there are a bunch of stuff we don't need but there are a few lines with the `/genre/` string that look promising, and there are some duplicated lines too, lets `grep` for for every instance of `/genre/` and also remove duplicates

```
cat genre.html | htmlq -a href a | grep "/genre/" | sort | uniq
```

```
...
https://thenonstopradio.com/genre/spanish-music
https://thenonstopradio.com/genre/spanish-talk
https://thenonstopradio.com/genre/spiritual
https://thenonstopradio.com/genre/sports
https://thenonstopradio.com/genre/sports-talk
https://thenonstopradio.com/genre/sports-talk-&-news
https://thenonstopradio.com/genre/synthiepop
...
```

`grep` looks for all the instances of `/genre/`, `sort` does exactly what it sounds like and `uniq` removes duplicates

that's more like it, while we at it lets remove the repeated characters in each line since it's easier to just add that back in later than have to be present each and every time, you'll see what i mean later


```
cat genre.html | htmlq -a href a | grep "/genre/" | sort | uniq | awk -F '/' '{print $5}' > genres.txt
```

```
...
dance-hits
dance-pop
death-metal
deep-house
deutscher-hip-hop
disco
dj
...
```

slash `/` is the separator that `awk` uses here to split this piped out and we got the 4th item after an instance of slash which is set in awk by `{print $5}` , this data is also been copied to the `genres.txt` so we can use it later

finding the other categories is also easy since this website uses a similar naming scheme for all of them, just replace `/genre/` with `/country/` `/language/` `/network/` for other pages and change the raw html and output names accordingly

```
cat country.html | htmlq -a href a | grep "/country/" | sort | uniq | awk -F '/' '{print $5}' > country.txt
cat language.html | htmlq -a href a | grep "/language/" | sort | uniq | awk -F '/' '{print $5}' > language.txt
cat network.html | htmlq -a href a | grep "/network/" | sort | uniq | awk -F '/' '{print $5}' > network.txt
```

<br>

## get the addresses of all radio streams
now that we have `genre.txt` `country.txt` `language.txt` `network.txt` we can start finding the stream names, lets have a look at the `trance` page of the `genre` section and see how it is set up

this is the first two pages of this genre

https://thenonstopradio.com/genre/trance

https://thenonstopradio.com/genre/trance/2

the second page has an extra `/2`, good to know, we will use this in our future `for loop`, now lets make a local copy of the first page to see how it's set up

```
curl -s https://thenonstopradio.com/genre/trance > trance.html
```
```
cat trance.html
```
```
javascript:void(0)
https://thenonstopradio.com/radio/gigacraft_de
https://thenonstopradio.com/radio/gigacraft_de
javascript:void(0)
javascript:void(0);
```

hmmm, looks very similar, has extra fluff we don't want, has duplicates and the stream pages has a `/radio/` in it, se lets clean this output and get something useful out of it

```
cat trance.html | htmlq -a href a | grep "/radio/" | uniq | awk -F '/' '{print $5}'
```
```
...
boomundspeed_de
na_radio_de
danceradio_nrw_de
bn_radio_de
music_hall_de
party_radio_station_de
...
```

looks very similar and we got what we wanted, so far so good

the issue is there are many genres and some of them have many pages, what we need is a `for loop` that goes thru and do everything for us, so lets have a simple example of how a for loop looks

```
for i in "apple" "orange" ; do echo $i ; done
```
```
apple
orange
```

what happened? we assigned `apple` and `orange` to `i`, all `echo` does is print the things you asked of it, this for loop went thru apple and orange and inserted them one by one to `echo`

so lets do something similar but for our `trance` genre of this website, there are a total of 3 trance pages in this website and we know the second and third page are assigned by `/2` and `/3` and the first page doesn't have anything extra, so lets put it together

```
for i in "" /{2..3} ; do curl -s https://thenonstopradio.com/genre/trance$i | htmlq -a href a | grep "/radio/" | uniq | awk -F '/' '{print $5}' >> A-trance.txt ; done
```

the double quotes that don't have anything in them is equel to is nothing but we still need it because we don't want to write a new command for each page, as you see everything is the same but the `$i` after the `trance` page was ran 3 times and each time the value of it was different

`{2..3}` here tells the shell to have a sequence starting with 2 and ending with 3, now this is useless in this case but helpful overall since there are genres with more than 3 pages, for example if i want the first 10 pages of `pop` streams i would do it like this `{2..10}`

i've added a `A-` to the beginning of the new file name so it's easier to find the file and not replace the older ones if there are any

<br>

## now lets do this for all the genres and pages
the scale is bigger now, lets use that `genre.txt` we made earlier to find all the names for genres

here another for loop is added to the mix which might look a bit complicated but everything will make sense if you pay attention

```
for i in "" /{2..10} ; do for j in $(cat genre.txt) ; do curl -s https://thenonstopradio.com/genre/$j$i | htmlq -a href a | grep "/radio/" | uniq | awk -F '/' '{print $5}' >> A-$j.txt ; done ; done
```

so what happened? we assigned nothing `"` thru `/10` to `$i` and repeating each genre names `$(cat genre.txt)` to `$j`, so the genre also shown as `$j` is first and the page number also known as `$i` is second in the `curl` command

other stuff is the same, just pipe the data thru each program, extracting only the page name and also saving them by appending `A-` prefix, the genre name at the middle and the `.txt` extension at the end

the same thing can be done to other categories

```
for i in "" /{2..5} ; do for j in $(cat country.txt) ; do curl -s https://thenonstopradio.com/country/$j$i | htmlq -a href a | grep "/radio/" | uniq | awk -F '/' '{print $5}' >> A-$j.txt ; done ; done
for i in "" /{2..5} ; do for j in $(cat language.txt) ; do curl -s https://thenonstopradio.com/language/$j$i | htmlq -a href a | grep "/radio/" | uniq | awk -F '/' '{print $5}' >> A-$j.txt ; done ; done
for i in "" /{2..5} ; do for j in $(cat network.txt) ; do curl -s https://thenonstopradio.com/network/$j$i | htmlq -a href a | grep "/radio/" | uniq | awk -F '/' '{print $5}' >> A-$j.txt ; done ; done
```

<br>

## scrape the stream titles and link
now that we have everything ready, lets actually scrape things, lets take that first page we started with and extract the title and link from it

```
curl -s https://thenonstopradio.com/radio/edm_sessions_us | grep -oP 'data-audio-url="\K[^"]+' | head -n 1 
```
```
https://s2.radio.co/s30844a0f4/listen
```

`grep -oP 'data-audio-url="\K[^"]+'` extracts the contents within the two quotes after every instance of `data-audio-url=` , `head -n 1` only shows the first line of this output since we know the other 20 are not the ones we are looking for

now for getting the title, as you might know html pages have their most prominent titles inside a `h1` tag, so using `htmlq` lets extract it


```
curl -s https://thenonstopradio.com/radio/edm_sessions_us | htmlq -t h1
```
```
EDM Sessions
```

so we have it, lets put it together, the way `m3u` files are set up is the first line is always `#EXTM3U` other lines are alternated by first having the stream title and then the stream linke, something like this 

```
#EXTM3U
#EXTINF:-1,EDM Sessions
https://s2.radio.co/s30844a0f4/listen
```

so we need to scrape each page twice, to keep things as smooth as possible and not overburden the server lets first save each page as a local copy, get the title, then the stream link and alternate them to have a proper `m3u` playlist

i will call this temporary file `mep1`, lets also append another `A` prefix to the output text files to know which one is which

```
for i in A-*.txt ; do for j in $(cat $i) ; do curl -s https://thenonstopradio.com/radio/$j > mep1 ; cat mep1 | htmlq -t h1 | awk '{print "#EXTINF:-1,"$0}' >> A$i ; cat mep1 | grep -oP 'data-audio-url="\K[^"]+' | awk NF | head -n 1 | sed 's|https://thenonstopradio.com/play?url=||g' | sed 's/\;//g' | sed '/^$/d' >> A$i ; echo -e "$i - $j" ; done ; done
```

there is quite a lot going on so lets unpack it:

`for i in A-*.txt` find the text files starting with `A-` these are the genre, country, language and network names

`do for j in $(cat $i)` repeat the contents of above files so we can append each stream name to the link we feed to `curl`

`curl -s https://thenonstopradio.com/radio/$j > mep1` save the temporary file

`cat mep1 | htmlq -t h1 | awk '{print "#EXTINF:-1,"$0}' >> A$i` cat the temporary file and extract the title, append a `#EXTINF:-1,` to the begining of it and send it to the new text file

`cat mep1 | grep -oP 'data-audio-url="\K[^"]+' | awk NF | head -n 1 | sed 's|https://thenonstopradio.com/play?url=||g' | sed 's/\;//g' | sed '/^$/d' >> A$i` send the stream link to the text file as well

`echo -e "$i - $j"` tell me what you are working on

<br>

## finishing touches
some streams might not have a link attached to them, lets exclude them from our playlists, lets use `grep` and `awk` for this

```
for i in AA-*.txt ; do cat $i | awk '!seen[$0]++' | grep -B1 "http" | grep -A1 "EXTINF" | awk 'length>4' > A$i ; echo -e $i ; done
```

find all output text files that start with `AA-`, repeat them, using awk remove duplicates `awk '!seen[$0]++'`, using grep find the first line above every instance of `http` and the first line below every instance of `EXTINF` which will effectively remove stream titles that don't have links and vise versa

finally use awk to remove items that are only 4 character or less in length, now our new file names have `AAA-` at the beginning

now lets convert these text files to m3u format by adding `#EXTM3U` to their first line

```
for i in AAA-*.txt ; do sed '1s/^/#EXTM3U\n/' $i > $i.m3u ; done
```

and remove all instance of `AAA-` and the extra extension of `.txt`

```
for i in *.m3u ; do mv "$i" "`echo $i | sed -e 's/AAA-//' -e 's/.txt//'`" ; done
```
