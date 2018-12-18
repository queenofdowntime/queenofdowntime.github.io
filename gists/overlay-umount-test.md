---
layout: page
---

## Overlay umount reproduction script

```sh
#!/bin/sh

mountandwrite() {
        i=$1
        echo "mounting $i"
        mount -t overlay -o lowerdir=./lower,upperdir=./upper$i,workdir=./work$i overlay target$i

        echo "writing $i"
        dd if=/dev/urandom of=target$i/foo bs=10M count=10
}

unmount() {
        i=$1
        echo "unmounting $i"
        start_time=$(date +"%s.%N")

        umount target$i

        end_time=$(date +"%s.%N")
        result="$(echo $end_time-$start_time | bc)"
        echo "$i took $result seconds to unmount"
}


mkdir lower

for i in `seq 1 10`; do
        mkdir upper$i work$i target$i
        mountandwrite $i
done

for i in `seq 1 10`; do
        unmount $i &
done
```
