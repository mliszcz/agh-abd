# lab02

* RAID 0 consists of striping, without mirroring or parity.
* RAID 1 consists of data mirroring, without parity or striping.
* Poziom piąty pracuje bardzo podobnie do poziomu czwartego z tą różnicą, iż
  bity parzystości nie są zapisywane na specjalnie do tego przeznaczonym dysku,
  lecz są rozpraszane po całej strukturze macierzy. RAID 5 umożliwia odzyskanie
  danych w razie awarii jednego z dysków przy wykorzystaniu danych i kodów
  korekcyjnych zapisanych na pozostałych dyskach (zamiast tak jak w 3. na
  jednym specjalnie do tego przeznaczonym, co nieznacznie zmniejsza koszty
  i daje lepsze gwarancje bezpieczeństwa).

  Macierz składa się z 3 lub więcej dysków. Przy macierzy liczącej N dysków
  jej objętość wynosi N – 1 dysków. Przy łączeniu dysków o różnej pojemności
  otrzymujemy objętość najmniejszego dysku razy N – 1. Sumy kontrolne danych
  dzielone są na N części, przy czym każda część składowana jest na innym
  dysku, a wyliczana jest z odpowiedniego fragmentu danych składowanych na
  pozostałych N-1 dyskach.

## Zidentyfikować pliki specjalne dysków

```
ll /dev/disk/by-id
sudo lshw -class disk
lsblk -io KNAME,TYPE,SIZE,MODEL
sudo fdisk -l
sudo parted -l
```

## Wykasować tablice partycji na tych dyskach

`sudo fdisk /dev/sdc`

```
Command (m for help): o
Building a new DOS disklabel with disk identifier 0x75aa352c.
Changes will remain in memory only, until you decide to write them.
After that, of course, the previous content won't be recoverable.

Warning: invalid flag 0x0000 of partition table 4 will be corrected by w(rite)

Command (m for help): w
The partition table has been altered!
```

```
To format just your MBR
dd if=/dev/zero of=/dev/sda bs=446 count=1

To format (remove) your MBR & partition table
dd if=/dev/zero of=/dev/sda bs=512 count=1
```

```
Inform kernel about this change by partprobe commands and verify with fdisk -l commands
```

## Wykonać pomiary wydajności dysków. Odczyt - za pomocą hdparm, zapis - za pomocą dd, źródło - /dev/zero,
rozmiar danych 500MB. Wyniki pomiarów zapisać do poniższej tabeli

```
dd if=/dev/zero of=/dev/sdx bs=1048576 count=500
```

Using hdparm with the -Tt switch, one can time sequential reads. This method is independent of partition alignment!

```
$ sudo hdparm -tT /dev/sdc

/dev/sdc:
Timing cached reads:   9970 MB in  2.00 seconds = 4987.88 MB/sec
Timing buffered disk reads: 712 MB in  3.00 seconds = 237.21 MB/sec
```

## stworzyć partycje o rozmiarze 1 GB, na każdym dysku z półki JBOD

```
Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-31277231, default 2048): 2048
Last sector, +sectors or +size{K,M,G} (2048-31277231, default 31277231): +1G
```


## skonfigurować macierz RAID0 na 2,3,4 dyskach i wykonać pomiar wydajności. Wypełnić tabelę poniżej

mdadm - manage MD devices aka Linux Software RAID

https://raid.wiki.kernel.org/index.php/RAID_setup
http://computernetworkingnotes.com/disk-managements/raid.html

```
mdadm --create /dev/md0 --level=5 --raid-disk=3 /dev/sda6 /dev/sda7 /dev/sda8
```
