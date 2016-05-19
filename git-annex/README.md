
## Wymagane paczki:

* git-1.7.1-4.el6_7.1.x86_64
* rsync-3.0.6-12.el6.x86_64
* openssh-server-5.3p1-114.el6_7.x86_64
* git-annex-3.20120522-2.1.el6.x86_64

## Setup

* `hostA`
  * repozytorium *hostA - backup-server* (ssh remote)
* `hostB`
  * repozytorium *hostB - backup-local* (repozytorium w ktorym pracujemy)
  * repozytorium *hostB - backup-server* (ssh remote)
  * repozytorium *hostB - backup-disk* (local remote)
* `gitlab.com` (opcjonalnie)
  * repozytorium *hostC - backup-server* (https remote)

## Uruchomienie kontenerów

```
docker-compose up
```

## Konfiguracja `hostA`:


```bash
docker-compose exec --user git hostA /bin/bash

cd
git init backup-server.git --shared
cd backup-server.git/
git annex init 'hostA - backup-server'
```

## Konfiguracja `hostB`:

TODO do przemyslenia:

* moze lepiej nie klonowac tylko zrobic nowe repozytoria
  (remote origin i tak nie uzywamy)
  * Nie ma to znaczenia specjalnie. I tak jeśli to już 2+ repo, to najpierw
    trzeba będzie dodać remote'y i z nich ściągnąć historię - co jest
    jednoznaczne z clone'em.
* nie inicjalizowac annexa w backup-server i backup-disk
  (pracujemy tylko w backup-local) ??
  * Inicjalizacja powinna być wszędzie. Każde repo powinno mieć obsługę annexa.
* wszystkie poza backup-local powinny byc --bare
  * Wręcz przeciwnie, wszystkie powinny być zwykłymi repozytoriami. No może
    oprócz tego w gitlab.com.


```bash
docker-compose exec --user git hostB /bin/bash

cd
git clone git@hostA:/home/git/backup-server.git backup-local
cd backup-local/
git annex init 'hostB - backup-local'

cd
git clone git@hostA:/home/git/backup-server.git backup-disk
cd backup-disk/
git annex init 'hostB - backup-disk'

cd
git clone git@hostA:/home/git/backup-server.git backup-server
cd backup-server/
git annex init 'hostB - backup-server'

cd ~/backup-local/
git remote add backup-server-a git@hostA:/home/git/backup-server.git
git remote add backup-server-b git@hostB:/home/git/backup-server
git remote add backup-disk ~/backup-disk
```

## Test

```bash
cd ~/backup-local/
echo "my-test-file" > test.txt
git annex add test.txt
ll
# total 4
# lrwxrwxrwx 1 git git 178 May 15 13:18 test.txt -> .git/annex/objects ...
git add -A .
git commit -m "test.txt added"

cd ../backup-disk/
git pull origin master
ll
# total 4
# lrwxrwxrwx 1 git git 178 May 15 16:03 test.txt -> .git/annex/objects ...
cat test.txt
# cat: test.txt: No such file or directory
git annex get test.txt
# (not available)
#   No other repository is known to contain the file.
# failed
# git-annex: get: 1 failed

cd ../backup-local/
git annex whereis test.txt
# whereis test.txt (2 copies)
#    	e5c1bc2a-141f-4184-ad00-6414a604f8a0 -- here (hostB - backup-local)
# ok
git annex copy --to backup-server-a test.txt
git annex whereis test.txt
# whereis test.txt (2 copies)
#   	675249ea-3f19-40e3-8a5d-f06b94bafff7 -- origin (hostA - backup-server)
#    	e5c1bc2a-141f-4184-ad00-6414a604f8a0 -- here (hostB - backup-local)
# ok

cd ../backup-disk/
git pull origin
git annex whereis test.txt
# (merging origin/git-annex into git-annex...)
# whereis test.txt (1 copy)
#   	675249ea-3f19-40e3-8a5d-f06b94bafff7 -- origin (hostA - backup-server)
# ok
git annex get test.txt
cat test.txt
# my-test-file
```

## Mozliwe scenariusze na laborke

* archiwizujemy jakis katalog/dysk - robimy archiwum czy zapisujemy
  bezposrednio ?;
* mamy jedno repozytorium (robocze) gdzie recznie wrzucamy / synchronizujemy
  pliki, pozostale repozytoria to backupy;
* z tego roboczego mozna usuwac pliki po przeniesieniu ich do ktoregos
  z repozytoriow zdalnych;
* mozna ustawic minimalna liczbe kopi na dwa lub wiecej - nadmiarowosc +
  scenariusz gdzie wylaczamy jedno repozytorium backupowe i plik jest
  kopiowany do innego
