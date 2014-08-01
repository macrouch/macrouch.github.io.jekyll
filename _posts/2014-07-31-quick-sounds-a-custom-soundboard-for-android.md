---
layout: post
title: "Quick Sounds, A Custom Soundboard for Android"
tags:
 - Android
---

I don't know about you, but I have a ton (ok, a couple dozen, tops) of funny 1-2 second sound clips on my phone. Most of them seem to originate in some inside joke from years ago, and its fun to play them for friends when that inside joke pops into the conversation. This is where I got the idea to make a custom soundboard for my Android device.

Previously, I had to open up the music player on the phone which sorts all the audio on the device by standard metadata such as artist, album or genre. This just didn't work for my use case. These short audio clips have no artist, album or genre, and it simply took too long to find the clip and play it. By the time I could find the sound and play it, the moment had passed.

At the time, way back in 2012, I searched the Google Play Store for a custom soundboard, and to my surprise I could only find one. I hadn't tried to do anything with Android development before, but I thought it couldn't be all that hard to write an app to do what I wanted. I didn't keep track of my progress like I try to in more current projects, but I do have all of the source code on <a href="https://github.com/macrouch/quick-sounds" target="_blank">github</a>.

This last month I bought a <a href="https://play.google.com/store/devices/details/LG_G_Watch_Black_Titan?id=lg_g_watch_black&hl=en" target="_blank">LG G Watch</a>. This got me wanting to take a another look at the app, and I spent a few nights updating the code and removing all of the *numerous* deprecation warnings. I also thought it would be fun to make Quick Sounds compatible with Android Wear devices. I hoped to give a voice command such as, "play a sound," and the app on the phone would play an audio clip. Unfortunately custom voice commands like this are not available at this time. To just learn a bit about how the smart watch interfaces with a phone I added a watch version of the app, that has a button to play the first clip. I moved on from this project to put together a Android Wear demo I presented at work. If I have the time I would like to come back to this idea and work on it more. I would like to display the name of the audio clip on the watch, so the user knows what sound will be played, and possibly add more features.

<a href="" class="btn btn-lg btn-primary" target="_blank">View on Github</a>
<a href="http://play.google.com/store/apps/details?id=com.sleekcoder.quick_sounds" target="_blank">
  <img src="http://www.android.com/images/brand/get_it_on_play_logo_small.png" alt="Get it on Google Play">
</a>
