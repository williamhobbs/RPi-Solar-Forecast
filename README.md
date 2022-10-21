# How to use a Rasberry Pi with SolarForecastArbiter to generate solar forecasts
Will Hobbs
2021-07-22

***
**2022-10-21 Update:** [solarforecastarbiter.org](https://solarforecastarbiter.org) is now [forecastarbiter.epri.com](https://forecastarbiter.epri.com). This guide needs to be updated accordingly, and new user accounts may not be available currently. Everything related to fetching NWPs and creating forecasts should still work, it's just that uploading forecasts to the arbiter may not work.

***

**2022-02-03 Update:** Now that there is an official 64-bit Raspberry PI OS available (https://www.raspberrypi.com/news/raspberry-pi-os-64-bit/), I should try that out and update this guide accordingly. There may be some advantages to 64-bit...
***
To-do list: 
- Manage disk space issues elegantly. Current custom domain I'm using for HRRR results in about 14 GB/month of disk space, and when the SD card fills up, all NWP fetching stops. Could try:
	- alerts?
	- automatically overwrite old .nc files?
	- move old files to external drive/network drive/cloud?
- Try out 64-bit OS
***

# Table of contents
1. [Introduction](#introduction)
2. [Setup and Installation](#setup_and_installation)
   1. [Get ready](#get_ready)
   2. [Install some stuff we will need later](#install_some_stuff)
   3. [Compile wgrib2](#wgrib2)
   4. [Install solarforecastarbiter](#install_sfa)
3. [Numerical Weather Prediction models (NWPs)](#nwps)
   1. [Get setup for fetching NWP data](#fetch_setup)
   2. [Start fetching NWP data](#fetch)
   3. [Make sure we keep fetching](#keep_fetching)
4. [Solar Forecasts](#solar_forecasts)
   1. [Create forecasts on solarforecastarbiter.org](#create_forecasts)
   2. [Python script to generate and upload forecasts](#upload_forecasts)
   3. [Run the python forecast script on a schedule](#scheduled_forecasts)
   4. [The end](#the_end)
5. [Extras](#extras)
   1. [Ideas for improvement](#ideals)
   2. [Fetch GFS for day-ahead forecasts](#fetch_gfs)
   3. [Fetch HRRR Hourly out to 48 hours](#48hr_hrrr)
   4. [Backup the micro SD card periodically](#sd_backup)


## Introduction <a name="introduction"></a>
This should be considered a "for fun" exercise, unless you are familiar with running a Raspberry Pi for important tasks (and know how to manage/avoid things like SD card corruption).

Tested with:

 - OS:
	 - Raspberry Pi OS (32-bit) (with desktop)
	 - Raspbian GNU/Linux 10 (buster) 
	 - Released: 2021-05-07
 - Hardware:
	 - Rapsberry Pi 4 Model B 8G
	 - 64 GB micro SD card (SanDisk Ultra Class 10)
	 - an external HD, network drive, etc., may be needed for storing NWP files long-term

It's recommended to start with a fresh OS install using Raspberry Pi Imager to setup your micro SD card with another computer. See: https://www.raspberrypi.org/software/.

Other prerequisites:
 1. have an account on solarforecastarbiter.org
 2. have a site setup so you can create (register) forecasts that match the forecast timeseries we will be generating

## Setup and Installation <a name="setup_and_installation"></a>

### Get ready <a name="get_ready"></a>
 1. Boot up your RPi
 2. Consider: setting a new password, enabling VNC and/or installing XRDP
 3. Start with standard RPi update and upgrade. From the terminal, run:  
```bash
sudo apt update 
sudo apt full-upgrade
```
### Install some stuff we will need later <a name="install_some_stuff"></a>
Get ready for netcdf ([ref](get%20ready%20for%20netcdf4,%20from%20https://raspberrypi.stackexchange.com/questions/104791/installing-netcdf4-on-raspberry-3b)):
```bash
sudo apt-get install -y libhdf5-dev libhdf5-serial-dev netcdf-bin libnetcdf-dev
```
Get ready for numpy ([ref](https://stackoverflow.com/questions/53784520/numpy-import-error-python3-on-raspberry-pi)):
```bash
sudo apt-get install -y python-dev libatlas-base-dev
```
### Python environment setup and package installation
Create a virtual environment with `venv`. We will call it `test_venv`:
```bash
python3 -m venv test_venv
```
Activate the virtual environment to make sure it works:
```bash
source test_venv/bin/activate
```
To deactivate the virtual environment, use:
```bash
deactivate
```
But we want to stay in it for now, so re-enter:
```bash 
source test_venv/bin/activate
```

Now let's check which packages we have with:
```bash
pip3 list
```
It should be a short list. Let's fix that. First install `wheel`:
```bash
pip3 install wheel
```

Then grab a text file with the list of other requirements:
```bash
wget https://raw.githubusercontent.com/SolarArbiter/solarforecastarbiter-core/master/requirements.txt
```
And install them:
```bash
pip3 install -r requirements.txt
```
Check the package list again:
```bash
pip3 list
```
and deactivate the virtual environment:
```bash
deactivate
```

### Compile wgrib2 <a name="wgrib2"></a>
wgrib2 is a tool from NOAA for processing grib2 files. Included in the description on [NOAA's site](https://www.cpc.ncep.noaa.gov/products/wesley/wgrib2/): *wgrib can slice and dice grib1 files. wgrib2 is more like four drawers of kitchen utensils as well as the microwave and blender.* We will use it to convert grib2 files to netcdf files. 
Since wgrib2 is mostly intended for x86/Intel Linux, and the Raspberry Pi has an ARM processor, we need to customize a few things and compile an executable file.
First, some setup (modified from [ref](https://github.com/SolarArbiter/solarforecastarbiter-core/blob/23ac3c7a699f720fb642c3cc7d1acb83b486c2a6/build/Dockerfile) and [ref](https://askubuntu.com/questions/1132099/ungrib-exe-usr-lib-x86-64-linux-gnu-libpng12-so-0-version-png12-0-not-fou)):
```bash
sudo apt-get install -y gfortran libgrib2c-dev python3-gpg libnetcdf13 
sudo apt install -y m4
```
Then make sure we are in `/home/pi`:
```bash
cd
```
Get the compressed source, extract it, and go to that directory:
```bash
wget ftp://ftp.cpc.ncep.noaa.gov/wd51we/wgrib2/wgrib2.tgz
tar -xvf wgrib2.tgz
cd grib2
```
Add netcdf4 and hdf5 source tar.gz files to directory:
```bash
wget ftp://ftp.unidata.ucar.edu/pub/netcdf/netcdf-c-4.7.3.tar.gz
wget https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.10/hdf5-1.10.4/src/hdf5-1.10.4.tar.gz
```
Since the RPi has an ARM processor, we need to modify the makefile. See [here](https://www.cpc.ncep.noaa.gov/products/wesley/wgrib2/) for some details. Open the file `makefile`, either by finding it in the File Manager and opening with Text Editor (right-click the file and select "Text Editor"), or run `nano makefile` in the Terminal to open it with the nano editor.  Scroll down to the text that looks like this:
```
\# Warning do not set both USE_NETCDF3 and USE_NETCDF4 to one
USE_NETCDF3=1
USE_NETCDF4=0
USE_REGEX=1
USE_TIGGE=1
```
[...]

```
MAKE_SHARED_LIB=0

USE_G2CLIB=0
USE_PNG=1
USE_JASPER=1
USE_OPENJPEG=0
```
And set `USE_NETCDF3=0`, `USE_NETCDF4=1`, and `USE_JASPER=0`. It seems to compile and work with `USEJASPER=1`, but [NOAA's site](https://www.cpc.ncep.noaa.gov/products/wesley/wgrib2/) says to use 0 when compiling for ARM. Save and close the file (CTRL-S, CTRL-X in nano). 

Next, set some environment variables ([ref](https://www.ftp.cpc.ncep.noaa.gov/wd51we/wgrib2/INSTALLING)):
```bash
export CC=gcc
export FC=gfortran
```
and compile wgrib2 using gnu make with the command:
```bash
make
```
Now wait up to 1 or 2 hours for wgrib2 to compile, hopefully with no errors (some warnings along the way are expected). Go for a walk, make some coffee, etc...

Try executing wgrib2:
```'bash
wgrib2/wgrib2 -config
```
It should return something that looks like:
```
wgrib2 v3.0.2 3/2021  Wesley Ebisuzaki, Reinoud Bokhorst, John Howard, Jaakko Hyvätti, Dusan Jovic, Daniel Lee, Kristian Nilssen, Karl Pfeiffer, Pablo Romero, Manfred Schwarb, Gregor Schee, Arlindo da Silva, Niklas Sondell, Sam Trahan, George Trojan, Sergey Varlamov
    stock build

Compiled on 17:40:06 Jul 13 2021

Netcdf package: 4.7.3 of Jul 13 2021 17:36:31 $ is installed
hdf5 package: hdf5-1.10.4.tar.gz is installed
...
```

Now copy the resulting wgrib2 binary file to a reasonable place on the standard path, like:
```bash
sudo cp wgrib2/wgrib2 /usr/local/bin
```
and return to /home/pi/:
```bash
cd
```

### Install solarforecastarbiter <a name="install_sfa"></a>
Activate the virtual environment:
```bash
source test_venv/bin/activate
```
and install SFA (you can probably use `[fetchnwp]` instead of `[all]`):
```bash
pip3 install solarforecastarbiter[all]
```
You can test the installation with:
```bash
pytest test_venv/lib/python3.7/site-packages/solarforecastarbiter
```
I received 4 errors and many warnings. If you receive a bunch of errors (13 in my case) and many include `ValueError: numpy.ndarray...`, try uninstalling and reinstalling `numpy`:
```
pip3 uninstall -y numpy
pip3 install numpy
```
## Numerical Weather Prediction models (NWPs) <a name="nwps"></a>
Consider subscribing to `NCEP.List.NOMADS-ftpprd` and `NCEP.List.Nomads-Users` mailing lists [here](https://nomads.ncep.noaa.gov/txt_descriptions/Help_Desk_doc.shtml). Among other things, this can provide information about NOMADS issues and outages. 

### Get setup for fetching NWP data <a name="fetch_setup"></a>
Make a directory to store NWP data, e.g., /home/pi/Downloads/sfa 
```bash
mkdir /home/pi/Downloads/sfa
```
To reduce file sizes, download and processing times, and memory requirements,	you can pull NWP data for a small subset of the model domain. Open `nwp.py` in `home/pi/test_venv/lib/python3.7/site-packages/solarforecastarbiter/io` and modify `DOMAIN` to have a much smaller area, e.g., a rectangle covering Mississippi, Alabama, and Georgia:
```py
DOMAIN = {'subregion': '',
          'leftlon': -90.5,
          'rightlon': -80.5,
          'toplat': 35.2,
          'bottomlat': 30}
```
**Note that running the full default NWP domain could lead to memory issues on a 8 GB Raspberry Pi, and will likely cause issues on models with less memory.** 

**Also note that if you do change the domain you will need to change it again if you update `solarforecastcastarbiter-core`.**

### Start fetching NWP data <a name="fetch"></a>
Activate the virtual environment:
```bash
source test_venv/bin/activate
```
And run solararbiter for the HRRR subhourly NWP:
```bash
solararbiter fetchnwp -v /home/pi/Downloads/sfa hrrr_subhourly
```
and the NAM:
```bash
solararbiter fetchnwp -v /home/pi/Downloads/sfa nam_12km
```
We use `-v` for "verbose" to see what is happening, and the directory files are saved to, `/home/pi/Downloads/sfa`, has to match the directory we created in the last section. 

Grib files (.grib2) should start downloading and then be converted to netcdf (.nc) files in subfolders based on NWP model, year, month, day, and initialization hours.

Be patient! A few things to consider:
- it could take 20+ minutes for the first batches to download and get converted
- NOAA's servers may limit the number/rate of requests (see NWS NCEP [Public Information Statement 20-85](https://www.weather.gov/media/notification/pdf2/pns20-85ncep_web_access.pdf))
- if you get errors, start with just one NWP at a time, and try using the `--once` option (e.g., `solararbiter fetchnwp -v --once /home/pi/Downloads/sfa nam_12km`) to pull one forecast initialization time at a time

Even if you don't get errors, NOAA might start returning empty .grib2 files, which could  get converted to .nc files with missing data. NOMADS keeps 1 week or so of old forecast files for some NWPs. To catch up initially, you could run something like this in the terminal (from within the virtual environment):
```bash
x=100; for ((n=0; n < x; n++)); do solararbiter${IFS}fetchnwp${IFS}-v${IFS}--once${IFS}/home/pi/Downloads/sfa${IFS}hrrr_subhourly; sleep 120; done
```
This repeats the command `solararbiter fetchnwp -v --once /home/pi/Downloads/sfa hrrr_subhourly` 100 times, pausing 120 seconds between each run (to try to avoid NOAA download rate limits). The `${IFS}` adds necessary spaces to the command. 

### Make sure we keep fetching <a name="keep_fetching"></a>

We will use `supervisor` ([supervisord.org](http://supervisord.org/)) to start the fetching process and make sure it keeps running. There's a nice introduction to using `supervisor` [here](https://serversforhackers.com/c/monitoring-processes-with-supervisord).

Install `supervisor`:
```bash
sudo apt-get install -y supervisor
```
Start it as a service:
```bash
sudo service supervisor start
```
Go to the folder where `supervisor` looks for configuration files:
```bash
cd /etc/supervisor/conf.d
```
We will create a configuration file named `sfa_fetch_hrrr.conf` for fetching the HRRR subhourly NWP data:
```bash
sudo nano sfa_fetch_hrrr.conf
```
Inside of nano, add the following text:
```bash
[program:sfa_fetch_hrrr]
command=bash -c "sleep 300 && source /home/pi/test_venv/bin/activate && solararbiter fetchnwp -v /home/pi/Downloads/sfa hrrr_subhourly"
startsecs=0
directory=/home/pi/Python/SFA
autostart=true
autorestart=true
startretries=6
stderr_logfile=/home/pi/Python/SFA/sfa_fetch_hrrr.err.log
stdout_logfile=/home/pi/Python/SFA/sfa_fetch_hrrr.out.log
user=pi
```
and save and close (CTRL-S, CTRL-X). 

Then repeat the above steps for the NAM:
```bash
sudo nano sfa_fetch_nam.conf
```
With content:
```bash
[program:sfa_fetch_nam]
command=bash -c "sleep 300 && source /home/pi/test_venv/bin/activate && solararbiter fetchnwp -v /home/pi/Downloads/sfa nam_12km"
startsecs=0
directory=/home/pi/Python/SFA
autostart=true
autorestart=true
startretries=6
stderr_logfile=/home/pi/Python/SFA/sfa_fetch_nam.err.log
stdout_logfile=/home/pi/Python/SFA/sfa_fetch_nam.out.log
user=pi
```

Adding `sleep 300` and `startsecs=0` was based on an answer  [here](https://serverfault.com/questions/608804/how-can-i-configure-supervisord-managed-program-to-wait-x-secs-before-attemting). It should cause things to pause for 300 seconds (5 minute) before restarting after an error, which may help if there is a NOMADS server issue. 

Load the new configuration files with:
```bash
sudo supervisorctl reread 
sudo supervisorctl update
```
Check to see if our programs are running with:
```bash
sudo supervisorctl
```
which should return something like (where the `uptime` values are the time in hh:mm:ss that the program has been running):
```
pi@raspberrypi:~ $ sudo supervisorctl
sfa_fetch_hrrr                   RUNNING   pid 581, uptime 0:09:22
sfa_fetch_nam                    RUNNING   pid 582, uptime 0:09:22
supervisor> 
```
You can exit the `supervisor>` command line with `exit` or CTRL-C. If you edit program configuration files, re-run the `reread` and `update` commands from above. You can also stop and programs with `sudo supervisorctl stop all` and `sudo supervisorctl start all`, or restart them with `sudo supervisorctl restart all`. 

There is also a web interface that lets you monitor, start, and stop `supervisor` processes. **Make sure to read the security warning [here](http://supervisord.org/configuration.html?#inet-http-server-section-settings) about enabling the http server.** To enable it, edit the main configuration file:
```bash
sudo nano /etc/supervisor/supervisord.conf
```
And add text like this at the end of the file:
```
[inet_http_server]
port = 9001
username = sample_username 
password = sample_password
```
Then stop and start the `supervisor` service with:
```bash
sudo service supervisor stop
sudo service supervisor start
```
And navigate to (RPi Local IP address):9001 (e.g., http://192.168.2.98:9001/) in a web browser on the Raspberry Pi or another device on the same network, login using the credentials you set in the configuration file, and you should be able to monitor the processes. 

It appears that `supervisor` only catches the verbose output of `solararbiter -v fetchnwp` as error messages (`stderr`), so everything shows up in the `*.err.log` files, and the "tail" in the web interface doesn't show anything. 

## Solar Forecasts <a name="solar_forecasts"></a>
### Create forecasts on solarforecastarbiter.org <a name="create_forecasts"></a>
 1. Log in to https://dashboard.solarforecastarbiter.org/. 
 2. For one of your sites, create a new forecast. 
 3. Carefully select Forecast Metadata to match the forecast(s) you will be generating. 
 4. Record the UUID(s) of the forecast(s).

### Python script to generate and upload forecasts <a name="upload_forecasts"></a>
Create a python script to generate forecasts and upload to SFA. We will name it `run_ref_fcasts_rev1.py`.  

I recommend starting with two existing jupyter notebooks:
1. reference_forecasts.ipynb from SFA workshop, https://github.com/SolarArbiter/workshop, for examples of generating forecasts
2. data_upload_download.ipynb from the same workshop and site, for examples of uploading data (except we will upload forecasts instead of observations)
(both of these can be run in a virtual machine without installing anything in MyBinder here: https://mybinder.org/v2/gh/SolarArbiter/workshop/master)

In reference_forecasts.ipynb, here are a few pointers:

 - change the base path to match the directory you are putting NWP files, e.g.: `base_path = '/home/pi/Downloads/sfa'`
 - update lat and long to be within DOMAIN defined in `nwp.py` previously
 - update initialization and other times to fit in the NWP files you have fetched and to match the Forecast Metadata you entered on the SFA website
 - update the models to match the NWP models you are fetching (e.g., change `model = models.rap_ghi_to_instantaneous` to `model = models.hrrr_subhourly_to_hourly_mean` for the HRRR)
 - Make sure you assign the correct UUID(s)

And, in general, consider adding some form of logging so you can see what's going on. 

### Run the python forecast script on a schedule <a name="scheduled_forecasts"></a>
We will schedule a cron job to run that script every hour. Since NWP files seem to finish processing each hour at 10-25 min after the hour, we will schedule this at 30 min after the hour (e.g., 12:30, 1:30, etc.).

Open crontab:
```bash
crontab -e
```

The format we will use is:
```
[minute of hr to run] * * * * cd path/to/script && path/to/virtual/env/python3 path/to/script/script_name.py && cd
```
So enter the following text at the end of the file:
```bash
30 * * * * cd /home/pi/Python/SFA && /home/pi/test_venv/bin/python3 /home/pi/Python/SFA/run_ref_fcasts_rev1.py && cd
```
Save and close (CTRL-S, CTRL-X).

### The end <a name="the_end"></a>
Congrats! You can reboot and keep an eye on the Raspberry Pi (to make sure NWP files are getting fetched) and on solarforecastarbiter.org (to make sure forecast data is showing up). 

## Extras <a name="extras"></a>
### Ideas for improvement <a name="ideals"></a>
I'm sure there are many ways to make this better. One area is in scheduling the python forecast script: triggering it once `solararbiter fetchnwp` finishes with a given group of NOAA NWP files would ensure that forecasts are as "fresh" as possible. There is also room for better handling of errors and issues. 

### Fetch GFS for day-ahead forecasts <a name="fetch_gfs"></a>
Fetching GFS and RAP with this setup results in "chunksize" errors for some reason (see this [issue](https://github.com/SolarArbiter/solarforecastarbiter-core/issues/699)). I wanted to look at GFS for day-ahead forecasting, and found that limiting the number of valid hours fetched for a forecast initialization time avoided the error. 

You may want to create a second virtual environment, as we will be modifying the fetch process in ways that could impact the NAM and HRRR fetching. I created one called `test_venv2` and installed a new copy of `solarforecastarbiter` and requirements. 

Open the file `nwp.py` in `/test_venv2/lib.python3.7/site-packages/solarforecastarbiter/io/fetch`. Under `GFS_0P25_1HR`, change:
```py
'valid_hr_gen': lambda x: chain(range(120), range(120, 240, 3),
                                range(240, 385, 12)),
```
to:
```py
'valid_hr_gen': lambda x: chain(range(22, 43, 1)),
```
so we will grab forecasts 22 to 42 hours out, in 1 hour steps (for 21 hours total). Then, further down in the file, under `def _optimize_netcdf(nctmpfile, out_path)`, change
```py
	else:
		chunksizes.append(50)
```
to
```py
	else:
		chunksizes.append(21)
```	

Then create a new `supervisor` program to run that fetch process, and forecast(s) using GFS. I'm not sure why this works, but it seems to fix the problem.

### Fetch HRRR Hourly out to 48 hours <a name="48hr_hrrr"></a>
The current NWP fetch process only grabs HRRR (hourly) out to 36 hours (for cycles where it goes out more than 18 hours), but HRRR has forecasts out to 48 hours (see issue [#793](https://github.com/SolarArbiter/solarforecastarbiter-core/issues/793) on GitHub). See this pull request for details on an update to get hourly forecasts out to 48 hours for relevant cycles: https://github.com/SolarArbiter/solarforecastarbiter-core/pull/794. 

Setup a `supervisor` configuration file to fetch:
```bash
cd /etc/supervisor/conf.d
sudo nano sfa_fetch_hrrr_hourly.conf
```
Put this in the file:
```bash
[program:sfa_fetch_hrrr_hourly]
command=bash -c "sleep 300 && source /home/pi/test_venv/bin/activate && solarar$
startsecs=0
directory=/home/pi/Python/SFA
autostart=true
autorestart=true
startretries=6
stderr_logfile=/home/pi/Python/SFA/sfa_fetch_hrrr_hourly.err.log
stdout_logfile=/home/pi/Python/SFA/sfa_fetch_hrrr_hourly.out.log
user=pi
```
Then load the file and make sure it is running:
```bash
sudo supervisorctl reread 
sudo supervisorctl update
sudo supervisorctl
```
### Backup the micro SD card periodically <a name="sd_backup"></a>
Follow https://www.tomshardware.com/how-to/back-up-raspberry-pi-as-disk-image.

Check the USB drive's mount point path with `lsblk`. In my case it was `/media/pi/CORSAIR`:
```
pi@raspberrypi:~ $ sudo lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda           8:0    0  477G  0 disk 
└─sda1        8:1    0  477G  0 part /media/pi/CORSAIR
mmcblk0     179:0    0 59.5G  0 disk 
├─mmcblk0p1 179:1    0  256M  0 part /boot
└─mmcblk0p2 179:2    0 59.2G  0 part /
```
Get `pishrink` and get it ready:
```
wget https://raw.githubusercontent.com/Drewsif/PiShrink/master/pishrink.sh
sudo chmod +x pishrink.sh
sudo mv pishrink.sh /usr/local/bin
```

Make a backup image:
```
sudo dd if=/dev/mmcblk0 of=/media/pi/CORSAIR/RPi_forecasting_backup_image_20220702.img bs=1M
```

It might take an hour or so, depending on the size of the SD card. 

Then shrink it:
```
cd /media/pi/CORSAIR/
sudo pishrink.sh -z RPi_forecasting_backup_image_20220702.img
