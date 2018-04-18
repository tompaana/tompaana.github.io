# Remotely Controlled Bots #

*Published originally on March 15th, 2017*<br />
*Updated on April 18th, 2018*

You know. Because it’s good to have a fail-safe around in case of Skynet.

But to the point: Let’s say you have a great backend heavy application and you want to deliver a bot
experience to broaden your user base and to provide a new way of interacting in your app context.
Well your new bot can surely be made to access the application data in your backend, but how can you
make it work the other way around – say, in case of notifications?

![An image with three arrows pointing at things.](/content/images/BackendControlledAppModel.png)<br />
*That’s the bot on the right, by the way.*

Hence the title – by remote control I simply refer to a backend controlling the bot remotely (over
the interwebs) by messages that can be interpreted as commands to execute an action.

## Backchannel ##

That’s what we call it and apparently it’s just one word. What we mean by the word is a type of
message (like the ones users send to talk to a bot and the bot uses to reply back), but we just put
the meaningful content in a different place of the message object
([Activity](https://docs.microsoft.com/en-us/dotnet/api/microsoft.bot.connector.activity?view=botconnector-3.12.2.4) in C#).
Namely, we put the message the bot should react to somehow in
[IMessageActivity.ChannelData](https://docs.botframework.com/en-us/csharp/builder/sdkreference/d1/de8/interface_microsoft_1_1_bot_1_1_connector_1_1_i_message_activity.html)
instead of [IMessageActivity.Text](https://docs.botframework.com/en-us/csharp/builder/sdkreference/d1/de8/interface_microsoft_1_1_bot_1_1_connector_1_1_i_message_activity.html).
Tadaa! End of article.

No, but it really is that simple! In a nutshell you devise a simple custom protocol that your bot
knows, for example, when the IMessageActivity.Text contains “notification”, you look at the channel
data content to see who and with what message to notify. Then let your implementation in the bot
code to do it’s job. Still don’t believe me? Look, here’s [a sample](https://github.com/tompaana/remote-control-bot-sample) (in C#).

Ok, you got me. What I failed to mention is that you have to have some Microsoft Bot Framework
specific code in your backend. Perhaps the easiest way to implement this backchannel messaging
pipeline **between** the backend and the bot is using
[Direct Line](https://docs.botframework.com/en-us/restapi/directline3/).
And the easiest way to use the Direct Line is by utilizing the ready-made client components for
[Node.js](https://www.npmjs.com/package/directline-api) and
[C#](https://www.nuget.org/packages/Microsoft.Bot.Connector.DirectLine). If your backend is not
compatible with Node or C# components, implementing your own Direct Line connection is quite
straightforward (the first link about Direct Line describes the protocol). They are, after all, only
HTTP calls. My sample comes with
[a super simple console app sending notification commands to the bot](https://github.com/tompaana/remote-control-bot-sample/tree/master/RemoteControlBotControllerSample).
You should be able to use the code almost as-is, if your backend is built with C#.

What about **security**? I’m not an expert, but there are three points here I want to make:

1. The Direct Line pipeline is secured by a secret key and TLS
2. The user cannot inject content to the channel data (think of SQL injection vulnerability) as long
   as the channel (e.g. Skype) is secure
3. You can encrypt the channel data content

Note that some descriptions of backchannel say that you should also change the value of
[the Type property](https://docs.botframework.com/en-us/csharp/builder/sdkreference/dc/d2f/class_microsoft_1_1_bot_1_1_connector_1_1_activity.html)
of your Activity; from being “message” to “event”. This is a matter of taste. The benefit of this is
that you can be sure that your backchannel message is not treated as a regular message (because the
type is not “message”).

## Where to, sir? ##

Where am I supposed to place this backchannel messaging specific code in my bot project? To me, this
introduces some controversy; The bot framework utilizes [Autofac](https://autofac.org/), an
*inversion of control (IoC) container* for dealing with dependencies, and I am **not** a fan. In my
opinion wide use of IoC leads to incoherent code and architecture with little benefits to offer. And
it can make writing tests (which I don’t do unlike true professionals I guess) a pain! But that’s
just me – maybe my brain is not sophisticated enough to understand these kinds of exquicite
concepts.

Just to show I can do things I don’t like I integrated the backchannel bot code using Autofac in my
sample. Take a look at
[GlobalMessageHandlerModule.cs](https://github.com/tompaana/remote-control-bot-sample/blob/master/RemoteControlBotSample/GlobalMessageHandlerModule.cs)
and [Global.asax.cs](https://github.com/tompaana/remote-control-bot-sample/blob/master/RemoteControlBotSample/Global.asax.cs).
I’ve created classes derived from [ScorableBase](https://docs.botframework.com/en-us/csharp/builder/sdkreference/de/d7b/class_microsoft_1_1_bot_1_1_builder_1_1_scorables_1_1_internals_1_1_scorable_base.html),
which are automatically invoked when (and only when) I forward the received `Activity` object to my
root dialog in
[MessagesController.cs](https://github.com/tompaana/remote-control-bot-sample/blob/master/RemoteControlBotSample/Controllers/MessagesController.cs).
Then if a backchannel message is detected, the specific scorable class
([NotificationsScorable](https://github.com/tompaana/remote-control-bot-sample/blob/master/RemoteControlBotSample/Notifications/NotificationsScorable.cs)
in my sample) consumes and deals with the Activity and it is never given to my dialog. Special
thanks to my brilliant colleague, [Lilian Kasem](http://liliankasem.com/), for coming up with this
idea!

Call me old-fashioned, but I still find the code a lot easier to understand if I simply put this
logic to my MessagesController class (or equivalent) before passing anything to any dialog. That’s
just the way I roll…

```cs
if (we got a valid backchannel message)
{
    // Do what needs to be done
}
else
{
    // Looks like a message from a user, let the dialog handle it
    await Conversation.SendAsync(activity, () => new RootDialog());
}
else ...
```

See?

## Related resources ##

* My sample demonstrating how to remotely force the bot to notify bunch of users about something:
  [Remote Control Bot Sample](https://github.com/tompaana/remote-control-bot-sample) (C#)
* [Proactive Bots blog post](http://blog.codemoggy.com/index.php/2017/04/26/using-the-microsoft-bot-framework-and-azure-to-create-a-proactive-bot/)
  and [code sample](https://github.com/CodeMoggy/ProactiveBotMessaging) by
  [Richard Custance](https://github.com/CodeMoggy)
* [Somethingsomething and Contextual Bots via Back Channel](https://blogs.msdn.microsoft.com/richard_dizeregas_blog/2017/02/15/sharepoint-framework-and-contextual-bots-via-back-channel/)
  (article where Back Channel is apparently two words)
