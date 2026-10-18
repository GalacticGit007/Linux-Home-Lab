# Lot of dump files kept creating in my /home directory

Lot of files where keep filling my device with names like  core.134543.

### What to do when it happen again.

First find out which partition is full or taking up the space

``` bash 
df -h
```

Trace where the massive data buildup is happening (starting at root / or your home directory.

``` bash
sudo du -ahx --max-depth=1 / | sort -rh | head -n 10
```

If you see thousands of files with name core.12345 or generic names inspect and find out which program generate it.

``` bash
file ~/core.340218
```

* If it says core file: An app is crashing repeatedly. 
* If it says ASCII text: A log file is runaway printing errors (check /var/log/).

Batch delete the file safely

``` bash
find . -maxdepth 1 -name "core.*" -exec rm -f {} +
```
If the file keep generating find the program and kill it.

``` bash
sudo lsof /path/to/runaway.log
```

Find PID and kill the app make sure you are not killing a critical system service.

```bash
ps -fp <PID>
 ```

If the app is running and it is a single file you can use truncate to resize the file.

``` bash
sudo truncate -s 0 /path/to/runaway.log
 ```

