+++
title = "Voice to Text telegram bot"
date = 2022-06-19T12:01:15+02:00
thumbnail = "images/v2t.png"
+++

# Intro
Well, we all have that friend, who is too lazy to type messages and just spams you with voices, and if you don't - that most likely you're that friend.
It's ok, we're all not without a flaw and to be honest this one is not the worst to have. Most people are put up with that and patiently listen to your horrible quality audios, on the streets, through Bluetooth headphones, with traffic in the background. But what if that person is in a place, where they can't just listen to your audio message (e.g. church)? Well...
![modern.png](/images/modern.png)
# Bots to the rescue!
We're gonna build the telegram bot, which would be able to patiently listen to the message that was sent to you and then just return you beautiful, pure text, the one that we all love so much. Ok, so if it's a speech recognition task, this means neural networks (there are of course available APIs, but we don't want to send our private messages to someone like Google). If it's neural networks - then we think Python. 
Actually, we won't train our own network, so we would be able to in fact be done with most other programming languages, but Python just feels more natural tool of choice for this task.
We will start by initializing a new project (I'll be using [poetry](https://python-poetry.org/) as a virtual environment and dependency management tool).

```
mkdir v2t
cd v2t
poetry init
```

and just follow the instructions.

Now that we have a fresh project, let's proceed with the dependencies

```
poetry add python-dotenv
poetry add python-telegram-bot@20.0a1 --allow-prereleases
```

The first one is my go-to package in most of the python projects, cause it eases up usage of the values in the `.env` files and the second is basically our library to abstract usage of telegram APIs (they're not the easiest one to use)

Let's start with a basic bot, which will ignore messages that do not contain audio or video notes and will save those as a file.

```
import os

from dotenv import load_dotenv
from telegram import Update
from telegram.ext import ApplicationBuilder, ContextTypes, CommandHandler, MessageHandler, filters

class Bot:
  def __init__(self):
    load_dotenv()
    token = os.getenv('TELEGRAM_TOKEN')
    self.app = ApplicationBuilder().token(token).build()

  async def get_voice(self, update: Update, context) -> None:
    # get basic info about the voice note file and prepare it for downloading
    new_file = await context.bot.get_file(update.message.voice.file_id)
    # download the voice note as a file
    await new_file.download(f"voice_note.ogg")

  async def get_video(self, update: Update, context) -> None:
    # get basic info about the video note file and prepare it for downloading
    new_file = await context.bot.get_file(update.message.video_note.file_id)
    # download the video note as a file
    await new_file.download(f"video_note.mp4")

  def start(self):
    self.app.add_handler(MessageHandler(filters.VOICE, self.get_voice))
    self.app.add_handler(MessageHandler(filters.VIDEO_NOTE, self.get_video))
    self.app.run_polling()


if __name__ == '__main__':
  bot = Bot()
  bot.start()
```

We'll create a class `Bot`, which will be the core of our program (that's unsurprising, considering we're writing the telegram bot). During its instantiation, it will read the token from your telegram bot (to learn how to register one go [here](https://core.telegram.org/bots)) and build a basic bot app. Along with this, we're gonna add two async methods to our class, which will serve as message handlers (functions that are invoked when a specific message is received), one for video notes and one for audio. And the last, but not least `start` method, which is responsible for registering the message handlers and basically starting our bot. In the end, since it's our `main.py` file we'll add a couple of lines to invoke this bot and start the app when the script is running.

Now let's try this thing out and see if it even works.
First, we'll send a simple voice recording to our bot:
![tg1.png](/images/tg1.png)
And then check our folder:
```
(v2t-FJMpmN5L-py3.8) ➜  v2t ll
total 36K
-rw-r--r-- 1 sonac sonac 1.2K Jun 18 17:15 main.py
-rw-r--r-- 1 sonac sonac  19K Jun 18 17:09 poetry.lock
-rw-r--r-- 1 sonac sonac  366 Jun 18 17:09 pyproject.toml
-rw-r--r-- 1 sonac sonac 5.2K Jun 18 17:15 voice_note.ogg
```

Awesome! Seems working!

# Voice recognition stuff
Now to the interesting part. We'd like to be able to recognize all those things that are being sent to our bot into a nice text. As we said before, we'd like to be able to do it offline and thankfully there are tools for that. In this particular case, I'm referring to a pretty cool project [Vosk](https://alphacephei.com/vosk/). This bad boy contains trained models for more than 20 languages and allows us with just a tiny bit of code transcript audio or video to text. In its most basic form, Vosk works with `wav` files, so we'd need to do a bit of transcoding. Fortunately, on their Github, there is already an example that we can use, with only a tiny bit of adaptation, but first, let's install our dependency.
```
poetry add vosk
```

Now let's define our transcriptor in a separate file (not that our project will grow further, but still, let's keep stuff clean):
```

import subprocess
import json 

from vosk import Model, KaldiRecognizer

class Transcryptor:
  def __init__(self, model_name: str) -> None:
    self.sample_rate=16000
    self.model = Model(model_name)
    self.rec = KaldiRecognizer(self.model, self.sample_rate)
    self.cur_res = ""

  def __append(self, res) -> None:
    js = json.loads(res)
    if 'text' in js.keys():
      self.cur_res += js['text']
    if 'partial' in js.keys():
      self.cur_res += js['partial']
    self.cur_res += '.\n'

  def transcrypt(self, file_path) -> str:
    self.cur_res = ""
    process = subprocess.Popen(['ffmpeg', '-loglevel', 'quiet', '-i', file_path, '-ar', str(self.sample_rate),
      '-ac', '1', '-f', 's16le', '-'], stdout=subprocess.PIPE)
    while True:
      data = process.stdout.read(4000)

      if len(data) == 0:
        break
      if self.rec.AcceptWaveform(data):
        self.__append(self.rec.Result())
      else:
        print(self.rec.PartialResult())
    self.__append(self.rec.FinalResult())

    self.cur_res = self.cur_res[:-2]
    return self.cur_res
    
```


We'll again define it inside the class, to keep stuff organized, it will take one positional parameter at invocation, which is gonna be the name of the model, we gonna be using for our speech recognition (all models are available here: https://alphacephei.com/vosk/models). During class instantiation it will also instantiate the model, which will take some time, so be patient during app startup, it will not be as immediate as before. Our class will contain two methods, one private (well, kinda private, since it's Python) and one public. The first one will be responsible for concatenating pieces of transcripted text and the second accepts the path to the file, which contains audio to the transcript. Here we'll use the subprocess library to launch `ffmpeg` for converting to wave format. If you don't have `ffmpeg` in your system, you can install it using your package manager. To read more about `ffmpeg` you can go to https://ffmpeg.org.

So now let's assemble the pieces together, this is how our reworked bot class now looks like:

```
class Bot:
  def __init__(self):
    load_dotenv()
    token = os.getenv('TELEGRAM_TOKEN')
    model_name = os.getenv('MODEL_NAME')
    self.app = ApplicationBuilder().token(token).build()
    self.transcryptor = Transcryptor(model_name)

  async def get_voice(self, update: Update, context) -> None:
    # get basic info about the voice note file and prepare it for downloading
    new_file = await context.bot.get_file(update.message.voice.file_id)
    # download the voice note as a file
    await new_file.download(f"voice_note.ogg")
    # transcrypt the voice note
    transcrypted_text = self.transcryptor.transcrypt("voice_note.ogg")
    await context.bot.send_message(chat_id=update.effective_chat.id, text=transcrypted_text)

  async def get_video(self, update: Update, context) -> None:
    # get basic info about the video note file and prepare it for downloading
    new_file = await context.bot.get_file(update.message.video_note.file_id)
    # download the video note as a file
    await new_file.download(f"video_note.mp4")
    # transcrypt the video note
    transcrypted_text = self.transcryptor.transcrypt("video_note.ogg")
    await context.bot.send_message(chat_id=update.effective_chat.id, text=transcrypted_text)

  def start(self):
    self.app.add_handler(MessageHandler(filters.VOICE, self.get_voice))
    self.app.add_handler(MessageHandler(filters.VIDEO_NOTE, self.get_video))
    self.app.run_polling()
```

And let's run a new test.

![tg2.png](/images/tg2.png)
Alrighty, seems working, even with me being non-native, it recognizes speech pretty well.
So that'd be it. There are some ways to improve it furthermore. First of all, if you want to make this publically usable - it would be good to make transcription also async, not only message handlers and of course, we'd need to generate new names for each audio message, since they might overlap from different chats. If you're bilingual (or even more, like I have chats with friends speaking Ukrainian, Russian and English) it would be also super beneficial to include language detection, there are some tools for that either (you can check out this one: https://huggingface.co/TalTechNLP/voxlingua107-epaca-tdnn)

# Conclusion
So that'd be it. Overall we ended up having a nice and tidy piece of code, which actually is super useful for many of you out there. This is literally opposite from the majority of enterprise stuff, that many software engineers are writing on their 9-to-5, but let's not be their judges. 
If you find any of this confusing, or just want to jump straight to the code, the final version is on my Github, here: https://github.com/sonac/v2t

P.S. It just so happened that in the fresh Telegram Premium this feature is inbuilt, but... Now you know how to enable the only important feature of the Premium for free :) 