* niektore repozytoria mozna umiescic w ograniczonym systemie plikow
  jakas quota lub rozmiar np. <1GB i sprawdzac jak zareaguje git i annex - tj.
  czy czesc plikow bedzie w innym mirrorze
* wszelkie pomiary wydajnosci - transfery do repozytorium po ssh przez siec,
  transfery przez ssh@localhost, transfery do repo w lokalnym systemie plikow,
  transfery do np. gitlaba (jest tutorial na stronie git-annexa, gitlab daje
  10GB na gitlab.com dla darmowych kont)
* troche czasu zejdzie na zainstalowanie i konfiguracje openssh, kluczy, gita

## Możliwy przebieg laborki

1. Skonfiguruj dwie maszyny wirtualne centos6 z zainstalowanymi odpowiednimi
   paczkami.
   Maszyny niech będą dostępne dla siebie pod nazwami `hostA` i `hostB`.

   ```bash
   docker-compose up
   docker-compose exec --user git hostA /bin/bash
   git@hostA:~$ ssh-keygen
   git@hostA:~$ cat 'git' | ssh-copy-id git@hostB
   docker-compose exec --user git hostB /bin/bash
   git@hostB:~$ ssh-keygen
   git@hostB:~$ cat 'git' | ssh-copy-id git@hostA
   ```

2. Utwórz na obu maszynach repozytoria `hostA-main` i `hostB-main` (gdzie
   prefiks oznacza maszynę, na której dane repozytorium powinno się znajdować).
   Remote'y maszyn powinny nawzajem na siebie wskazywać.

   ```bash
   git@hostA:~$ git init hostA-main.git
   git@hostA:~$ cd hostA-main.git
   git@hostA:hostA-main.git$ git annex init 'hostA - main'

   git@hostB:~$ git init hostB-main.git
   git@hostB:~$ cd hostB-main.git
   git@hostB:hostB-main.git$ git annex init 'hostB - main'
   git@hostB:hostB-main.git$ git remote add hostA-main git@hostA:hostA-main.git
   git@hostB:hostB-main.git$ git pull hostA-main master # nie zadziala dopoki nie ma commitow na master

   git@hostA:hostA-main.git$ git remote add hostB-main git@hostB:hostB-main.git
   ```

3. W repozytorium hostA-main utwórz plik 'important_file.txt' i dodaj go do
   indeksu git-annex.
   Uzyskaj dostęp do zawartości pliku na maszynie hostB.
   Zweryfikuj gdzie znajduje się aktualnie plik z hostA i hostB.

   ```bash
   git@hostA:hostA-main.git$ echo "important big file" > important_file.txt
   git@hostA:hostA-main.git$ git annex add important_file.txt
   git@hostA:hostA-main.git$ git commit -m "important_file"

   git@hostB:hostB-main.git$ git annex sync
   git@hostB:hostB-main.git$ ll
   total 4
   lrwxrwxrwx 1 git git 178 May 19 10:18 important_file.txt -> .git/annex/objects/3J/Z8/SHA256-s19--ac2b10210a29d0275fec27792ecbc2b641669cfc76b0a36b2a7062de274ecf31/SHA256-s19--ac2b10210a29d0275fec27792ecbc2b641669cfc76b0a36b2a7062de274ecf31
   git@hostB:hostB-main.git$ cat important_file.txt
   cat: important_file.txt: No such file or directory
   git@hostB:hostB-main.git$ git annex get important_file.txt
   git@hostB:hostB-main.git$ cat important_file.txt
   important big file
   git@hostB:hostB-main.git$ git annex whereis important_file.txt
   whereis important_file.txt (2 copies)
       bad498fb-e968-467a-98a8-c59fd7800932 -- hostA-main (hostA - main)
       f0c9f4ba-7550-40ce-9eb0-a325b991d306 -- here (hostB - main)
   ok

   git@hostA:hostA-main.git$ git annex whereis important_file.txt
   whereis important_file.txt (1 copy)
       bad498fb-e968-467a-98a8-c59fd7800932 -- here (hostA - main)
   ok
   git@hostA:hostA-main.git$ git annex sync
   git@hostA:hostA-main.git$ git annex whereis important_file.txt
   whereis important_file.txt (2 copies)
       bad498fb-e968-467a-98a8-c59fd7800932 -- here (hostA - main)
       f0c9f4ba-7550-40ce-9eb0-a325b991d306 -- hostB-main (hostB - main)
   ok
   ```

