---
layout: post
title:  "Day 2: NLP and embedding"
date:   2023-09-05 15:54:36 +0200
categories: days
---

Ho joy ! In one of my research side project I get the opportunity to work on some hackaton, where one needs to create a malware/goodware classifier using any machine learning techniques. The data is given for the challenge and the main idea is to try to develop some model that generalizes quite well to new data. Pretty neat. 


# Basics of Android apps dissassembly 

The data comes in ```json``` format with some hashes to identify each piece of data to some Android software. This data is a collection of information taken from the ```AndroidManifest.xml```, a document packaged in the Android binaries, and information taken from the dissassembly of the ```.pkg``` source code. 

In the challenge, I need to have the ```.pkg``` as an input. This format being nothing else than a ```.zip``` format one can simply use the ```filezip``` standard python library to extract manually the ```AndroidManifest.xml```. 

Given the filepath to the interesting APK, one has:


```python 
def load_apk(path_to_file: str, verbose: bool) -> dict:
    """Desc: Takes the relative path of the APK to dissassemble, returns the AndroidManifest.xml as a JSON format
    
        NOTE: This is a little bit harder than this because the manifest is encrypted, need to use other tools such 
               as
               https://github.com/ytsutano/axmldec
    """
    
    try: 
        with zipfile.ZipFile(path_to_file, 'r') as zip_ref:
            print("Unpacking APK...\n")
            namelist = zip_ref.namelist()

            if verbose:
                print(zip_ref.namelist())
            
            # Search for the AndroidManifest
            
            if not "AndroidManifest.xml" in namelist:
                raise FileNotFoundError
            
            else: 
                # extract a specific file from the zip container
                manifest = zip_ref.open("AndroidManifest.xml")
                manifest_content = manifest.read()

                f = open('AndroidManifest.extracted.xml', 'wb')
                f.write(manifest_content)
                f.close()
                
            # Decrypting the AndroidManifest.xml ---
            decrypt_manifest('AndroidManifest.extracted.xml')
            
            # Converting to json data/ usable features --- 
            unpack_manifest_to_json("AndroidManifest.xml")
            
            
    except FileNotFoundError:
        print("No file found at given path or unvalid Android APK") 
```

Which could probably be written in a more compact way. With no surprise, turns out that this version of the retrieved manifest is encrypted. Luckily Google is kind enough to give us some tools to unpack this document. There are a few ways of doing this, namely using the full Android dissassembly suite (```APKTool```), ```axmldec``` (but needs to be compiled from source for Linux users, which worked on my local machine but not on the remote servers). Finally the easiest to me, ```AXMLprinter```.

From the extracted file one can simply install java-jre and use the tools provided by Google at ```https://code.google.com/archive/p/android4me/downloads```. Installing java: 

``` 
conda install -c bioconda java-jdk
```
Then converting to standart output: 

```
java -jar AXMLPrinter2.jar AndroidManifest.xml > AndroidManifest.xml.txt
```

This is a nice first step and we now have the manifest data as plain text, in xml format. To solve the challenge we still need to figure out how to get additional data from the source code, which I am postponing for later.


# Encoding data and simplistic DL classification 

In machine learning experimenting is key, so we need to setup experiments using the data we have been given, establish some sensible embeddings and work with this, we can figure out the dissassembly process while the experiments are running. 

In the litterature very good malware/goodware classifiers have been put forward using a large variety of ML and DL techniques, and embedding. The approach we are taking is features frequencies (computing the average amount of preselected features and simply put those in a discrete vector), and using an natural language processing approach from the raw data (interpreting thus data as raw text).

From the data, one can dump the json data as a multiline string. We can preventively do some regex work on redundant words, symbols, ponctuation, leaving only raw data separated by spaces. 

Code piece: 

```python
for malware in onlyfiles:
    stringblob = json.dumps(malware)    # Loading data as text
    
    # Regular expression 
    stringblob = re.sub(r"[;,>:\"\{\[\]\}]", "", stringblob)    # Removing special characters
    stringblob = re.split(r'\s+',stringblob)[:-2]               # Removing hash information 
    buffer = []
    
    # Removing individual elements (keys, redundant information)    # This needs some experimenting
    
    for word in stringblob: 
        if word not in keys:
            buffer.append(word)
    
    stringblob = ' '.join(buffer).lower()

    print(stringblob)
```

At this point the blob of information looks like this: 

