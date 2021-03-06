# Tools for working with a NetDimm

This repository started when I looked at triforcetools.py and realized that it was ancient (Python2) and contained a lot of dead code. So, as one of the first projects I did when setting up my NNC to netboot was to port triforcetools.py to Python3, clean up dead code and add type hints. I also added percentage display for transfer and `--help` to the command-line. This has been tested on a Naomi with a NetDimm, but has not been verified on Triforce/Chihiro. There is no reason why it should not work, however.

## Setup Requirements

This requires at least Python 3.6, and a few packages installed. To install the required packages, run the following. You may need to preface with sudo if you are installing into system Python.

```
python3 -m pip install -r requirements.txt
```

## Script Invocation

### netdimm_send

This script handles sending a single binary to a single cabinet, assuming it is on and ready for connection. Invoke the script like so to see options:

```
python3 -m scripts.netdimm_send --help
```

You can invoke it identically to the original triforcetools.py as well. Assuming your NetDimm is at 192.168.1.1, the following will load the ROM named `my_favorite_game.bin` from the current directory:

```
python3 -m scripts.netdimm_send 192.168.1.1 my_favorite_game.bin
```

As well as sending a single game to a Net Dimm, this script can optionally handle applying patches. See `--help` for more details.

### netdimm_ensure

This script will monitor a cabinet, and send a single binary to that cabinet whenever it powers on. It will run indefinitely, waiting for the cabinet to power on before sending, and then waiting again for the cabinet to be powered off and back on again before sending again. Invoke the script like so to see options:

```
python3 -m scripts.netdimm_ensure --help
```

It works identically to netdimm_send, except for it only supports a zero PIC, and it tries its best to always ensure the cabinet has the right game. Run it just like you would netdimm_send. Just like `netdimm_send`, this script can optionally handle applying patches. See `--help` for more details.

### binary_patch

This script can either diff two same-length binaries and produce a patch similar to the files found in `patches/` or it can take a binary and one or more patch files and produce a new binary with the patches applied. Invoke the script like so to see options:

```
python3 -m scripts.binary_patch --help
```

To diff two binary files, outputting their differences to a file, run like so:

```
python3 -m scripts.binary_patch diff file1.bin file2.bin --patch-file differences.binpatch
```

To apply a patch to a binary file, outputting it to a new file, run like so:

```
python3 -m scripts.binary_patch patch file.bin newfile.bin --patch-file differences.binpatch
```

### rominfo

This script will read a rom and output information found in the header. Currently it only supports Naomi ROMs but it can be extended to Triforce and Chihiro ROMs as well if somebody wants to put in the effort. Invoke the script like so to see options:

```
python3 -m scripts.rominfo --help
```

To output information about a particular binary, run like so:

```
python3 -m scripts.rominfo somefile.bin
```

### attach_sram

This script will take an SRAM file from an emulator such as demul and attach it to an Atomiswave conversion game so that your Naomi initializes the SRAM with its contents. If an Atomiswave conversion ROM already has an SRAM initialization section, it will overwrite it with the new SRAM. Otherwise, it enlarges the ROM to make room for the init section. Use this to set up defaults for a game using the test menu in an emulator and apply those settings to your game for netbooting with chosen defaults. Invoke the script like so to see options:

```
python3 -m scripts.attach_sram --help
```

To attach a SRAM file from demul to a ROM named demo.bin, run like so:

```
python3 -m scripts.attach_sram demo.bin dummy.sram
```

### Free-Play/No Attract Patch Generators

Both `scripts.make_freeplay_patch` and `scripts.make_no_attract_patch` can be invoked in the same manner, and will produce a patch that applies either forced free-play or forced silent attract mode. You can run them like so:

```
python3 -m scripts.make_no_attract_patch game.bin --patch-file game_no_attract.binpatch
python3 -m scripts.make_freeplay_patch game.bin --patch-file game_freeplay.binpatch
```

You can also see options available for running by running it with the `--help` option:

```
python3 -m scripts.make_no_attract_patch --help
python3 -m scripts.make_freeplay_patch --help
```

## Web Interface

Along with scripts, several libraries and a series of patches, this repository also provides a simple web interface that can manage multiple cabinets. This is meant to be run on a server on your local network and will attempt to boot your Naomi to the last game selected on every boot. Use this if you want to treat your netboot setup much like a cabinet with a legitimate cartridge in it, but still allow yourself and friends to select a new game easily.

To try it out with the test server run the following:

```
python3 -m scripts.host_debug_server --config config.yaml
```

Like the other scripts, you can run this command with the `--help` flag to see additional options. By default, the config will look for roms in the `roms/` directory, and look for patches in the `patches/` directory. It will listen on port 80. You do not want to run the above script to serve traffic on a production setup as it is single-threaded and will dump its caches if you change any files. By default, the server interface will hide advanced options, such as cabinet configuration. To show the options, either type the word 'config' on the home page, or go to the `/config` page. Both of these will un-hide the configuration options until you choose again to hide them.

### Production Setup

Since this uses a series of threads to manage cabinets in the background, it can be somewhat difficult to install in a normal nginx/uWSGI setup. A wsgi file is provided in the `scripts/` directory that is meant to be used alongside a virtualenv with this repository installed in it. Config files should be copied to the same directory, and don't forget to update the ROMs/patches directories. If you do something similar to my setup below, don't forget to set up a virtualenv in the venv directory!

My uWSGI config looks something similar to the following:

```
[uwsgi]
plugins = python3
socket = /path/to/wsgi/directory/netboot.sock
processes = 1
threads = 2
master = false
enable-threads = true

virtualenv = /path/to/wsgi/directory/venv
chdir = /path/to/wsgi/directory/
wsgi-file = server.wsgi
callable = app
```

My nginx config looks something like the following:

```
server {
    server_name your.domain.here.com;
    listen 443 ssl;
    server_tokens off;

    gzip on;
    gzip_types text/html text/css text/plain application/javascript application/xml application/json;
    gzip_min_length 1000;

    ssl_certificate /path/to/letsencrypt/fullchain.pem;
    ssl_certificate_key /path/to/letsencrypt/privkey.pem;

    location / {
        include uwsgi_params;
        uwsgi_pass unix:/path/to/wsgi/directory/netboot.sock;
    }

    location ^~ /static/ {
        include  /etc/nginx/mime.types;
        root /path/to/wsgi/directory/venv/lib/python3.6/site-packages/netboot/web/;
    }
}
```

With the above scripts, you should be able to visit `https://your.domain.here.com/`. The running uWSGI server will also manage your cabinets for you so there is no need to also use `netdimm_ensure` or any other scripts.

## Developing

The tools here are fully typed, and should be kept that way. To verify type hints, run the following:

```
mypy --strict .
```

The tools are also lint clean (save for line length lints which are useless drivel). To verify lint, run the following:

```
flake8 --ignore E501 .
```
