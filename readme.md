convert your voicemail to text email using whisper
====
* in our medical practice we do use voicmail quite much to postpone as much work as possible during the rush hours (*much* patients arriving and calling at the same time at opening).
* to do so we configured our auerswald telephone system with IVR to record voicemail and send them to a local mailbox in the practice.
* because of datenschutz we do not want the informations provided by the patients to leave the building
* the following script downloads the messages from voicemail1 = ab1@praxis.local and sends the text and attached the sound file to voicemail2 = ab2@praxis.local (in our case at the same machine using postfix and dovecot with system users)

thanks to the heise computer magazin c't where the idea came from (https://www.heise.de/select/ct/2023/14/2305417013270445796) and thanks to https://github.com/jamesridgway/attachment-downloader

![Screenshot](https://github.com/thomaskien/whisper-praxis-voicemail/blob/main/Screenshot.png)

requirements
==

hardware
===
* 1 separate or very powerful machine
* needs 8GB of RAM
  - computer crashes if less RAM installed
  - 5GB+ need to be free for whisper
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
# change language if needed:
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
* maybe use other user than root

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

 benchmark
 ==
 * using same wav file of typical 33sec recorded on fritzbox
 * output of the file:
<pre>
Ja, hallo, ich teste einmal hier XXX XXX geboren XXX. XXX 19XX, Telefonnummer 0xxxxxxxx.
Ich möchte bestellen Thorosemite 5 mg, die 100er Packung N3 und dann bitte auch Ramipril 2,5 mg, die 100er Packung.
Dann bitte auch Metoprolol Neuraxafarm 47,5 mg, auch die große Packung N3.
Genau, ich bedanke mich. Tschüss.
</pre>

model large-v3-turbo
=
 * Raspberry pi 5 without any cooler/ventilator (throtteling last seconds)
 <pre>
   Tue 18 Feb 11:03:40 GMT 2025
   Tue 18 Feb 11:07:06 GMT 2025
   =206sec
 </pre>
  * Raspberry pi 5 with passive cooler (not throtteling)
 <pre>
   Tue 18 Feb 11:22:05 GMT 2025
   Tue 18 Feb 11:25:30 GMT 2025
   =205sec
 </pre>
* iMac 2012 Intel(R) Core(TM) i5-2500S CPU @ 2.70GHz
 <pre>
   Tue Feb 18 12:16:07 CET 2025
   Tue Feb 18 12:18:14 CET 2025
   =127sec
 </pre>
* Mac mini M4 2025
  <pre>
  Tue Feb 18 12:14:07 CET 2025 
  Tue Feb 18 12:14:24 CET 2025
  =17s
</pre>

model large-v3
=
* Mac mini M4 2025
<pre>
Tue Feb 18 12:39:03 CET 2025
Tue Feb 18 12:40:10 CET 2025
  =67s
</pre>
* iMac 2012 Intel(R) Core(TM) i5-2500S CPU @ 2.70GHz
<pre>
Tue Feb 18 12:58:17 CET 2025
Tue Feb 18 13:04:44 CET 2025
  =387sec
</pre>

discussion
=
* even a raspberry pi 5 can be used if you get less than 7min of voicemail per hour (factor six conversion time)
* an old desktop computer provides better performance (factor 3) but consumes much more energy
* a new fast machine like the M4 can convert voicemail in realtime even for two voice lines
* model large-v3 can only be used on fast machines like the M4 (factor 2 conversion time)
* I found no benefit in the error rate of using large-v3 instead of the lage-v3-turbo model, so turbo is sufficient
