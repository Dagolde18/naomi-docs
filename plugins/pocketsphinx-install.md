---
title: PocketSphinx Install
source: https://github.com/naomiproject/naomi-docs/blob/master/plugins/pocketsphinx-install.md
meta:
  - property: og:title
    content: "Pocketsphinx and Phonetisaurus Guide"
  - property: og:description
    content: Naomi, The privacy focused personal assistant
---

# PocketSphinx setup

These instructions are for installing pocketsphinx on Debian 9 (Stretch). I have also tested on Raspbian Stretch. These instructions should translate to other distros pretty easily. In many cases the package names will be the same, and in other cases you should be able to locate the package by searching for its name or the name of a file within it.

## test the microphone ("hello, can you hear me?")

You want to make sure that the level indicator at the bottom of the screen goes up to about 60% when you are speaking. Use alsamixer to adjust your recording and playback levels.

Also, play it back and make sure the audio does not contain any hissing or popping.

We will use Phonetisaurus to prepare PocketSphinx to transcribe this audio later in these instructions.

```shell
[~]$ sudo apt install alsa-utils

[~]$ alsamixer

[~]$ arecord -vv -r16000 -fS16_LE -c1 -d3 test.wav

[~]$ aplay test.wav
```

If you are on a Raspberry Pi, most likely when you use the arecord command, you will get an error such as "arecord: main:788: audio open error: No such file or directory". This is because the first sound device (card 0) is output only. You will need to specify the recording device. To get a list of recording devices, use "arecord -l". This will return something like this:

```shell
[~]$ arecord -l
**** List of CAPTURE Hardware Devices ****
card 1: Phone [PH USB Speaker Phone], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```

This means that audio card 1, subdevice 0 is capable of recording audio. Usually you will either reference the device as hw:1,0 or plughw:1,0. hw:1,0 accesses the device more directly, while plughw:1,0 includes a translation layer allowing it to be used to record in formats that the device does not support natively. You can use "arecord -L" to see which interfaces are available:

```shell
[~]$ arecord -L
null
    Discard all samples (playback) or generate zero samples (capture)
default:CARD=Phone
    PH USB Speaker Phone, USB Audio
    Default Audio Device
sysdefault:CARD=Phone
    PH USB Speaker Phone, USB Audio
    Default Audio Device
dmix:CARD=Phone,DEV=0
    PH USB Speaker Phone, USB Audio
    Direct sample mixing device
dsnoop:CARD=Phone,DEV=0
    PH USB Speaker Phone, USB Audio
    Direct sample snooping device
hw:CARD=Phone,DEV=0
    PH USB Speaker Phone, USB Audio
    Direct hardware device without any conversions
plughw:CARD=Phone,DEV=0
    PH USB Speaker Phone, USB Audio
    Hardware device with all software conversions
```

Use "-D" to specify the device, and "--list-hw-params" to get more information about what formats the device supports:

```shell
[~]$ arecord -Dhw:1,0 --dump-hw-params
Recording WAVE 'stdin' : Unsigned 8 bit, Rate 8000 Hz, Mono
HW Params of device "hw:1,0":
--------------------
ACCESS:  MMAP_INTERLEAVED RW_INTERLEAVED
FORMAT:  S16_LE
SUBFORMAT:  STD
SAMPLE_BITS: 16
FRAME_BITS: 32
CHANNELS: 2
RATE: 16000
PERIOD_TIME: [1000 8192000]
PERIOD_SIZE: [16 131072]
PERIOD_BYTES: [64 524288]
PERIODS: [2 1024]
BUFFER_TIME: [2000 16384000]
BUFFER_SIZE: [32 262144]
BUFFER_BYTES: [128 1048576]
TICK_TIME: ALL
--------------------
arecord: set_params:1299: Sample format non available
Available formats:
- S16_LE
```

The important bits here are "CHANNELS: 2", "RATE: 16000" and "Available formats: - S16_LE". The rate and format match the format that Naomi expects audio to be captured in, but we need mono audio, not stereo, so we will most likely need to use the plughw version.

```shell
[~]$ arecord -Dhw:1,0 -vv -r16000 -fS16_LE -c1 -d3 test.wav
Recording WAVE 'test.wav' : Signed 16 bit Little Endian, Rate 16000 Hz, Mono
arecord: set_params:1305: Channels count non available

[~]$ arecord -Dplughw:1,0 -vv -r16000 -fS16_LE -c1 -d3 test.wav
Recording WAVE 'test.wav' : Signed 16 bit Little Endian, Rate 16000 Hz, Mono
```

# Install Phonetisaurus

## Build and install openfst

```shell
[~]$ sudo apt install gcc g++ make python-pip autoconf libtool
[~]$ wget http://www.openfst.org/twiki/pub/FST/FstDownload/openfst-1.6.9.tar.gz
[~]$ tar -zxvf openfst-1.6.9.tar.gz
[~]$ cd openfst-1.6.9
[~/openfst-1.6.9]$ autoreconf -i
[~/openfst-1.6.9]$ ./configure --enable-static --enable-shared --enable-far --enable-lookahead-fsts --enable-const-fsts --enable-pdt --enable-ngram-fsts --enable-linear-fsts --prefix=/usr
[~/openfst-1.6.9]$ make
[~/openfst-1.6.9]$ sudo make install
[~/openfst-1.6.9]$ cd
```

## Build and install mitlm-0.4.2

Building mitlm is only necessary because we are training our own fst model a little further on.

```shell
[~]$ sudo apt install git gfortran autoconf-archive
[~]$ git clone https://github.com/mitlm/mitlm.git
[~]$ cd mitlm
[~/mitlm]$ ./autogen.sh
[~/mitlm]$ make
[~/mitlm]$ sudo make install
[~/mitlm]$ sudo ldconfig
[~/mitlm]$ cd
```