4. Zmodyfikuj zawartość pliku w hostB-main i zweryfikuj jak rozprzestrzeniają
   się zmiany.

   ```bash
   git@hostB:hostB-main.git$ echo " modified" >> important_file.txt
   bash: important_file.txt: Permission denied
   git@hostB:hostB-main.git$ git annex unlock important_file.txt
   git@hostB:hostB-main.git$ ll
   total 4
   -rw-r--r-- 1 git git 20 May 19 10:55 important_file.txt
   git@hostB:hostB-main.git$ echo " modified" >> important_file.txt
   git@hostB:hostB-main.git$ git commit important_file.txt -m "Modified important_file.txt"
   git@hostB:hostB-main.git$ ll
   total 4
   lrwxrwxrwx 1 git git 178 May 19 10:55 important_file.txt -> .git/annex/objects/wG/2j/SHA256-s29--ac1d39fe5ee700d3b1bd2b464fd4dc47b847a4e8dd6d796451ba5ca836cce2e3/SHA256-s29--ac1d39fe5ee700d3b1bd2b464fd4dc47b847a4e8dd6d796451ba5ca836cce2e3
   git@hostB:hostB-main.git$ git annex whereis important_file.txt
   whereis important_file.txt (1 copy)
       f0c9f4ba-7550-40ce-9eb0-a325b991d306 -- here (hostB - main)
   ok

   git@hostA:hostA-main.git$ git annex whereis important_file.txt
   whereis important_file.txt (2 copies)
       bad498fb-e968-467a-98a8-c59fd7800932 -- here (hostA - main)
       f0c9f4ba-7550-40ce-9eb0-a325b991d306 -- hostB-main (hostB - main)
   ok
   git@hostA:hostA-main.git$ git annex sync
   git@hostA:hostA-main.git$ git annex whereis important_file.txt
   whereis important_file.txt (1 copy)
       f0c9f4ba-7550-40ce-9eb0-a325b991d306 -- hostB-main (hostB - main)
   ok
   ```

5. Utwórz kolejne repozytorium hostB-media.
   Może to być repozytorium, które przechowywane będzie na przykład na napędzie USB.

   ```bash
   git@hostB:~$ git init hostB-media.git
   git@hostB:~$ cd hostB-media.git
   git@hostB:hostB-media.git$ git annex init 'hostB - media'
   git@hostB:hostB-media.git$ git remote add hostB-main /home/git/hostB-main.git
   git@hostB:hostB-media.git$ git pull hostB-main master
   git@hostB:hostB-media.git$ git annex sync

   git@hostB:hostB-main.git$ git remote add hostB-media /home/git/hostB-media.git
   git@hostB:hostB-main.git$ git annex sync
   ```

6. Utwórz plik o wielkości 500MB w hostB-main.
   Zbadaj szybkość transferu do hostA-main i hostB-media.

   ```bash
   git@hostB:hostB-main.git$ dd if=/dev/zero of=large.bin bs=1024 count=500000
   git@hostB:hostB-main.git$ git annex add large.bin
   git@hostB:hostB-main.git$ git commit -m "Added large.bin"
   git@hostB:hostB-main.git$ git annex copy large.bin hostB-media
   copy large.bin (to hostB-media...) ok
   (Recording state in git...)
   git@hostB:hostB-main.git$ git annex copy large.bin hostA-main
   copy large.bin (checking hostA-main...) (to hostA-main...)
   SHA256-s512000000--4c98cf638799a07eb85872e7f5f5d8d661c09e6e15af02f3655ed9250cae13b0
      512000000 100%   83.22MB/s    0:00:05 (xfer#1, to-check=0/1)

   sent 512062644 bytes  received 31 bytes  78778873.08 bytes/sec
   total size is 512000000  speedup is 1.00
   ok
   ```

7. Zbadaj działanie flagi numcopies.
   Zwróć uwagę na to, że wersja git-annex dostępna w repozytorium centos6 nie
   implementuje jeszcze polecenia `numcopies`.

   Dodaj wpis `numcopies = 2` w sekcji [annex] w .git/config.
   ```bash
   git@hostB:hostB-main.git$ git config annex.numcopies 2 # konfig sie nie propaguje przy sync-u
   git@hostB:hostB-main.git$ git annex copy important_file.txt --to hostA-main
   git@hostB:hostB-main.git$ git annex drop important_file.txt
   drop important_file.txt (checking hostA-main...) (unsafe)
     Could only verify the existence of 1 out of 2 necessary copies

     No other repository is known to contain the file.

     (Use --force to override this check, or adjust annex.numcopies.)
   failed
   git-annex: drop: 1 failed
   ```
