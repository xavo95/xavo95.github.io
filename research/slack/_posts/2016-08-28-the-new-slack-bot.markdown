---
layout: post
title: "The new Slack Bot"
date: 2016-08-28
author: xavo95
---

### Intro

First of all thanks to the excellent starter guide by [FullStackPython](https://twitter.com/fullstackpython), you can find his guide here [FullStackPython SlackBot Guide](https://www.fullstackpython.com/blog/build-first-slack-bot-python.html)

### Index

* [Requirements](#requirements)
* [Setting up](#setting-up)
* [Demo example bot](#demo-example-bot)
    * [Code](#code)
    * [Parsing slack output](#parsing-slack-output)
    * [Handling commands](#handling-commands)
* [Improvement](#improvement)
* [ToDo List](#todo-list)

### Requirements

* Python 3 and pip
* Python slackclient library
* Slack account to create the bot
* Optional but not required - virtualenv (python)

### Setting up

Fist of all head up to the team admin page on Slack, select integrations -> custom integration -> bot, set the parameters and write down your api token.

I will assume that you are using virtualenv, so:

Enter on your working directorty with Terminal/Command Prompt and type:

```shell
virtualenv SlackBot
```

Before activation let's modify the activation script so:

#### Windows

Add this to `SlackBot/Scripts/activate.bat`:

```shell
set "SLACK_BOT_TOKEN=your slack token pasted here"
```
And then on Command Prompt:

```shell
SlackBot/Scripts/activate.bat
```

#### Linux/OSX

Add this to `SlackBot/bin/activate`:

```shell
export SLACK_BOT_TOKEN='your slack token pasted here'
```
And then on Terminal:

```shell
source SlackBot/bin/activate
```

#### Continuation(same for Linux/OSX/Windows)
Now on Python:

```shell
pip install slackclient
```

Now, we will need the bot ID so we will use this code

`print_bot_id.py` by [FullStackPython](https://twitter.com/fullstackpython):

```python
import os
from slackclient import SlackClient


BOT_NAME = 'your bot name here'

slack_client = SlackClient(os.environ.get('SLACK_BOT_TOKEN'))


if __name__ == "__main__":
    api_call = slack_client.api_call("users.list")
    if api_call.get('ok'):
        # retrieve all users so we can find our bot
        users = api_call.get('members')
        for user in users:
            if 'name' in user and user.get('name') == BOT_NAME:
                print("Bot ID for '" + user['name'] + "' is " + user.get('id'))
    else:
        print("could not find bot user with the name " + BOT_NAME)
```

From Terminal/Command Prompt:

```shell
python print_bot_id.py
```

When finished. type this on the same console(it will unload virtualenv to be able to modify activate script):

```shell
deactivate
```

Now, write down the bot_id and modify again the activate script

#### Windows

Add this to `SlackBot/Scripts/activate.bat`

```shell
set "BOT_ID=bot id returned by script"
```
And then on Command Prompt:

```shell
SlackBot/Scripts/activate.bat
```

#### Linux/OSX

Add this to `SlackBot/bin/activate`

```shell
export BOT_ID='bot id returned by script'
```
And then on Terminal:

```shell
source SlackBot/bin/activate
```

### Demo example bot

Now let's see this code, and analyze it, this demo code is again from [FullStackPython](https://twitter.com/fullstackpython):

#### Code

```python
import os
import time
from slackclient import SlackClient


# starterbot's ID as an environment variable
BOT_ID = os.environ.get("BOT_ID")

# constants
AT_BOT = "<@" + BOT_ID + ">:"
EXAMPLE_COMMAND = "do"

# instantiate Slack & Twilio clients
slack_client = SlackClient(os.environ.get('SLACK_BOT_TOKEN'))


def handle_command(command, channel):
    """
        Receives commands directed at the bot and determines if they
        are valid commands. If so, then acts on the commands. If not,
        returns back what it needs for clarification.
    """
    response = "Not sure what you mean. Use the *" + EXAMPLE_COMMAND + \
               "* command with numbers, delimited by spaces."
    if command.startswith(EXAMPLE_COMMAND):
        response = "Sure...write some more code then I can do that!"
    slack_client.api_call("chat.postMessage", channel=channel,
                          text=response, as_user=True)


def parse_slack_output(slack_rtm_output):
    """
        The Slack Real Time Messaging API is an events firehose.
        this parsing function returns None unless a message is
        directed at the Bot, based on its ID.
    """
    output_list = slack_rtm_output
    if output_list and len(output_list) > 0:
        for output in output_list:
            if output and 'text' in output and AT_BOT in output['text']:
                # return text after the @ mention, whitespace removed
                return output['text'].split(AT_BOT)[1].strip().lower(), \
                       output['channel']
    return None, None


if __name__ == "__main__":
    READ_WEBSOCKET_DELAY = 1 # 1 second delay between reading from firehose
    if slack_client.rtm_connect():
        print("StarterBot connected and running!")
        while True:
            command, channel = parse_slack_output(slack_client.rtm_read())
            if command and channel:
                handle_command(command, channel)
            time.sleep(READ_WEBSOCKET_DELAY)
    else:
        print("Connection failed. Invalid Slack token or bot ID?")
```

Nice code, it works but how?
Let's look inside main:

* Firstly it connects to slack rtm service
* Then he expects output from slack(`parse_slack_output`)
* If command and channel are set it process the command(`handle_command`)
* Finally the script runs in a loop forever(or until you close it)

Easy, yeah? Now let's look deep and the interesting functions; firstly to `parse_slack_output`, and finally `handle_command`

**NOTE: To simplify and make the code small I have removed the comments to explain it more. The original comments are in the demo previously shown**

#### Parsing slack output

```python
def parse_slack_output(slack_rtm_output):
    output_list = slack_rtm_output
    if output_list and len(output_list) > 0:
        for output in output_list:
            if output and 'text' in output and AT_BOT in output['text']:
                return output['text'].split(AT_BOT)[1].strip().lower(), \
                       output['channel']
    return None, None
```

So time for `parse_slack_output` analysis:

* It receives the Slack rtm output as a parameter and check if list is empty
* If it's not it will do the iterations on the list
* For each item it will check 
    * That output(the actual list object) is not null
    * That 'text' is contained in output
    * Finally it will also check that the message is headed to our bot
* If all that conditions are true the function, it will strip the the bot name and the whitespace from the message
* Finally
    * If all conditions are true it will return the content of the message and the channel where it happened
    * If not, it will return None, None which is equivalent to null, null

#### Handling commands

```python
def handle_command(command, channel):
    response = "Not sure what you mean. Use the *" + EXAMPLE_COMMAND + \
               "* command with numbers, delimited by spaces."
    if command.startswith(EXAMPLE_COMMAND):
        response = "Sure...write some more code then I can do that!"
    slack_client.api_call("chat.postMessage", channel=channel,
                          text=response, as_user=True)
```

And finally we will analyze `handle_command`:

* It takes the output from `parse_slack_output` if the condition in `main` is true
* It will check if command starts with the EXAMPLE_COMMAND defined a top
* Finally
    * If matches, it will send and information message
    * If fail it will send and error message with an usage example

### Improvement

#### My Proposals

* Add the possibility to interact with the bot with other keywords than it's name
* Customize `handle_command` and add some examples
 
#### Add the possibility to interact with the bot with other keywords than it's name

On constants section replace `AT_BOT = "<@" + BOT_ID + ">:"` for:

```python
AT_BOT = "<@" + BOT_ID + ">:"
AT_BOT_2 = "<@" + BOT_ID + ">"
NAME = "nncubie"     """ what ever you want, in my case the bot name 
                         (disable that nickname on Slack panel to disable overlap) """
SHORTCUT = "Nn"      """ same, but now to create a shortcut 
                         (disable that nickname on Slack panel to disable overlap) """
EVERYONE = "<!everyone>:"
EVERYONE_2 = "<!everyone>"
TRIGGERS = [AT_BOT, AT_BOT_2, NAME, SHORTCUT, EVERYONE, EVERYONE_2]
```

**I know that I could put the strings directly on the array and get ride of some lines, but for testing purposes this is better**

Finally modify `parse_slack_output` function:

```python
def parse_slack_output(slack_rtm_output):
    output_list = slack_rtm_output
    if output_list and len(output_list) > 0:
        for output in output_list:
            if output and 'text' in output:
                for receiver in TRIGGERS:
                    if receiver in output['text']:
                        # return text after the @ mention, whitespace removed
                        return output['text'].split(receiver)[1].strip().lower(), \
                            output['channel']
```

What we have done? 

* We add an array of bot trigger(magic words)
* Modify the function to iterate accross the triggers see if matches

#### Customize `handle_command` and add some examples

ToDo

### ToDo List

#### Original author proposals

* Attach a persistent [relational database](https://www.fullstackpython.com/databases.html) or [NoSQL back-end](https://www.fullstackpython.com/no-sql-datastore.html) such as [PostgreSQL](https://www.fullstackpython.com/postgresql.html), [MySQL](https://www.fullstackpython.com/mysql.html) or [SQLite](https://www.fullstackpython.com/sqlite.html) to save and retrieve user data
* Add another channel to interact with the bot via [SMS](https://www.twilio.com/blog/2016/05/build-sms-slack-bot-python.html) or [phone calls](https://www.twilio.com/blog/2016/05/add-phone-calling-slack-python.html)
* [Integrate other web APIs](https://www.fullstackpython.com/api-integration.html) such as [GitHub](https://developer.github.com/v3/), [Twilio](https://www.twilio.com/docs) or [api.ai](https://docs.api.ai/)