> android.permission.write_external_storage android.permission.access_network_state android.permission.wake_lock com.android.vending.billing android.permission.read_external_storage android.permission.internet com.infokombinat.coloringbynumbersmandala.coloringactivity com.infokombinat.coloringbynumbersmandala.splashscreenactivity com.android.billingclient.api.proxybillingactivity com.infokombinat.coloringbynumbersmandala.categoryactivity com.google.android.gms.ads.adactivity com.google.android.gms.ads.mobileadsinitprovider androidx.core.content.fileprovider android.intent.action.main android.permission.use_fingerprint android.permission.access_network_state access_network_state android.permission.media_content_control android.permission.read_phone_state android.permission.use_biometric android.permission.write_external_storage android.permission.access_network_state android.permission.wake_lock android.permission.update_device_stats android.permission.access_fine_location android.permission.interact_across_users android.permission.modify_phone_state android/location/locationmanager-getlastknownlocation android/net/connectivitymanager-getactivenetworkinfo android/hardware/fingerprint/fingerprintmanager-hasenrolledfingerprints android/media/session/mediasessionmanager-istrustedformediacontrol androidx/core/content/contextcompat-getexternalfilesdirs android/hardware/fingerprint/fingerprintmanager-authenticate androidx/core/content/contextcompat-getexternalcachedirs android/content/context-getexternalmediadirs android/media/audiomanager-requestaudiofocus androidx/appcompat/app/twilightmanager-getlastknownlocationforprovider android/os/environment-getexternalstoragedirectory android/app/alarmmanager-set android/content/context-getobbdir android/content/context-getobbdirs android/content/context-getexternalfilesdirs android/net/connectivitymanager-getnetworkinfo android/content/context-getexternalcachedirs android/hardware/fingerprint/fingerprintmanager-ishardwaredetected android/content/context-getexternalcachedir android/net/connectivitymanager-isactivenetworkmetered android/telephony/telephonymanager-isdataenabled android/os/powermanager-newwakelock android/content/context-getexternalfilesdir android/telephony/telephonymanager-getnetworktype android/location/locationmanager-requestlocationupdates getpackageinfo getexternalstoragedirectory getdeviceid cipher base64 getsystemservice ldalvik/system/dexclassloader ljava/io/ioexception-printstacktrace ldalvik/system/pathclassloader https//googleads.g.doubleclick.net/mads/static/mad/sdk/native/sdk-core-v40-loader.html https//proton.flurry.com/sdk/v1/config https//adservice.google.com/getconfig/pubvendors https//pagead2.googlesyndication.com/pagead/gen_204?id=gmob-apps https//data.flurry.com/aap.do https//www.google.com/dfp/senddebugdata https//www.google.com/dfp/debugsignals https//googleads.g.doubleclick.net/mads/static/mad/sdk/native/production/native_ads.js https//www.google.com/dfp/inapppreview https//imasdk.googleapis.com/admob/sdkloader/native_video.html https//plus.google.com/ https//goo.gl/j1swqy https//googleads.g.doubleclick.net/mads/static/mad/sdk/native/native_ads.html http//schemas.android.com/apk/res/android https//csi.gstatic.com/csi http//www.google.com https//googleads.g.doubleclick.net/mads/static/mad/sdk/native/production/mraid/v3/mraid_app_banner.js https//play.google.com/store/apps/details?id http//schemas.android.com/apk/res-auto https//www.example.com https//googleads.g.doubleclick.net/mads/static/mad/sdk/native/production/sdk-core-v40-impl.html http//data.ikppbb.com/coloring-by-numbers-super-data/ https//googleads.g.doubleclick.net/mads/static/mad/sdk/native/production/mraid/v3/mraid_app_interstitial.js https//support.google.com/dfp_premium/answer/7160685#push https//www.google.ru/ https//data.flurry.com/pcr.do http//www.example.com https//googleads.g.doubleclick.net/mads/static/mad/sdk/native/production/mraid/v3/mraid_app_expanded_banner.js https//googlemobileadssdk.page.link/admob-android-update-manifest https//googleads.g.doubleclick.net/mads/static/mad/sdk/native/production/sdk-core-v40-impl.js https//www.google.com/dfp/linkdevice http//data.flurry.com/aap.do https//googlemobileadssdk.page.link/admob-interstitial-policies


Which is of course a whole lot of nonsense, however, there is a little bit of a dilemna. We are going to have to be a little subtle about the text to vector encoding, either we get rid of some information in the URLs, or we find a way to keep those. In those URLs is probably some malware fingerprints that might help the model, but at the same time it might make the embedding worse. 

A potential solution is to partially clean the URLs, for example only keeping part of it. It might be the most sensible choice. Step by step we start by removing redundant http and https predicants. 