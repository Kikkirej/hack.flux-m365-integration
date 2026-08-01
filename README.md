# Hack.flux Integration - Microsoft 365

[Hack.flux](https://sr.ht/~chirpcel/hack.flux/) is a tool for organizing hackathons. To integrate in existing infrastructure it enables integrating by reacting to events. This integration is meant for integrating in Microsoft 365 Ecosystem.

Features:

* Sending Mails
* Sending events
* Create Teams Channel
* Add User to Team
* Add User to Teams-Channel
* Remove User from Teams-Channel

## Events

### User registered

This event is triggered when a user registers for the hackathon.

The following steps happen:

1. Mail to the User from a template
2. Add user to Hackathon Team
3. Add User to Event-Blockers from shared mailbox, defined in the settings. 

### User added to topic

1. Create Teams Channel (Private Teams Channel) - if it not exists
2. Add User to Teams Channel.
3. Add Organization Team to Teams Channel.

Teams Channel is identified by UUID of the topic, written in the description field)

Prefix can be defined via an option.

### User removed from topic

1. Remove User from Teams Channel

## Mail Templates

* User Registration Mail
