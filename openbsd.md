# Tim's OpenBSD guide
This guide is to help setup OpenBSD for a Lenovo X1 carbon Gen 9. 

# Setup the bios

Enable legacy s3 mode suspend then disable two items of hardware: bluetooth and the figerprint reader. 

# Install needed tools
	Use ``pkg_add`` to install all needed packages. 

	pkg_add git chromium neovim i3 a2ps neomutt ffmpeg go pandoc\
	texlive_texmf-minimal texlive_base wpa_supplicant glpk enscript\
	cmus zathura-ps zathura-pdf-mupdf ghostscript gnuplot openconnect\
	password-store w3m unzip expect font-awesome sc ranger abook\
	wordnet diction a2ps mpv lagrange Xdg-utils fzf bat recutils\
    fann calc sox 

# Add my user to the staff group 
```
doas usermod -G staff tim
```

# Disable services
I do not need smpd or sshd on a laptop, so lets disable all the stuff that I do not use. 
```
doas rcctl disable smtpd
doas rcctl disable sshd
doas rcctl disable vmd
doas rcctl disable ntpd
doas rcctl disable slaacd
doas rcctl disable check_quotas
```

# Turn off additional virtual terminals 
	
	doas vi /etc/ttys
	
Turn ttyC2 and ttyC3 to "off"


# Setting up wifi
/etc/hostname.iwx0:
```
	join WiFi-111111 wpakey 11111111 up 
	dhcp

	join eduroam wpa wpaakms 802.1x up 
	dhcp
```

If you want to join a random open network when travelling: 


```
    doas ifconfig iwx0 join ""
```

# Setting the timezone 

```
    doas ln -sf /usr/share/zoneinfo/Australia/Sydney /etc/localtime
```

If you are say in the Netherlands then you need to set this to: 

	/usr/share/zoneinfo/Europe/Amsterdam


# Setting x resources
~/.Xresources                                   

	xterm*faceName: Monospace
	xterm*faceSize: 8
	XTerm*background: #002B36
	XTerm*foreground: #D2D2D2
	XTerm.vt100.scrollBar: false
	XLock.dpmsoff     : 1
	XLock.mode        : blank
	XLock.echokeys    : off
	Xft.autohint  : 0
	Xft.lcdfilter : lcddefault
	Xft.hintstyle : hintslight
	Xft.hinting   : 1
	Xft.antialias : 1
	Xft.rgba      : rgb
	*font         : -misc-fixed-medium-r-semicondensed-*-13-*-*-*-*-*-iso10646-1

Remember to run:

	xrdb -merge .Xresources

## Edit ~/.xsession to add i3 and remove fvvwm 

This adds mouse scroll, sets the background colour, sets monitor dpms timeout, locks the screen if idle,
disables capslock, hides the mouse pointer when typing, and sets i3 as the WM. 

	```
	xrdb -load $HOME/.Xresources
	xinput set-prop "/dev/wsmouse" "WS Pointer Wheel Emulation" 1
	xinput set-prop "/dev/wsmouse" "WS Pointer Wheel Emulation Button" 2
	xinput set-prop "/dev/wsmouse" "WS Pointer Wheel Emulation Axes" 6 7 4 5
	export ENV=$HOME/.kshrc
	xsetroot -solid steelblue &
	xset dpms 0 60 150&
	xidle -delay 150 -sw -program "/usr/X11R6/bin/xlock -mode blank -dpmsoff 10"&
	xmodmap -e "keycode 66 = Shift_L"&
	ulimit -Sc 0
	mkdir -p /tmp/cache
	mkdir -p /tmp/cache/.mozilla
	exec /usr/local/bin/i3 
	```

My approach is that the xset commands will handle screen blanking, and command+l will lock the screen.

ulimit is used here to limit the number of coredump files to zero. 

# Edit .kshrc
I put some aliases here and a notes database script ( with search ).
note I use neomutt, neovim, and I have installed todo.sh.
this adds a plain text note taking alias n() 

```
alias vim=nvim
alias mutt=neomutt
alias svim="doas nvim"
alias t=todo.sh
alias o="sh ~/scripts/fopen.sh"
alias prnt="sh ~/scripts/prnt.sh"
alias ss="sh ~/scripts/search.sh $1"
alias s="cat $HOME/scratch.md | grep $1"
alias p="sysctl -n hw.sensors.acpibat0.power"
alias sp="cat /usr/share/dict/words /usr/share/dict/british | fzf"
alias komp="cd /usr/src/sys/arch/amd64/compile/CUSTOM.MP && make -j 16 && make install

export LC_CTYPE=en_US.UTF-8

sse(){
    cp ~/scratch.md /tmp/scratch.md
    sh ~/scripts/search.sh $1 > /tmp/extract.md
    date +'%n## %Y-%m-%d %H:%M:%S' >> /tmp/extract.md
    nvim -c 'startinsert' + /tmp/extract.md
    awk 'NR==FNR{a[$0]; next} !($0 in a)' /tmp/extract.md /tmp/scratch.md > /tmp/diff.md
    awk '{ if ($0 ~ /##/) print "\n\n" $0; else print $0 }' /tmp/diff.md > /tmp/diff_c.md
    cat /tmp/diff_c.md /tmp/extract.md > $HOME/scratch.md
}

ssq(){
        tags=`cat /home/tim/scratch.md | grep ^@ | fzf`
            sh ~/scripts/search.sh $tags
}

ssi(){
        ss @idea | sed 's/##/.Sh /g' > /tmp/test.md
            cat /tmp/test.md
}

n() {
	nf="$HOME/scratch.md"
	date +'%n## %Y-%m-%d %H:%M:%S' >> $nf
	sed -i -e '$a\' $nf
	echo >> $nf && nvim -c 'startinsert' + $nf
}

yta() {
    hasher=`echo -n "$*" | md5`
    file_path=~/audio/"$hasher".ogg
    if [ -f "$file_path" ]; then
        mpv "$file_path"
    else
        mpv --stream-record="$file_path" --ytdl-format=bestaudio ytdl://ytsearch:"$*"
    fi

}

play() {
        while IFS= read -r line || [ -n "$line" ]; do
            [ -z "$line" ] && continue  # Skip empty lines
            echo "Playing: $line"
            yta "$line"
        done < ~/audio/playlist.play
}


ytv() {
    hasher=`echo -n "$*" | md5`
    file_path=~/video/"$hasher".webm
    if [ -f "$file_path" ]; then
        mpv "$file_path"
    else
        mpv --stream-record="$file_path" --ytdl-format="bestvideo[height<=?720][vcodec!=h264]" ytdl://ytsearch:"$*"
    fi
}

calendar

```