## Build and install Phonetisaurus

```shell
[~]$ git clone https://github.com/AdolfVonKleist/Phonetisaurus.git
[~]$ cd Phonetisaurus
[~/Phonetisaurus]$ ./configure --enable-python
[~/Phonetisaurus]$ make
[~/Phonetisaurus]$ sudo make install
[~/Phonetisaurus]$ cd python
[~/Phonetisaurus/python]$ cp -iv ../.libs/Phonetisaurus.so ./
[~/Phonetisaurus/python]$ sudo python setup.py install
[~/Phonetisaurus/python]$ cd
```

# Build and install CMUCLMTK

```shell
[~]$ sudo apt install subversion
[~]$ svn co https://svn.code.sf.net/p/cmusphinx/code/trunk/cmuclmtk/
[~]$ cd cmuclmtk
[~/cmuclmtk]$ ./autogen.sh
[~/cmuclmtk]$ make
[~/cmuclmtk]$ sudo make install
[~/cmuclmtk]$ sudo ldconfig
[~/cmuclmtk]$ cd
[~]$ sudo pip install cmuclmtk
```

# Install Pocketsphinx

## Build and install sphinxbase-0.8

```shell
[~]$ sudo apt install swig libasound2-dev bison
[~]$ git clone --recursive https://github.com/cmusphinx/pocketsphinx-python.git
[~]$ cd pocketsphinx-python/sphinxbase
[~/pocketsphinx-python/sphinxbase]$ ./autogen.sh
[~/pocketsphinx-python/sphinxbase]$ make
[~/pocketsphinx-python/sphinxbase]$ sudo make install
[~/pocketsphinx-python/sphinxbase]$ cd ..
```

## Build and install pocketsphinx-0.8

```shell
[~/pocketsphinx-python]$ cd pocketsphinx
[~/pocketsphinx-python/pocketsphinx]$ ./autogen.sh
[~/pocketsphinx-python/pocketsphinx]$ ./configure
[~/pocketsphinx-python/pocketsphinx]$ make
[~/pocketsphinx-python/pocketsphinx]$ sudo make install
[~/pocketsphinx-python/pocketsphinx]$ cd ..
```

## Install python PocketSphinx library

```shell
[~/pocketsphinx-python]$ sudo python setup.py install
```

## Format cmudict.dict and train model.fst

I'm not exactly sure why this is, but apparently it is necessary to reformat the default cmudict.dict file.

* When there are multiple pronunciations for a word, this removes the trailing "(n)".
* Then it compresses multiple white spaces into a single space.
* Then it removes white space from the beginning and end of the line.
* Finally, it replaces the first space on the line with a tab character.

```shell
[~/pocketsphinx-python]$ cd pocketsphinx/model/en-us
[~/pocketsphinx-python/pocketsphinx/model/en-us]$ cat cmudict-en-us.dict | perl -pe 's/^([^\s]*)\(([0-9]+)\)/\1/;s/\s+/ /g;s/^\s+//;s/\s+$//; @_=split(/\s+/); $w=shift(@_);$_=$w."\t".join(" ",@_)."\n";' > cmudict-en-us.formatted.dict
[~/pocketsphinx-python/pocketsphinx/model/en-us]$ phonetisaurus-train --lexicon cmudict-en-us.formatted.dict --seq2_del
[~/pocketsphinx-python/pocketsphinx/model/en-us]$ cd
```

## Test

```shell
[~]$ mkdir test
[~]$ cd test
[~/test]$ echo "<s> hello can you hear me </s>" > test_reference.txt
```

## Create test.vocab

```shell
[~/test]$ text2wfreq < test_reference.txt | wfreq2vocab > test.vocab
```

## Create test.idngram

```shell
[~/test]$ text2idngram -vocab test.vocab -idngram test.idngram < test_reference.txt
```

## Create test.lm

```shell
[~/test]$ idngram2lm -vocab_type 0 -idngram test.idngram -vocab test.vocab -arpa test.lm
```

## Create test.formatted.dict

```shell
[~/test]$ phonetisaurus-g2pfst --model=`ls ~/pocketsphinx-python/pocketsphinx/model/en-us/train/model.fst` --nbest=1 --beam=1000 --thresh=99.0 --accumulate=true --pmass=0.85 --nlog_probs=false --wordlist=./test.vocab > test.dict
[~/test]$ cat test.dict | sed -rne '/^([[:lower:]])+\s/p' | perl -pe 's/([0-9])+//g;s/\s+/ /g;@_=split(/\s+/);$w=shift(@_);$_=$w."\t".join(" ",@_)."\n";' > test.formatted.dict
```

## Test with audio file

```shell
[~/test]$ pocketsphinx_continuous -hmm ~/pocketsphinx-python/pocketsphinx/model/en-us/en-us -lm ./test.lm -dict ./test.formatted.dict -samprate 16000/8000/48000 -infile ~/test.wav 2>/dev/null
```

## Test with microphone

```shell
[~/test]$ pocketsphinx_continuous -hmm ~/pocketsphinx-python/pocketsphinx/model/en-us/en-us -lm ./test.lm -dict ./test.formatted.dict -samprate 16000/8000/48000 -inmic yes 2>/dev/null
```

Here's what this section of the profile.yml looks like

```shell
active_stt:
  engine: sphinx
pocketsphinx:
  fst_model: /home/pi/pocketsphinx-python/pocketsphinx/model/en-us/train/model.fst
  hmm_dir: /home/pi/pocketsphinx-python/pocketsphinx/model/en-us/en-us
  phonetisaurus_executable: phonetisaurus-g2pfst
```

<DocPreviousVersions/>
<EditPageLink/>
