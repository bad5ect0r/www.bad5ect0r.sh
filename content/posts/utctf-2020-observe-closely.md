+++
title = "UTCTF 2020 - Observe Closely"
date = "2020-03-09T00:00:00+11:00"
author = "bad5ect0r"
authorTwitter = "bad5ect0r" #do not include @
cover = "/utctf-2020-observe-closely/cover.png"
tags = ["ctf", "utctf", "forensics"]
keywords = ["binwalk"]
description = "Using Binwalk to extract a binary from an image."
showFullContent = false
+++

Use Binwalk to find embedded files in the image.

![Running Binwalk.](/utctf-2020-observe-closely/binwalk.png)

Use Binwalk to extract all the files. Unzip the zip file.

![Extracting files.](/utctf-2020-observe-closely/binwalk2.png)

Execute the `hidden_binary`.

![Run the executable to get the flag.](/utctf-2020-observe-closely/getting_flag.png)

The flag was: `utflag{2fbe9adc2ad89c71da48cabe90a121c0}`