# scratchpad search script (search.sh)

```
awk -v RS='##' -v search="$1" '
	{
		if ($0 == "") next;
		if (index($0, search) > 0) {
			printf "##%s" , $0;
		}
	}
' scratch.md

```


# Kill xconsole starting 
Find the xconsole line in the xenodm file and kill it to stop the console starting up. 
	sed -i 's/xconsole/#xconsole/' /etc/X11/xenodm/Xsetup_0

# Generate a new RSA key 
```
ssh-keygen -t rsa
```

Adding the key to a remote server is easy:

```
cat ~/.ssh/id_rsa.pub | ssh tim@bigcass 'cat >> ~/.ssh/authorized_keys'
```

Note add ssh-agent to login.

```
eval $(ssh-agent -s)
ssh-add ~/.ssh/id_rsa
```

I think Openbsd just prompts to open it next login...

This is because of /etc/X11/xenodm/Xsession
If you want this to stop comment out the line 
	eval `ssh-agent -s -k`


# Add self to sudoers list
This will let you use ``doas`` which is like ``sudo``

	echo 'permit persist keepenv tim' > /etc/doas.conf

edit /etc/doas.conf and also add the following lines:

```
	permit persist keepenv tim
	permit nopass tim as root cmd mount
	permit nopass tim as root cmd umount
	permit nopass tim as root cmd ntfs-3g
	permit nopass tim as root cmd /usr/sbin/arp
	permit keepenv nopass tim as _pbuild
	permit keepenv nopass tim as _pfetch
```
This will let you mount stuff as a non root user and send arp wakeup packets. 

