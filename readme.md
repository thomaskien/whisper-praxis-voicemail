convert your voicemail to text email using whisper
====
* in our medical practice we do use voicmail quite much to postpone as much work as possible during the rush hours (*much* patients arriving and calling at the same time at opening).
* to do so we configured our auerswald telephone system with IVR to record voicemail and send them to a local mailbox in the practice.
* because of datenschutz we do not want the informations provided by the patients to leave the building
* the following script downloads the messages from voicemail1 = ab1@praxis.local and sends the text and attached the sound file to voicemail2 = ab2@praxis.local (in our case at the same machine using postfix and dovecot with system users)

thanks to the heise computer magazin c't where the idea came from (https://www.heise.de/select/ct/2023/14/2305417013270445796) and thanks to https://github.com/jamesridgway/attachment-downloader

requirements
==
hardware
===
* needs 8GB of RAM
  - computer crashes if less RAM installed
  - >5gb need to be free for whisper
* ca. 2012 old intel core i3/i5 is sufficient
  - we run it on a i3-3220 which was taken out of service since no windows 11 support
  - convertion time on that machine ca. 2min for a common prescription order with 20sec length
  - on my mac M4 the task takes about 10 seconds only
  - maybe it will also run on raspberry pi 5 8GB ram (I will test that)
software
===
* linux (script be adapted for mac easily and maybe windows)
* local installed mailserver the manual way or mailcow for docker
  - set SKIP_FTS=y SKIP_SOGO=y SKIP_CLAMD=y in mailcow.conf otherwise mailcow will eat all the ram
  - adjust "Forwarding Hosts" in configuration otherwise the "spam" the script sends is not accepted (missing headers etc)
  - create two mailboxes like ab1 and ab2
* telephone system that sends out voicemail like any fritzbox or auerswald
  - configure it to send the voicemail to ab1@praxis.local using the mail server of your local machine
 
preperations
==
* install fresh debian/raspbian (tested with debian 12 bookworm)
<pre>
# become root
sudo su
# maybe connect via ssh from your other machine if minimal raspberry install:
# service ssh start
# update and install requirements and whisper:
apt update
apt upgrade
apt install screen ffmpeg git mailutils swaks
python3 -m venv whisperenv; source whisperenv/bin/activate
pip install pip-review attachment-downloader
pip install git+https://github.com/openai/whisper.git
cd /root/
mkdir wav
mkdir done
touch email-transc.sh 
chmod +x email-transc.sh 
# now paste end edit file
nano email-transc.sh 
</pre>
script
==
* paste the following text and adjust what needed // remove that is not needed
<pre>
<code>
#!/bin/bash

#activate environment
cd /root/ 
python3 -m venv whisperenv; source whisperenv/bin/activate

#looping all 30 sec.
while : 
do
# =Fetch mails / empty mailbox1=
# choose the right for you (help: attachment-downloader -h)
#attachment-downloader --host=imap.1und1.de --username=XXXXXXX --password=XXXXXXX --output=./wav/
# the following good for auerswald since no caller number included in the file...
#attachment-downloader --unsecure --host=127.0.0.1 --username=ab1 --password=ab1 --output=./wav/ --filename-template="{{ subject }}.mp3" --delete 
#or:
#attachment-downloader --unsecure --host=127.0.0.1 --username=ab1 --password=ab1 --output=./wav/ --delete --filename-template="{{ subject }} -- {{ attachment_name }}" 
#attachment-downloader --unsecure --host=127.0.0.1 --username=ab1 --password=ab1 --output=./wav/ --delete 
#attachment-downloader --unsecure --host=127.0.0.1 --username=ab1 --password=ab1 --output=./wav/ 
#fritzbox-voicemail+mailcow
attachment-downloader --starttls --host=127.0.0.1 --username=ab1@praxis.local --password=ab1 --output=./wav/  --delete
date 

# maybe remove system characters like auerswald uses in the subject
cd wav 
#for file in *; do mv "$file" $(echo "$file" | sed -e 's/[^A-Za-z0-9._-]/_/g'); done 
cd .. 

# adapt to *.mp3 depending on you telephone system
FILES="./wav/*.wav" 
#FILES="./wav/*.mp3" 

# =Whisper/email send loop itself=
for f in $FILES 
do 
if [ -e "$f" ]; then 
echo "Processing $f file..." 
# take action on each file. $f store current file name 
whisper --model_dir ./models "$f" --language de --model large-v3-turbo --output_format txt --fp16 False 
fi 
# change if your system uses mp3
# OUT="${f/%mp3/txt}" 
OUT="${f/%wav/txt}" 
echo "Output: $OUT" 
mv *.txt wav/ 
if [ -e "$OUT" ]; then 
echo "File exists." 
echo >> "$OUT" 
echo >> "$OUT" 
#auerswald
#SUB=$(echo "$f" | sed 's/Voicemail___/+/g' | sed 's/wav//g' | sed 's/[.//]//g'| sed 's/_/ /g' | sed 's/+49/0/g' | sed 's/mp3//g') 
#fritz
SUB=$(echo "$f" | sed 's/Anruf./von__/g' | sed 's/.wav//g' | sed 's/wav//g' | sed 's/[.//]//g') 
echo Infos: "$SUB" >> "$OUT" 
echo >> "$OUT" 
echo Dateiname: "$f" >> "$OUT" 
echo >> "$OUT" 

#=send out email=
#system-mail needs to configre MTA e.g. postfix with smarthost:
#cat "$OUT" | mail -aFrom:Voicemail\<root@praxis.local\> -s "TEXT: $SUB" ab2@praxis.local -A $f
#sendEmail.pl also is good:
#cat "$OUT" | ./sendEmail.pl -t ab2@praxis.local -f ab1@praxis.local -s 127.0.0.1:25 -v -u "TEXT: $SUB" -a $f 
#swaks is the best option
swaks -s 127.0.0.1 -f Voicemail -t ab2@praxis.local --body $OUT --attach $f –suppress-data --header 'Subject: TEXT: '$SUB'' 

=clean up=
mv "$f" ./done/ 
cat "$OUT" 
mv "$OUT" ./done/ 
fi 
date 
sleep 3 
done
echo "Press [CTRL+C] to stop.."
# wait 30 seconds and then start from top
sleep 30 
done  
</code>
</pre>
test
==
* just run:
<pre>
./email-transc.sh
</pre>
start on boot
==
 * very easy using screen
 * just add the following line to the end of /etc/crontab (nano /etc/crontab)
<pre>@reboot root screen -dmS email-tra /root/email-transc.sh</pre>
check process
==
<pre>screen -r email-tra</pre>
* to detach screen to background again: crtl+a -> ctrl+d
use productivly
==
* change the passwords of the mailboxes
* change the admin-password of mailcow (standard admin:moohoo)
* set up ssh-key authentication and disable password-login on the machine
install updates
==
* whisper
<pre>
pip-review --interactive
pip install --upgrade --no-deps --force-reinstall git+https://github.com/openai/whisper.git
</pre>
* system:
<pre>
apt update
apt upgrade
</pre>
* mailcow:
 <pre>/opt/mailcow-dockerized/update.sh</pre>

