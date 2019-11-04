---
title: PicoCTF 2017 Writeups
author: Andy Smith
date: 2018-01-27 06:51:53
tags:
---

## Forensics
#### Digital Camoflage (50pts)
Instructions
> We need to gain access to some routers. Let's try and see if we can find the password in the captured network data: data.pcap.

Hints
> It looks like someone logged in with their password earlier. Where would log in data be located in a network capture?
> If you think you found the flag, but it doesn't work, consider that the data may be encrypted.

Included Files
[data.pcap](https://ajsmith.us/files/f?h=1HyFDBFh&d=1)

This challenge requires a packet analysis tool such as Wireshark. Opening data.pcap in Wireshark reveals multiple GET and POST requests. The instructions specify that we need to find a password. By applying a filter of `http.request.method == POST` we end up with one request. By expanding the "HTML Form URL Encoded" subtree of this request, we find a userid and password. This may seem like the solution to the challenge, but `cHJ2cUJaTnFZdw==` is not the final flag. The hint claims this data is encrypted, but it is actually base64 encoded, noticing the `==` padding at the end. Decoding this password with `echo cHJ2cUJaTnFZdw== | base64 -d` reveals the flag, `prvqBZNqYw`.


#### Special Agent User (50pts)
Instructions
> We can get into the Administrator's computer with a browser exploit. But first, we need to figure out what browser they're using. Perhaps this information is located in a network packet capture we took: data.pcap. Enter the browser and version as "BrowserName BrowserVersion". NOTE: We're just looking for up to 3 levels of subversions for the browser version (ie. Version 1.2.3 for Version 1.2.3.4) and ignore any 0th subversions (ie. 1.2 for 1.2.0)

Hint
> Where can we find information on the browser in networking data? Maybe try reading up on user-agent strings.

Included Files
[data.pcap](https://ajsmith.us/files/f?h=35FnHcBa&d=1)

This challenge requires a packet analysis tool such as Wireshark. Opening data.pcap in Wireshark reveals multiple request types such as UDP, TCP, HTTP, ICMP and ARP. We just want to focus on HTTP since we are looking for the user agent. Applying the filter `http.request.method == GET` narrows down our search to 7 packets. The user agent will be inside the HTTP subtree. Going through each request reveals two different browsers, one of which is Wget, which can be ignored. The other is `Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/36.0.1985.67 Safari/537.36`. User agents have a long history and are a mess, so you can either guess which part of the string to use, or use a website to parse the string. In this case, the browser is almost certainly Google Chrome, but using a site like [UserAgentString.com](http://www.useragentstring.com/) confirms this. The answer format is specified in the instructions, so the flag is `Chrome 36.0.1985`


#### Meta Find Me (70pts)
Instructions
> Find the location of the flag in the image: image.jpg. Note: Latitude and longitude values are in degrees with no degree symbols,/direction letters, minutes, seconds, or periods. They should only be digits. The flag is not just a set of coordinates - if you think that, keep looking!

Hint
> How can images store location data? Perhaps search for GPS info on photos.

Included Files
[image.png](/blog/images/image.png)

This challenge deals with Exif data. Upon first look, there isn't anything visually unique about the photo. Some ways to start are to look for possible steganography or hidden data, but in this case the challenge is as simple as looking at the exif data. I used a tool called [Exiftool](https://www.sno.phy.queensu.ca/~phil/exiftool/) which can be easily installed in Kali with `apt install libimage-exiftool-perl`. For this challenge, the command usage is simply `exiftool [file]`.

The lines `Comment` and `GPS Position` have what we need.
```
root@kali:~/Downloads# exiftool image.jpg 
....
Comment                         : "Your flag is flag_2_meta_4_me_<lat>_<lon>_f8ad. Now find the GPS coordinates of this image! (Degrees only please)"
....
GPS Position                    : 91 deg 0' 0.00", 124 deg 0' 0.00"
```
Combining these together, the flag is `flag_2_meta_4_me_91_124_f8ad`.


#### Little School Bus (75pts)
Instructions
> Can you help me find the data in this littleschoolbus.bmp?

Hint
> Look at least significant bit encoding!!

Included Files
[littleschoolbus.bmp](https://ajsmith.us/files/f?h=3PCxCh6l&d=1)

This is a steganography problem which uses [least significant bit](https://en.wikipedia.org/wiki/Least_significant_bit) encoding. There are many tools out there that can help with this, and I found that [zsteg](https://github.com/zed-0xff/zsteg) is very useful. Just clone the repository and run `gem install zsteg`. Then just run `zsteg littleschoolbus.bmp` and that gives us our flag, `flag{remember_kids_protect_your_headers_8940}`


#### Connect The Wigle (140pts)
Instructions
> Identify the data contained within wigle and determine how to visualize it.

Hints
> Perhaps they've been storing data in a database. How do we access the information?
> How can we visualize this data? Maybe we just need to take a step back to get the big picture?
> Try zero in the first word of the flag, if you think it's an O.

Included Files
[wigle](https://ajsmith.us/files/f?h=1HvCiS_O&d=1)

Running `file wigle` reveals that this is a SQLite 3.x database. Kali has a preinstalled program `sqlitebrowser` that we can use to visually navigate the database. This reveals three tables: android_metadata, location, and network. Location seems to be the right table, since we are supposed to be able to visualize the information. Maybe the data points will spell out the flag? Let's export that location table as a CSV (File->Export->Table(s) as CSV file...). Make sure "New line characters" is set to either Windows or Linux. Google Maps lets you import data points through "My Maps". Create a new map and import the CSV file. Check the "lat" and "lon" fields. Title the markers with the "_id" column. Now if you zoom in on the map you can see that there are multiple characters that spell out the flag, which is `FLAG{F0UND_M3_C90C64E3}`.