# Some performance tweaks
derived from (https://www.c0ffee.net/blog/openbsd-on-a-laptop/)
first add self to the staff group:

```
	usermod -G staff tim
```

edit /etc/login.conf

look for staff ensure the following:
 
  - datasize-cur=1024M
  - datasize-max=8192M
  - maxproc-cur=512
  - maxproc-max=1024
  - openfiles-cur=4096
  - openfiles-max=8192
  - stacksize-cur=32M


edit /etc/sysctl.conf
```
	# enable hyper threading
	hw.smt=1
	
	# enable audio and video recording
	kern.video.record=1
	kern.audio.record=1

	# shared memory limits (chrome needs a ton)
	kern.shminfo.shmall=3145728
	kern.shminfo.shmmax=2147483647
	kern.shminfo.shmmni=1024

	# buffer cache
	kern.bufcachepercent=90
	
	# turn off ipv6 redirects
	net.inet6.ip6.redirect=0
	net.inet6.ip6.use_deprecated=0

    # to set a charge limit 
    hw.battery.chargestop=90

    # set powersave
    hw.setperf=0
```

edit /etc/fstab
Make sure to add **noatime** to most mounts.
Note softdep now does nothing. 

```
	e.g. / ffs rw,noatime 1 1
```
Sometime in 2023 softdep was disabled so this is no longer needed. 
noatime will not write access times. This should offer a minor speed up and not write when you access a file.

A note on the two last fields. The second last field tells dump when how many days between backups, and the last field indicates priority for fsck. 

## APMD
If you are not going to add SpeedShift to the kernel then you probably want apmd on. 

turn on apmd 

    rcctl enable apmd
	rcctl set apmd flags -A -Z 2
	rcctl start apmd
 


# tmpfs
The default config does not use tmpfs nor does tmpfs work under openbsd.
man mount_tmpfs notes that it does exist, however it will not work, the forums note that it was disabled in the kernel and thus we should use mfs. 

in /etc/fstab add:
	
	swap /tmp mfs rw,nodev,nosuid,async,-s=1024m 0 0
	
note the permissions of /tmp need to be sorted first 
When tmp is unmounted do the following. 

	chmod 1777 /tmp

Then do as follows when rebooted / mounted 

	mkdir -p /tmp/cache
	mkdir -p /tmp/cache/.mozilla
	ln -s /tmp/cache /home/tim/.cache
	ln -s /tmp/cache/.mozilla /home/tim/.mozilla

I added to .xsession: 
	mkdir -p /tmp/cache
	mkdir -p /tmp/cache/.mozilla

The point of using MFS here is that it will reduce the number of writes to the disk. 

# Middle button scroll 
## Enable scrolling with trackpoint and middle button
This came from (https://www.c0ffee.net/blog/openbsd-on-a-laptop/)
	
    xinput set-prop "/dev/wsmouse" "WS Pointer Wheel Emulation" 1
	xinput set-prop "/dev/wsmouse" "WS Pointer Wheel Emulation Button" 2
	xinput set-prop "/dev/wsmouse" "WS Pointer Wheel Emulation Axes" 6 7 4 5

Note that these commands will be placed in a config file see above. 

# Enable recoring of audio and video

	echo kern.audio.record=1 >> /etc/sysctl.conf
	echo kern.video.record=1 >> /etc/sysctl.conf

# Edit /etc/fbtab to enable access to /dev/video

add:
```
:/dev/video:/dev/dri/card1
```
to the end of the line. 

# eduroam

wpasupplicant needs to be installed as noted above. 
then copy the example config from /usr/local/share/examples/wpa_supplicant/wpa_supplicant.conf and edit it. 

in /etc/wpa_supplicant add the following: 

```
	network={
        ssid="eduroam"
		key_mgmt=WPA-EAP
		pairwise=CCMP TKIP
		group=CCMP TKIP
		eap=PEAP
		phase1="peaplabel=0"
		phase2="auth=MSCHAPV2"
		ca_cert="/etc/ssl/cert.pem"
		identity="1111111@youredu"
		password="Password"
	}
```

then enable wpa_supplicant with the following command: 

	rcctl enable wpa_supplicant

note that /etc/hostname.iwx0 will need to also contain the following line

	join eduroam wpa wpaakms 802.1x up
	dhcp

Note: my uni is now using a self signed certificate.
to make this work: 
download their CA cert. 

```
curl -0 https://www.myit.unsw.edu.au/sites/default/files/documents/UNSW_Root.crt > UNSW_root.crt
```

and copy it to the end of /etc/ssl/cert.pem (do as root) 


```
cat UNSW_Root.crt >> /etc/ssl/cert.pem
```

An alternative is to comment out
```ca_cert=
``` 

then your certificate will not be verified. 

Commenting out the certificate may be required when you are at a non-university location. This is required at an organisation I visit and the symptom of this being required is when you connect but continually dropout. 

Also of note is that android sets: 
	
```    
eapol_version=1
```

This setting is also required at organisation I visit otherwise there is constant dropout. 


# USB hub

I have a network port on the usb hub. it is automatically detected and works fine. 

run 
	doas ifconfig cdce0 inet autoconf 

# Webbrowser setup

I do use chromium but you can try: 
	chrome --enable-unveil --dbus-stub -enable-features=WebUIDarkMode,VaapiVideoDecoder,VaapiVideoEncoder --force-dark-mode

make sure you have installed the intel vaapi drivers: 
	
	pkg_add intel-media-driver 
	pkg_add intel-vaapi-driver

Unveil will restrict chrome to only be able to access Downloads. 
Enabling unveil is now the default. If you read forums there are lots of options to make chrome "better" but none of them seem to do much.  
Darkmode will make it easier on the eyes.

to get teams working I tried the following flags in chrome://flags
but the trick was to install the two packages above for intel vaapi


note:
chrome://gpu
will show or should show:
```
    Video Decode: Hardware accelerated
    Video Encode: Hardware accelerated
```

If you use Firefox esr
	firefox-esr -setDefaultBrowser

note configure to ensure that cache is stored in /tmp/cache
also note you need to tell it to set default browser or it will prompt each time. 

Note: H264 hardware encoding for cast streaming is needed for others to see your video. 


# Passwords
The password manager password_store is installed. This can be integrated with dmenu to allow for the quick retrieval of credentials for websites etc.

note: you need to setup a gpg key first.

to generate a gpg key run the following:
	gpg --full-gen-key

accept the defaults but remember the email address that you use. 

init with:
	pass init blah@gmail.com

add with:
	pass insert linkedin

list with:
	pass ls

## Setting up dmenu to work with pass

see [passmenu](https://github.com/cdown/passmenu)
you can get the code for passmenu from:
	/usr/local/share/examples/password-store/dmenu/passmenu

shopt is not found on openbsd, not in KSH. You need to run this script with bash or a bash compatable shell.
I installed bash to do this. 


# Suspend on lid close.
This just works if you set the bios to allow s3 suspend rather than windows+linux suspend. The latter is a new thing where firware is not really doing what it is doing in S3 suspend.  In 2025 either now work for me in Openbsd 7.3

## Lock on lid close 

To lock the screen on lid close create the suspend file and use it to send a kill signal to xidle. 

	doas mkdir -p /etc/apm

create a file named "suspend"

	vim /etc/apm/suspend

```
#!/bin/sh

pkill -USR1 xidle
```

	doas chmod +x /etc/apm/suspend

# Ms teams
Just works using chromium, when an external webcam is used. 
I plug it in and then: 

```
cd /dev
doas rm video
doas ln -s video1 video 
```

We now have a working system with no jerkyness or issues. 
note if you delete the wrong thing you can run **/dev/MAKEDEV video** 
and that might fix it.

The way to properly fix this is to modify the kernel as detailed below. 

When that is done chrome will work with the camera if you do the following: 

hardware accelerated decode must be enabled in flags and the correct packages installed (intel vaapi drivers) this is described latter. 

## Webcam issues. 
The compressed stream is not a standard mjpeg stream. It is pix_fmt yuvj422p 

If you try and use mjpeg with ffmpeg for the video stream you will see an error message like.

```
video4linux2 mjpeg no Jpeg data found.
```

but a raw stream is smooth and works. 

This works, because it is raw:
	
    ```
    video -s 800x480
    ```

The trick is the pix_fmt is yuvj422p then it works fine. 

you can get smooth compressed video from the webcam:

```
	ffmpeg -f v4l2 -input_format mjpeg -pix_fmt yuvj422p -video_size 1280x720 -i /dev/video0 ~/video.mkv
```

or run 
```
	mpv --profile=low-latency --untimed av://v4l2:/dev/video0
```

The following works with compressed mjpeg fine:

```
	ffplay -f v4l2 -input_format mjpeg -pix_fmt yuvj422p -video_size 1280x720 -i /dev/video0
```


We can get a smooth high res uncompressed image from the webcam.

in fact:
	ffplay -f v4l2 -input_format yuyv422 -video_size 848x480 -i /dev/video0

works super well. 


## Hacking the kernel to fix the video issue
There are two things you could do to fix this issue 1) set the maximum size of the returned stream or 2) change the described pixel format

edit (./sys/dev/usb/uvideo.c)

change this line which is about 2950. 
	fmtdesc->pixelformat = V4L2_PIX_FMT_MJPEG;
	to
	fmtdesc->pixelformat = V4L2_PIX_FMT_YUV422P;

Basically this makes it report the correct pixel format from the camera. 
this is inside the 
```uvideo_enum_fmt(void *v, struct v4l2_fmtdesc *fmtdesc)```
function


# Backup
Redacted 

# Cisco VPN 
There is an openconnect client that will do the vpn stuff for us named openconnect. There are heaps of options here but what works is:

	pkg_add openconnect-8.20p1-light
	openconnect https://vpn.blah.edu.au/blah

You will then be prompted to add your username and password. 
you can use flags for the username, password, and to background the application once it is going. 

You can use an expect script to automate the vpn login process. 

## expect script 

```
#!/usr/local/bin/expect
set password [lindex $argv 0];
spawn openconnect https://vpn.someplace.org -u 1111111 
expect "Please enter your zID and zPASS.\r"
sleep 2
send "$password\r"
sleep 1
interact
```

Launch the vpn as root. 

```
#!/bin/sh
doas expect vpn.expect `pass show mail/outlook`
exit 0
```


# Printing

Printing happens through lpd. You can find an example **printcap** config file at **/etc/examples/printcap** copy it to /etc then edit the remote example. 
In this case, at the uni the printer that I use has the ip address of 131.236.52.18
note that **sd** = spooler directory, **lf** = log file, **rp** = remote printer name. 
Once you have configured this you need to create those folders with ownership of **root:daemon** and with permissions of **770**. 

	mkdir /var/spool/output/lpd
	chown -R root:daemon /var/spool/output/lpd 
	chmod 770 /var/spool/output/lpd 

The configuration that I have for the university printer in /etc/printcap is

	lp| fatprinter:\
		:lp=:rm=111.111.11.11:rp=lp:sd=/var/spool/lpd:lf=/var/log/lpd-errs:

The difference between a remote printer and a local printer is the pipeline. 
if the printer is a **lp** then a pipeline of filters is used.  

add: **lpd_flags="-s"** to /etc/rc.conf.local
then 
	rcctl start lpd 
test it out and if it is ok 
	rcctl enable lpd 

to print now just pipe stuff to **lpr**

	cat test.txt | lpr

	or 
	lpr blah.pdf

Better yet postscript or PDFs can be piped to lpr. There are a few ways of getting postscript. pandoc, lowdown or enscript. 

Note **-s** ensures that lpd only listens to a unix socket and not from the whole world. 

## Clearing the printing queue
lpq displays the queue, lprm cancels jobs
lprm -

## Duplex and staples
We have an Apeos c4570 this can do duplex and staples.  

The command to do duplex and a staple is actually added to the postscipt file before it is sent to the printer.. 

a2ps can do duplex printing with the **-s2** option I like to also tell it to print one page per file. 
	
	a2ps -1 -s2 done.md  

To add duplex printing to PDFs you can use pdf2ps and then enscrypt.  


Staples are listed in the PPD for this printer and can be added in postscript. After:
	```
	%%Page: (1) 1
	```

	add the following:

	```
	<</Staple 3 >> setpagedevice
	<</OutputType (FINISHER TRAY1)>> setpagedevice

	```

Note that you need to send the document to the finishing tray and tell it to add a staple. 
The last page must also contain the following: 
	 ```
	showpage
	%%Trailer               
	<</Staple 0 >> setpagedevice      
	end
	```

## Proof of concept script to duplex and staple anything 
This is a hacky demo of how to do this do not use. 
Two scripts are required one bash script and one python script.
if it is not a pdf it will duplex it, then add the staple commands. 
If it is a pdf it will just send the staple commands. 


```
#!/bin/sh

if [[ $# -eq 0 ]]; then 
        echo "specify a file"
	exit
fi

filein=$1
fileout=${filein}

if [[ -s $filein ]]; then 

	echo $filein

	# if it is a pdf file run pdf2ps. 
	if [[ $filein == "*.pdf" ]]; then
		pdf2ps $filein /tmp/blah.ps 
	else
		sed 's/%%Trailer/%-%Trailer/g' $filein > /tmp/$fileout
		a2ps -1 -s2 -o /tmp/blah.ps /tmp/$fileout
	fi

	python3 ~/scripts/staple.py
	lpr /tmp/blah2.ps
	rm /tmp/blah.ps
	rm /tmp/blah2.ps
	rm /tmp/$fileout
	echo "Printing complete"
fi
exit 0
```

```
inputfile = open('/tmp/blah.ps', 'r').readlines()
write_file = open('/tmp/blah2.ps','w')
head_flag = False
tail_flag = False

for line in inputfile:
	write_file.write(line)
	if '%%Page: ' in line:
		if not head_flag:
			new_line = "<</Staple 3 >> setpagedevice"
			write_file.write(new_line + "\n")
			new_line = "<</OutputType (FINISHER TRAY1)>> setpagedevice"
			write_file.write(new_line + "\n")
			head_flag = True

	if '%%Trailer' in line:
		if not tail_flag:
			new_line = "<</Staple 0 >> setpagedevice"
			write_file.write(new_line + "\n")
			tail_flag = True

write_file.close()

```

I also edited ~/.kshrc and added the following alias:

	alias prnt="sh ~/scripts/prnt.sh"


# Brother Multi-function laser printer

I have a MFC-L2713DW BW laser printer. This is not a real postscript printer. 
This is a GDI printer so although it says on the box it supports postscript the printer does not,
Well it kinda does when used as an IPP printer but we will look at that latter. 
Basically in windows the driver converts postscript into a bitmap format and then sends that to the printer. 
We can use the brlaser package without cups in the following way: 

first install brlaser
	
	pkg_add brlaser

Then to print do the following: 
convert to postscript: 

	enscript -DDuplex:true test.txt -o /tmp/print.ps

convert postscript to cups raster format:
	
	gs -r600 -sPAPERSIZE=a4 -dFIXEDMEDIA -dPDFFitPage -sDEVICE=cups -o /tmp/print_cups.ps /tmp/print.ps

convert cups raster format to br-script format:

	/usr/local/libexec/cups/filter/rastertobrlaser 1 tim tim1 1 1 /tmp/print_cups.ps > /tmp/print_send.file

then send it to lpr or port 9100

	nc 192.168.1.111 9100 < /tmp/print_send.file


Note I can add all this as a filter to printcap and will do so in the future. 


This printer is an IPP printer, so you can use a chrome addon "IPP / CUPS printing for Chrome & Chromebooks" 
the printer should be configured as 
```
http://192.168.1.111:631
```
It will now just work.



# Screencasting
For normal screencasts remember that the device is **x11grab**
	
	ffmpeg -video_size 1680x1050 -framerate 30 -f x11grab -i :0.0 -c:v libx264 -qp 0 -preset ultrafast output.mkv

For a more advanced option where you have a video of yourself inside the screencast. Try the following, note that there are two **f** devices specified one is /dev/video0 and the other is **x11grab**.
```
ffmpeg -f x11grab -r 15 -video_size 1920x1080 -i :0.0\
        -f v4l2 -video_size 320x240 -framerate 15 -pix_fmt yuvj422p -i /dev/video0 \
	-f sndio -i snd/1 -vsync 0\
	-filter_complex 'overlay=main_w-overlay_w:main_h-overlay_h:format=yuv444' \
	-c:v libx264 -preset ultrafast -tune zerolatency -b 1m -c:a libmp3lame \
	-ab 24k -ar 22050 "$HOME/video.mov"

	echo "transcoding..."
mpv "$HOME/video.mov" -o="$HOME/screencast-$(date '+%y%m%d-%H%M-%S').mp4"\
	--profile=myencprofile

```

Just recording video try the following:

```
ffmpeg -f v4l2 -input_format mjpeg -pix_fmt yuvj422p -video_size 1280x720 -i /dev/video0 -f sndio -i snd/1 -c:v mpeg4 -b:v 2M -b:a 192k -c:a aac -vsync 1 ~/video.mkv

```

Transcoding can take a long time on a laptop so I scp the video to a server, transcode, and scp the output back 


```
echo "remote transcode..."
scp $1 tim@bigserver:~

ssh tim@bigserver "ffmpeg -y -i $1 -threads 128 -row-mt 1 -vcodec libvpx-vp9 -movflags +faststart -acodec libvorbis -af 'volume=1.5' -s hd1080 -preset slow -vsync -1 '$HOME/output.webm'"

scp tim@bigserver:~/output.webm T$(date '+%y%m%d-%H%M-%S').webm

```

# Misc tools 

## sc-im

sc-im is an improved version of sc. At the moment sc works well for me but I could and have used sc-im in the past. 
[sc-im](https://github.com/andmarti1424/sc-im.git)
git clone https://github.com/andmarti1424/sc-im.git

to make sc-im you need gmake rather than make. 

## fann
you can get git from here 
git clone git@github.com:tlynar/fann.git

# Banish mouse
To get rid of the mouse when typing run the following:
	xbanish -s 

# Plantronics headphones
To get these headphones to work plug them in and then run the following command: 

    sndioctl server.device=1

Suddenly sound will start comming through them. Like magic!

This can be made to automatically switch through the follwoing two commands:

	rcctl set sndiod flags -f rsnd/0 -F rsnd/1
	rcctl restart sndiod

Once the switch has been made to the headphones, you can record with: 
	aucat -o file.wav

and playback sound with 
	aucat -i file.wav. 

# general sound 
/etc/mixerctl.conf:
	outputs.spkr2_source=dac-0:1

I play music via mpv, using mpv you can set equalizer values. 

my mpv.conf (~/.config/mpv/mpv.conf) file has the following line: 
	af="superequalizer=6b=2:7b=2:8b=2:9b=2:10b=2:11b=2:12b=1:13b=2:14b=2:15b=2:16b=2:17b=2:18b=2

There are a huge number of options for equalization so I only show how to do it not what to do. 

# Disable capslock 
I have no need for capslock. I am not sure if anyone actually means to ever use it. 
Thus lets disable it. The manual suggests to do the following: 

	wsconsctl keyboard.map+="keysym Caps_Lock = Control_L"
If you want to make this perminate then put it in **wsconsctl.conf**

Note this will not help in X. In X11 run: 
	xmodmap -e "keycode 66 = Shift_L"
I place this in my .xsession


# /etc/wsconsctl.conf
```
keyboard.map+="keysym Caps_Lock = Control_L"
display.width=1920
display.height=1200
display.brightness=30

```

# i3 config

to lookup passmenu
```
bindsym $mod+Shift+p exec "/usr/local/bin/bash \
/usr/local/share/examples/password-store/dmenu/passmenu
```

to lock the screen

```
bindsym $mod+Shift+x exec "pkill -USR1 xidle"
```

# .vimrc
Note vim and neovim use a different location for the vimrc.
neovim places the config at  **~/.config/nvim/init.vim**
simlink this location to .vimrc

# Security updates
To patch the system
run: 
```
doas syspatch

```
To upgrade all the packages run: 
```
doas pkg_add -u
```

# Using an external monitor / projector 
run:
```
xrandr --output HDMI-1 --auto --same-as eDP-1
```

if it is over usb-c use ```DP-3``` instead of ```HDMI-1```

To turn the projector back off

```
xrandr --output HDMI-1 --off
```

# Red shift
It has become the thing to reduce the blue in your monitor at night. It might help you sleep and they do it on a mac.  

```
xrandr  --output eDP-1 --gamma 1:0.88:0.76 --brightness 0.75
```
Where gamma is B:G:R

when you are done run it again and but with gamma 1:1:1

# Todo list

A simple todo list tool for the console. 

git clone https://github.com/todotxt/todo.txt-cli.git

note my kshrc file includes an alias t() for this list. 
```
alias t=todo.sh

```

after cloning 

copy todo.sh to /usr/local/bin 

```
cp todo.sh /usr/local/bin/
```

copy config to .todo
```
mkdir ~/.todo/
cp todo.cfg ~/.todo/config
```

I have to following in my config
```
export TODO_FILE="$TODO_DIR/todo.txt"
export DONE_FILE="$TODO_DIR/.done.txt"
export REPORT_FILE="$TODO_DIR/.report.txt"
```

# Secure DNS (optional).
The reason this is optional is because by default you will be using secure DNS via chrome. This will resolve with the selected secure source. You can make this quad9 or something similar, inside your Chrome settings. Note you will need to turn this off to use unwind to resolve names. Thus this is optional, the bulk of my resolution occurs via chrome, so this is added complexity for minor benifit. 


Create a new file /etc/unwind.conf

```
v4quad9="9.9.9.9 authentication name dns.quad9.net DoT"
forwarder { $v4quad9 }
preference { DoT }

block list "/var/db/unwind-block.txt" log

```
	doas rcctl enable unwind
	doas rcctl start unwind 
	doas touch /var/db/unwind-block.txt

Note there is a way to download a list of known bad hosts each day and add it to your unwind-block.txt, this will block adds or whatever you want to block. 

unwind has a cache. So it should reduce network traffic. The manual notes that hte cache is not configurable.


# Network time - syncing the clock 
I have disbled ntpd. I consider it to be bloat as for a laptop you can basically get there with a cron job. 

	rcctl stop ntpd 
	rcctl disable ntpd 

you can then set the time through a cron job. 

	rdate -nv time.cloudflare.com

to do this add an entry to the crontab to do it every 3 hours. 
	doas crontab -e 

```
	0 */3 * * *  rdate -nv time.cloudflare.com
```
Note that there are tabs between the elemnets and the last element is a command and arguments. 

There is a potential new problem with this. That is that time can jump and the server may be spoofed. 

The jump should be negated by using the **-a** flag and the **-n** flag should ensure SNTP is used. SNTP uses client-server authentication via Symmetric key cryptography to alleviate spoofing. 


# gtk 3.0 
```
mkdir -p ~/.config/gtk-3.0

vim ~/.config/gtk-3.0/settings.ini
```

```
[Settings]
gtk-theme-name=Adwaita
gtk-icon-theme-name=Adwaita
gtk-font-name=Arimo 12
gtk-toolbar-style=GTK_TOOLBAR_ICONS
gtk-toolbar-icon-size=GTK_ICON_SIZE_SMALL_TOOLBAR
gtk-button-images=1
gtk-menu-images=1
gtk-enable-event-sounds=1
gtk-enable-input-feedback-sounds=1
gtk-xft-antialias=1
gtk-xft-hinting=1
gtk-xft-hintstyle=hintslight
gtk-xft-rgba=rgb
gtk-cursor-theme-size=0
gtk-cursor-theme-name=Default
gtk-key-theme-name=Default
```

# Calendar 
Use the normal calendar app 

```
mkdir ~/.calendar
vim ~/.calendar/calendar 
```

Then add entries that are in the format of date /t thing
```
July 1  - First of the month
```
note that the space between the 1 and the - is a tab. 

A calendar is not generally used for reminders, there is a remind command for that. Some people hook up a script to mutt to open their remind file and allow them to manually enter it. 

Note alot of people use calcurse but I have not goten into it. 

# News 
I like to read the news. 
newsboat works well for me. 
There are two things to configure.
  * your browser
  * the rss feeds to fillow 


	~/.newsboat/config 

```
browser "w3m %u"
```

	~.newsboat/urls

```
http://www.9news.com.au/rss "news"
https://rss.dw.com/rdf/rss-en-all "news"

```
There are some RSS feed modifiers out there. That is they take an rss feed and then follow the links for you, creating a more readable RSS feed with actual information. 

```
https://morss.it/feeds.bbci.co.uk/news/rss.xml "fnews"
https://morss.it/rss.cnn.com/rss/edition.rss "fnews"
```

# Firewall

This is not the firewall script I use. But block by default and then allowing selected flows sounds resonable. 

edit /etc/pf.conf

```
    block all
    tcp_services = "{ssh, domain, www}"
    udp_services = "{ domain }"

    pass out proto tcp to port $tcp_services
    pass proto udp to port $udp_services
    set skip on lo

    block return    # block stateless traffic
    pass            # establish keep-state

    # By default, do not permit remote connections to X11
    block return in on ! lo0 proto tcp to port 6000:6010

    # Port build user does not need network
    block return out log proto {tcp udp} user _pbuild

```

Then reload the PF
	
    doas pfctl -f /etc/pf.conf

Viewing pf log via tcpdump

```
tcpdump -n -e -ttt -r /var/log/pflog
```


# General audio 
The audio driver used is azalia, quirks can be selected by editing azalia_codec.c 
No quirks are needed for my laptop. 

Playing back audio at > 100% volume can be attempted with mpv:
	
	```
	mpv --volume-max=200 https://www.youtube.com/watch?v=blah
	```
	
	note replace blah with whatever it is, then use 9 and 0 to adjust the playback volume. 

Note: 
	
	```
	--vid=no
	```
will supress video output. 

# MPV with hardware decoding 
Hardware decoding reduces energy consumption. 
Note if --hwdec-extra-frames is two low this will fail, also it does not work on all videos. 

~/.config/mpv/mpv.conf

```
vo=gpu
hwdec=auto-safe
hwdec-codecs=all
hedec-extra-frames=5
ytdl-format="bestvideo[height<=?720][vcodec!=h264]+bestaudio/best" 
af="superequalizer=6b=2:7b=2:8b=2:9b=2:10b=2:11b=2:12b=1:13b=2:14b=2:15b=2:16b=2:17b=2:18b=2
[myencprofile]
vf-add = scale=1920:-2
ovc = libx264
ovcopts-add = preset=slow
ovcopts-add = crf=23
ovcopts-add = b=1500k
ovcopts-add = maxrate=1500k
ovcopts-add = bufsize=1000k
oac = aac
oacopts-add = b=128k
```

These lines basically set the resolution to <= 720 for the height and asks for hardware decoding. 
also I modify equalizer values and added an encoder profile. 


# a note about music 

If you do not have spotify you can just stream radio and tv via mpv. 

ABC news radio
	mpv http://www.abc.net.au/res/streaming/audio/aac/news_radio.pls

Tripple J
	http://www.abc.net.au/res/streaming/audio/aac/triplej.pls

Tripple j Hottest 
	mpv http://hot100.mediahubaustralia.com/TJHW/aac/

ABC classic
	mpv http://www.abc.net.au/res/streaming/audio/aac/classic_fm.pls

ABC JAZZ
	mpv http://www.abc.net.au/res/streaming/audio/aac/abc_jazz.pls

Or just surch for and play a song with yta 

	yta drivers license karma

# Building a custom Kernel 

```
$ cd /usr
$ cvs -qd anoncvs@anoncvs.example.org:/cvs checkout -rOPENBSD_7_1 -P src
$ cd /sys/arch/$(machine)/conf
$ cp GENERIC.MP CUSTOM.MP
$ vi CUSTOM.MP    # make your changes
$ config CUSTOM.MP
$ cd ../compile/CUSTOM.MP
$ make clean
$ make -j 16
$ doas make install
```


# Debug usb issues 
To see a list of connected usb devices run 
	
	usbdevs 

# Resources
Setting up eduroam and x11 was bassed on the following:

[X windows system](https://www.openbsdhandbook.com/x_window_system/)
[eduroam kariliq.nl](https://www.kariliq.nl/openbsd/eduroam-uva.html)
[astro-gr eduroam](http://astro-gr.org/openbsd-eduroam/)
[C0ffee.net](https://www.c0ffee.net/blog/openbsd-on-a-laptop/)
[romanzolotarev openbsd blog](romanzolotarev.com)
[someone named zaney](https://gitlab.com/Zaney/dotfiles/)
[XOSC](https://xosc.org/articles.html)
[dataswamp Solene](https://dataswamp.org/~solene/)

# General notes

The only physical thing that does not work is the microphone. 
This is not fantastic, but not a show stopper either. I have a headset that I use for video conferencing, and that works great with this setup. 

The only soft thing that does not work is DRM.
Chromium for openbsd does not support widevine DRM and thus there will be no netflix here unless I setup a linux vm for that purpose. which I might do. 


The Bad:

  - Inbuilt microphone is cactus under openbsd. ( this is solved with an external one see above )  
  - No DRM support. ( I dont really use this )
  - No fingerprint reader ( I dont use this) 
  - No bluetooth ( I dont use this)

The good:
  
  - Secure
  - Easy to use
  - Works. For everything that I want to


## A bit of history
```
cal 9 1752
```

## Smooth scrolling in a pdf 
zathura-pdf-mupdf performs smooth scrolling. 

	pkg_add zathura-pdf-mupdf

## Other
is the swap encrypted?
vm.swapencrypt.enable=1
vm.swapencrypt.keyscreated=0
vm.swapencrypt.keysdeleted=0

hw.cpuspeed=400
hw.setperf=0
dep.allowaperture=0
hw.perfpolicy=manual

vm.malloc_conf=CF

# MIDI 
A note on setup: The midi adapter plug labeled "out" needs to go into the port labeled "in"

write midi to a midi file: 

```
midicat -d -q midi/0  -o test.mid
```

Using tools with the midish package:

Play a standard midi file:

```
smfplay -d midi/0 mysong.mid
```

record a standard midi file
```
smfrec -y -d midi/0 mysong.mid 
```

to play midi on the laptop use timidity
```
timidity file.mid
```


# waking my NAS 
To wake my nas I need to send an arp wake on lan packet
first ping the host then:

```
arp 192.168.1.111
doas /usr/sbin/arp -W 00:11:11:11:11:1
```



# Kernel notes
in cpu.c 

Stuffing around with MSRs, note these seam to make no difference:
```
	uint64_t msr;                                      
	msr = rdmsr(MSR_IA32_CST_CONFIG_CONTROL);                    
	msr |= (1 << 25); //C3 auto demotion
	msr |= (1 << 26); //C1 auto demotion
	msr |= (1 << 27); //c3 undemotion                        
	msr |= (1 << 28); //c1 undemotion
	msr |= (1 << 29); //c state demotion
	msr |= (1 << 30); //c state undemotion
	wrmsr(MSR_IA32_CST_CONFIG_CONTROL, msr); 
```

Setting system wide MSR_IA32_ENERGY_PERF_BIAS also looks to make no difference with HWP enabled. 
```
     if (CPU_IS_PRIMARY(ci)){    
         wrmsr(MSR_IA32_POWER_CTL, 2);  //C1E promo
                              
         // IA32_ENERGY_PERF_BIAS, 15 (0x0f) is the most energy saving
         // It also kinda sucks performance wise. 
         uint64_t msr;
         msr = rdmsr(MSR_IA32_ENERGY_PERF_BIAS);
         msr = (msr & ~0xF) | 15;       
         wrmsr(MSR_IA32_ENERGY_PERF_BIAS, msr);
     }                
```

## HWP implementation AKA speed shift

IA-32 Intel Architecture Software Developer's Manual, Volume 3b:
see: section 14.4.1 

Here is some code that will basically run/enable HPM and disable EST.
The union structs are from JCS's branch of Openbsd.
```
 wrmsr(IA32_PM_ENABLE, 1); //ENABLE HPM
                               
//Union structs from JCS
                                 
     union hwp_capabilities {            
         uint64_t msr;
         struct {      
             uint8_t high_perf;                                  
             uint8_t guaranteed_perf;                            
             uint8_t most_efficient;                            
             uint8_t low_perf;                                     
             uint32_t reserved;         
         } __packed fields;
     } hwp_cap;                              
                                                       
     hwp_cap.msr = rdmsr(IA32_HWP_CAPABILITIES);
                            
     union hwp_request {    
             uint64_t msr;                   
             struct {  
                     uint8_t min_perf;
                     uint8_t max_perf;
                     uint8_t desired_perf;               
                     uint8_t epp;
                     uint16_t act_win : 10;                
                     uint8_t package : 1;
                     uint32_t reserved : 21;
             } __packed fields;   
} pstate_hwp_req;

pstate_hwp_req.msr = rdmsr(IA32_HWP_REQUEST);
pstate_hwp_req.fields.act_win = 0; // 0 = dynamic selection
pstate_hwp_req.fields.epp = 0xff; // 0FFH = most energy efficient, default 80H
pstate_hwp_req.fields.desired_perf = 0; //should be 0
pstate_hwp_req.fields.min_perf = hwp_cap.fields.low_perf;
pstate_hwp_req.fields.max_perf = hwp_cap.fields.high_perf;      
 
wrmsr(IA32_HWP_REQUEST, pstate_hwp_req.msr);
```

Note setperf() currently calls mp_setperf() which calls est_setper() on each core.
When HWP is on this will still be called but you will need to sort that. 

I will include in a seperate file my implementation of this. I have been doing a few tests on this and adjusting the clock rate, to see if I can reduce energy consumption and extend battery life.

## EST
IA-32 Intel Architecture Software Developer's Manual, Volume 3b:
System Programming Guide.
Section 14.1, Enhanced Intel SpeedStep technology.

Enabled by setting bit 16 of IA32_MISC_ENABLE
Control is via IA32_MPERF(E7H) and IA32_APERF(E8H)

Write a 16bit value to IA32_PERF_CTL(0199H) bits 15:0 set the target p-state.
you disengage turbo boost/IDA by setting bit 32 of IA32_PERF_CTL to 1 (R/W sequence needed)

IA32_PERF_CTL = 64bit 
0:15 = target_perf
16:31 = reserved 
32 = turbo disengage
33:63 = reserved 

The openbsd implementation is in /usr/src/sys/arch/amd64/amd64/est.c



