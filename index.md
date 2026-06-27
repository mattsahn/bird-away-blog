---
layout: home
title: Bird Away&#58; Building an AI-Powered Bird Deterrent with Raspberry Pi and Claude
---

*June 2026 · [Source on GitHub](https://github.com/mattsahn/bird-away)*

![Bird Away](https://github.com/user-attachments/assets/51d76a95-e8b6-47e1-b92c-d38c6e815195)

## The Problem

I live in NYC and my parents live in Florida and have a pool. The pool attracts a lot of birds and it was getting to be a bit of a problem with their associated output. We tried a few things (fake owl, floating alligator head) to keep them away, but birds are smart and determined and those did not work for very long. When I was there, I would sometimes sneak up on them with a hose and that would scatter them pretty effectively (except for one large duck that seemed to enjoy being drenched by a hose). So, I had the idea of automating this process as a fun project.

## The Idea

Simple. Have a camera pointed at the pool, take pics in a loop, have AI analyze the pic and return "yes" if there are birds in the pic, trigger a sprinkler with relay via GPIO on a Raspberry Pi. For extra credit, also save pics/video to the cloud so that I can get the satisfaction of seeing it work.

![Hardware chain](https://github.com/user-attachments/assets/c858b5ad-f7e6-4a01-8866-c37de9c4c621)

## The Hardware

- **Raspberry Pi 4** on home Wi-Fi — runs the Python monitoring service
- **IP camera** — aimed at the pool, connected on same Wi-Fi network
- **Relay module** — wired to GPIO pin
- **12V solenoid valve** — on its own 120V power supply, switches on by the relay for set duration
- **Sprinkler head** — connected to water supply, aimed at pool
- **Status LED button** — heartbeat-blinks while the service is running to give positive indication IRL. Also can trigger a manual run.

![Pi enclosure](https://github.com/user-attachments/assets/63336784-0bd0-4567-83ed-71b2f77b96e4)

## How It Works

A Python process runs on the Pi and is set to start itself whenever the Pi starts, so that it is resilient to power restart. It connects to the IP camera using RTSP protocol and captures an image every 2 mins. It posts this to OpenRouter as an API call along with a prompt asking if there is a bird in the picture. The AI model will respond with either "yes" or "no". If "yes", then the service will start recording a video clip, sleep for a few seconds, then power on the sprinkler relay, keep it on for 5 seconds, keep recording a few more seconds, and then post the image and video to an S3 bucket. 

![Bird detection event](https://github.com/user-attachments/assets/2fb09415-a968-4a3c-9c0a-fbbbb147f2fd)

*A snapshot from a detection event — the frame that triggered the spray*

## Cloud Stuff

The whole payoff is seeing birds get sprayed and startled. So, capturing this for posterity was key, and in a way I can see it remotely from NYC to admire my creation. I also need to be able to login to the Pi to maintain it, make changes and new features, etc, without being there. 
For remote access, I use [Raspberry Pi Connect](https://connect.raspberrypi.com/) which is free and allows me to login to a shell from any internet browser. $0. awesome. 
For visual evidence, the process will upload the still image when bird detection triggered and a video to Cloudflare R2 service which has a generous free tier. 
For ease of viewing, I created a simple webpage that organizes and displays all the snaphsots and links to videos so that I can check from my phone anytime. The page is deployed on Vercel, also free.

![Event dashboard](https://github.com/user-attachments/assets/b501166a-2edc-4928-a1f7-470b0a64b1cd)

*The web dashboard: each card is a detection with a thumbnail, timestamp, and link to the video clip*

## Engineering Notes

### Heat management ###

I was very concerned about heat in the florida sun baking the Pi in the plastic enclosure, and took a lot of measures to try to manage heat. For the Pi, I got an alumininum case that thermally connects to the processor and passively dissipates heat:

<img width="250" alt="image" src="https://github.com/user-attachments/assets/cb49d595-314d-41df-b61b-639249cea9f1" />

But, I needed that heat to not accumlate inside of the plastic enclosure, but also keep it waterproof, so I cut out an opening in the enclosure and caulked it in so that one side of the case is exposed to the atmosphere so heat can go there:

<img width="200" alt="image" src="https://github.com/user-attachments/assets/ddc48c04-7c90-4815-bf01-c4c75365bed4" />

For good measure, I added some reflective tape as well to bounce off the most intense direct sunlight:

<img width="200" alt="image" src="https://github.com/user-attachments/assets/9156d836-34f6-472a-9217-86af6f418b74" />


### Zero SD card writes

Raspberry Pi SD cards go bad when you write a lot to them and i've experienced this quite a lot over the past decade of doing home Pi projects. So, I have this setup to basically not write anything to the filesystem/SD card. The app logging is all in memory and same for the pics and video files. I can read logs when i'm connected with a service command and I don't care about storing the image/video locally at all, they are uploaded to the cloud. I really want this thing to last and run for months on its own and this should help a lot.

### Remote monitoring

I added a free [healthchecks.io](https://healthchecks.io) monitoring setup to send periodic heartbeats. If/when these stop, I get an email alert.

### AI stuff

Picking and tuning an AI model ended up being a project in itself. This system relies on running inference on images a lot - like thousands of times a month, so cost matters. I'm not interested in spending $100/mo to spray birds. So, I had to do a whole lot of evaluation of different models as well as experimenting with downscaling image sizes. I did this fairly scientifically by capturing a handful of "bird" and "no bird" images, scaling to some different sizes, and running against a number of different models via OpenRouter. OpenRouter was pretty key for how easy it is to swap models with a single API and account/billing. I wanted a model that accurately detected birds and no-birds, and costed as little as possible.

#### The winner? #### 

[gemma-4-31B-it](https://huggingface.co/google/gemma-4-31B-it) - This is an open-weight model and only costs me about 60 cents per month! For 10,000 images analyzed! I was pretty shocked I was able to bring the costs this low to less than $10/year.

<img width="850" alt="image" src="https://github.com/user-attachments/assets/de2cf6c7-adcb-40eb-9d71-aea3b9ea3791" />

*OpenRouter billing page*

My prompt:
```
 You are a bird detector for a backyard pool. Respond with exactly 'yes'
 if you see one or more birds in, on, or near the pool in the provided photo. 
 Respond with exactly 'no' otherwise. Output only the single word.
```


## Results

The thing works! It's detecting and spraying birds and (usually) shooing them away. Ducks are tricky. They DGAF about being sprayed for the most part. I guess being comfortable with water is kind of their thing, so i'm thinking of other deterrents I might add - like a blast of ultrasonic sounds or something.  

The full source, wiring notes, install instructions, and tuning guide are at [github.com/mattsahn/bird-away](https://github.com/mattsahn/bird-away).

---

*Questions or feedback: [open an issue on GitHub](https://github.com/mattsahn/bird-away/issues).*
