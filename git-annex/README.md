
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
  * repozytorium *hostC - backup-server* (ssh/https remote)

## Uruchomienie kontenerów

```
docker-compose up
```

## Konfiguracja `hostA`:


```bash
docker-compose exec --user git hostA /bin/bash

cd
git init backup-server.git --bare --shared
cd backup-server.git/
git annex init 'hostA - backup-server'
```

## Konfiguracja `hostB`:

TODO do przemyslenia:

* moze lepiej nie klonowac tylko zrobic nowe repozytoria
  (remote origin i tak nie uzywamy)
* nie inicjalizowac annexa w backup-server i backup-disk
  (pracujemy tylko w backup-local) ??
* wszystkie poza backup-local powinny byc --bare

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
