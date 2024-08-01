---
title: "Telegrambot"
date: 2024-06-05T18:35:55+03:00
draft: true
---

I really like Telegram and have long meant to make a bot for it from scratch, in Python, and properly start learning a language I really want to be proficient in.

I am using [FreeCodeCamp's](https://www.freecodecamp.org/news/how-to-create-a-telegram-bot-using-python/) guide to get started and will likely end up looking at multiple sources to get this going, as what I'm planning on doing is a bit more complex than a simple bot; it should be able to receive messages from a "real" Telegram user, translate them and then post them to another group. First I'd like the bot to be able to do this function, and then add in more groups to monitor so that I could have one Telegram group to look at multiple channels worth of content inside. 

I have already made a bot for this purpose by messaging @botfather in Telegram, so I can skip this part.

In the FreeCodeCamp guide I am now instructed to make a `.env` file to store my bot's token in, and while I understand the concept of what the file is and why it is important, I don't quite know how to make it. Let's see... Got it. I sometimes tend to think things in a more complex manner than required, and applied this logic to making the .env file, but in PyCharm, which is the code editor I'm using, it is literally as easy as right-clicking on your project folder > New > File and entering .env as the filename. That's it. Inside I will need to make a variable to store the bot token into.

After this I already noticed a hiccup in the guide due to it instructing to use certain functionality in TeleBot that I suppose is no longer there; therefore I go and look for a more up-to-date guide and found this https://tjtanjin.medium.com/how-to-build-a-telegram-bot-a-beginners-step-by-step-guide-c671ce027c55. I already learned from the previous guide that I can enter `import os` at the beginning of the code to read the environment variable I previously set for the bot token, and I can use this token with `os.environ.get('token')`, thus not saving it in the code itself, unlike this guide I found.  

I have already stumbled in some problems: using PyCharm, I wasn't sure where I should be placing my .env file, apparently (I guess this makes sense and I should've realised it) the .env file needs to be in the same folder where the Python code referencing it is. It obviously being in the wrong place, I kept getting an error stating that I should share my bot token in the code. Additionally, the older guide did not mention anything about possibly needing an additional line in the code that would load a local .env file, and consulting ChatGPT on this helped figure out that I should probably pip install `python-dotenv` and add a line  `load_dotenv()` before `os.environ.get('token')`. This fixed my problems and I can keep going. 

The Medium article uses a wrapper called python-telegram-bot, and since it seems to have solid documentation (at least in the eyes of someone who is not a developer), I decide that it is a good wrapper to use for my code.

 



